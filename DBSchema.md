Database Schema for Storing S3 objects in Relational Databases. 
=======================================================

Operations to be Supported:
-----------------------
- Creating a bucket
- removing a bucket
- listing buckets
- uploading an object
- removing an object
- listing objects in a bucket

Additonal Requirements:
--------------------
1. The data for large object uploads must be split into multiple chunks of a manageable size. (Ex: a 1 TB object wouldn't be read all at once, but rather one piece at a time that are stored over multiple entries)
2. if a large object is not finished uploading yet, it shouldn't show up in a bucket listing
3. until a large object upload completes, a GET for that object's data would return either the old object's data or none if it didn't exist, a GET after the upload finishes would return the new data


Steps:
-----
1. Make Simple Schema
2. Add S3 Attributes
3. Add Support for S3 operations
4. Look up and add support for chunked uploads
5. Look up and add support for very large object, Make the VLOB(very large object) storage atomic.
6. Look at scalability



## 1. Simple Schema.


#### RELATIONS: 
	`Bucket`()	{List of Buckets and Bucket Metadata}
	`ObjectDirectory`()	{List of objects and metadata}
	`ObjectStore`() {Stores small objects}	

`ObjectDirectory` links(foreign Key) to `Bucket` and `ObjectStorage` Links to `ObjectDirectory`.

## 2. Add the S3 Attributes.


#### RELATIONS:
```
   Bucket (
		BucketID: Integer PRIMARY KEY,
		BucketName: VARCHAR UNIQUE,
		// Bucket ACL: ??,
		// BuckerOwner: Interger FOREIGN KEY REFERENCES User.Uid,
		Created: Date,
		// Location: VARCHAR,
		ObjectCount: Integer
	)
  
  ObjectDirectory (
		ObjectID: VARCHAR PRIMARY KEY,
		ObjectName: TEXT,
		BucketId: Integer FOREIGN KEY REFERENCES Bucket,
		Created: Date,
		Size: Integer
	)

	ObjectStore (
		ObjectID: VARCHAR FOREIGN KEY REFERENCES ObjectDirectory, 
		Data: BLOB,
		PRIMARY KEY(ObjectID)
	)

```

ObjectID: A Hash of the objects contents
Size: In bytes
ObjectName: A user friendly name
BucketID: A Secondary Key

Regions: Not Supported
ACL, Owners: Not Currently Supported


## 3. Support for S3 Operations:


a.) Creating a bucket: Insert into the bucket table, with default values. Name provided by user.
b.) Removing a bucket: Along with all the objects in the Bucket. Atomic transaction that:
		i.) Removes all the objects from ObjectStore JOIN ObjectDirectory With given bucketId
		ii.) Removes all the Objects in the bucket from ObjectDirectory
		iii.) Removes the bucket from Bucket.

c.) Listing Buckets: SELECT * from Bucket
d.) Uploading an Object: Requires a Bucket Name, Object Content. Atomic transaction:
		i.) Hash the objectID, Store the Data and ID in ObjectStore
		ii.) Store the metadata including BucketID in ObjectDirectory
		iii.) Increment the number of objects in a bucket by 1
e.) Removing an Object: Exactly the opposite of Uploading, does not require bucketId.
f.) Listing Objects in a bucket: SELECT ObjectID FROM ObjectDirectory WHERE  BucketId = ID.


## 4. Chunked Uploads and Large Object Support:

In this case, a few extra attributed are added to `ObjectDirectory` and two extra tables: `Transfers` and `LargeObjectStore` are needed:

```
	LargeObjectStore (
		ObjectID: Varchar FOREIGN KEY REFERENCES ObjectDirectory,  // Just an incremental number
		Segment Number: Integer,
		FirstSegment Hash: VARCHAR,  // For quick GETting of objects
		Size of Segment: ?,  // Also implicit from the Segment number, Object size.
		Data: BLOB,
		PRIMARY KEY(FirstSegment Hash, Segment Number)
	)
  
 ```

And the modified the ObjectDirectory looks like this:
	
```
	ObjectDirectory (
		ObjectID: Varchar PRIMARY KEY,
		ObjectName: TEXT,
		BucketId: Integer FOREIGN KEY REFERENCES Bucket,
		Created: Date,
		LOB Flag: Char,
		Size: Integer,
		FirstSegment Hash: Varchar,
		Latest Transfer TimeStamp: Integer
	)

	Transfers (
		ObjectID: Integer,
		StartSegment Hash: Varchar,
		Size: Integer,
		Segments Remaining: Integer, // to check when the transfer finished and to resume transfer
		Start Timestamp: Integer,  // Incremental integers 0, 1, 2...
		Finish Timestamp: Integer, // Same as above
		PRIMARY KEY(ObjectID, Finsih Timestamp), // For quick access when uplaod completes
		UNIQUE(ObjectId, Start Timestamp)
	)

```
The extra fields and Tables is to ensure the correctness of the transfer orders. The `LOB flag` stores which store the object data will be found in.
This is implicit from the Size, anyway(but Semantics?). 

For a small value store, the object is simply referenced by ID. For the large value store, we join the different chunks of the object using the Attribute `FirstSegment Hash`, which is the Hashed value of the first data segment uploaded. This is necessary becuase the large value store may have 
multiple partially uploaded versions of the object. The versions to be read on a GET request is the one with the FirstSegment as that of the record in the
`ObjectDirectory`. This is also why `FirstSegment Hash` is also part of the primary key.

Clearly, since the bucket listing is done based on the `ObjectDirectry` table, it won't show partially uploaded objects(or versions).

The Synchronisation of the uploads and correctnedd of the `ObjectDirectory` table is the job of the transfer manager in the backend.
At any instant, there will be exactly one complete upload of an object. This is taken care of as follows:

```
For a particular ObjectID I:
	When an upload O completes:
		f = finishTimeStamp(O)
		O1 = Tranfer with finish timestamp of (f - 1) (if f > 0)
		if (startTimeStamp(O1) > startTimeStamp(O)):
			delete O
		else:
			set I.Hash = O.Hash
			delete O1

```

All the above operations needs to be done atomically. As an example, Let Object with ID = 5 have a copy O1 stored currently. If there are two
upload requests O2 and O3 for this object. If O2 completes before O3, on completion, O2 replaces O1 and O3 replaces O2. However,
if O3 completes before O2, when O3 finishes, it will delete O1 and set the object record in `ObjectDirectory`, and then when O2 completes, it won't replace O3 since O3 started after O2(that is, it is a more recent request).

This satisfies the third of the additional requirements, since the `FirstSegment Hash` only changes in the `ObjectDirectory` after the transfer finishes.

## 5. For Very Large Object Support:

For VLOB(Very large objects), storing the data in a database might be unfeasibale. Store on a filesystem? Then how do ensure 
ensure atomicity and integrity in this case?

## 6. Scalability


#### Questions to ponder:

1. In Chunked Uploads, Split object table into two, LOBDirectory and SOBDirectory?
2. Keep ObjectID incremental or make it hash of the object?
3. In chunked uploads, if a later upload request on an object finishes before the earlier one
   should the earlier one be aborted?
4. Should Transfers be an array of Transfers objects insted of Table?
