---
layout: post
show_meta: true
title: MongoDB changestream
header: FastAPI Realtime file uploads
date: 2026-03-17 00:00:00
summary: How to use realtime uploads via mongodb changesets in FastAPI
categories: mongodb fastapi s3 uploads
author: Chee Yeo
---

[Previous Post]: 

[MongoDB changestreams]: https://www.mongodb.com/docs/manual/changeStreams


In a [Previous Post], I discussed how to setup and run a mongodb replicaset locally for the purpose of using [MongoDB changestreams]. This post will discuss an approach of using changestreams in the context of the python mongodb driver.

MongoDB changestreams allow applications to access real-time changes with the application subscribing to the data changes for a specific collection and react to them accordingly. In the example below, we are using the `pymongo.MongoClient` to create a test database and a test collection of users as an example:

```python
DATABASE_URL = f"mongodb://{get_settings().mongo_db_root_username}:{get_settings().mongo_db_root_password}@mongo1:27017,mongo2:27017,mongo3:27017/"

client = MongoClient(DATABASE_URL)
db = client.get_database("teststreams")

users = db.create_collection("users")

pipeline = [
    {
        "$match": {
            "operationType": {"$in": ["insert", "update", "replace", "delete"]}
        }
    }
]

with users.watch(pipeline, full_document="updateLookup") as stream:
    for change in stream:
        print(change)
```

We created a configuration option called `pipeline` which specifies the operation type we are interested in watching. In this example, we are interested in change events for inserts, updates, replace when adding new fields and deletes. By default, change streams only return the delta of fields during the update operation. To return the most current majority-commited version of the updated document, we set `full_document="updateLookup"` which returns a `full_document` field that shows the current version of the document affected by the update operation.

The `watch` function is an iterator loop that blocks while waiting for change events to arrive. If we were to run another script to create an additional user of name Alice, we receive the following insert event:

```json
{
    '_id': {
      '_data': 'XXXX'
    }, 
    'operationType': 'insert', 
    'clusterTime': Timestamp(1773749951, 1), 
    'wallTime': datetime.datetime(2026, 3, 17, 12, 19, 11, 657000), 
    'fullDocument': {
        '_id': ObjectId('69b946bf78a20b5c76acb971'), 
        'name': 'Alice'
    }, 
    'ns': {
        'db': 'teststreams', 
        'coll': 'users'
    }, 
    'documentKey': {
        '_id': ObjectId('69b946bf78a20b5c76acb971')
    }
}
```

Note that the `operationType` is set to `insert` and the full document is provided under the `fullDocument` key since we set full_document to be updateLookup. 

If we were to add a new field of `email` to the same user, we trigger a `replace` event as follows:

```json
{
    '_id': {
        '_data': 'XXXX'
    }, 
    'operationType': 'replace', 
    'clusterTime': Timestamp(1773750282, 1), 
    'wallTime': datetime.datetime(2026, 3, 17, 12, 24, 42, 534000), 
    'fullDocument': {
        '_id': ObjectId('69b946bf78a20b5c76acb971'), 
        'name': 'Alice', 
        'email': 'test@example.com'
    }, 
    'ns': {
        'db': 'teststreams', 
        'coll': 'users'
    }, 
    'documentKey': {
        '_id': ObjectId('69b946bf78a20b5c76acb971')
    }
}
```

Note that the `operationType` is set to replace and the document returned include the new field added.

If we were interested in obtaining a version of the document before and after the changes are made, we need to set `changeStreamPreAndPostImages` to true when creating the collection.

Pre-image refers to the document before it was updated. Post-image refers to the document after it was updated. According to the documentation, when using `fullDocument: "updateLookup"` with a $match filter, we need to set it as a document deletion returns a null value in the `fullDocument` field which prevents the change stream from finding the resume token.

The updated codebase is now as follow:
```python

DATABASE_URL = f"mongodb://{get_settings().mongo_db_root_username}:{get_settings().mongo_db_root_password}@mongo1:27017,mongo2:27017,mongo3:27017/"

client = MongoClient(DATABASE_URL)
db = client.get_database("teststreams")
users = db.create_collection("users", changeStreamPreAndPostImages={"enabled": True})

pipeline = [
    {
        "$match": {
            "operationType": {"$in": ["insert", "update", "replace", "delete"]}
        }
    }
]

with users.watch(pipeline, full_document="updateLookup", full_document_before_change="required") as stream:
    for change in stream:
        print(change)
```

The returned event now includes a `fullDocument` and a `fullDocumentBeforeChange` fields. The below shows and update event on the user record after the email field is changed:

```json
{
    '_id': {
        '_data': 'XXXX'
    }, 
    'operationType': 'replace', 
    'clusterTime': Timestamp(1773752106, 1),
    'wallTime': datetime.datetime(2026, 3, 17, 12, 55, 6, 639000),
    'fullDocument': {
        '_id': ObjectId('69b94d6978a20b5c76acb975'),
        'name': 'Alice', 
        'email': 'alice@example.com'
    }, 
    'ns': {
        'db': 'teststreams', 
        'coll': 'users'
    }, 
    'documentKey': {'_id': ObjectId('69b94d6978a20b5c76acb975')}, 'fullDocumentBeforeChange': {
        '_id': ObjectId('69b94d6978a20b5c76acb975'), 
        'name': 'Alice', 
        'email': 'alice@test.com2'
    }
}
```

Using the built-in change stream feature, we are able to track changes to individual records, collections and databases. This allows us to build event-driven applications such as progress updates and custom user alerts on demand without polling.

The next post will examine how to incorporate this into a working FastAPI application to provide async real-time updates.
