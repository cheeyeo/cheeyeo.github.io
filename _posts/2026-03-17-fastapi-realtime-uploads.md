---
layout: post
show_meta: true
title: FastAPI Realtime file uploads
header: FastAPI Realtime file uploads
date: 2026-03-17 00:00:00
summary: How to use realtime uploads via mongodb changesets in FastAPI
categories: mongodb fastapi s3 uploads
author: Chee Yeo
---

[Previous post]: 
[MongoDB changestreams]: https://www.mongodb.com/docs/manual/changeStreams



```python
async def watch_uploads_changes():
    pipeline = [
        {
            "$match": {
                "operationType": {"$in": ["insert", "update", "replace", "delete"]}
            }
        }
    ]
    # call watch on the uploads collection; async_uploads ref the uploads collection
    async with await async_uploads.watch(
        pipeline, full_document="updateLookup", full_document_before_change="required"
    ) as stream:
        async for change in stream:
            doc_id = change["documentKey"]["_id"]
            before = change.get("fullDocumentBeforeChange", {})
            after = change.get("fullDocument", {})

            # logger.info(f"BEFORE: {before} AFTER: {after}")

            await broadcast(
                {
                    "event": change["operationType"],
                    "document_id": str(doc_id),
                    "before": before,
                    "after": after,
                }
            )
```