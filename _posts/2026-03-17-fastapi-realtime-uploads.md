---
layout: post
show_meta: true
title: FastAPI Realtime events with mongodb changestreams and S3 file uploads
header: FastAPI Realtime events with mongodb changestreams and S3 file uploads
date: 2026-03-17 00:00:00
summary: How to use mongodb changestreams with S3 uploads to track progress
categories: mongodb fastapi s3 uploads
author: Chee Yeo
---

[Previous post]: https://www.cheeyeo.xyz/mongodb/changestream/2026/03/10/mongodb-changestream/
[MongoDB changestreams]: https://www.mongodb.com/resources/products/capabilities/change-streams
[https://github.com/cheeyeo/FastAPI_S3_Uploads]: https://github.com/cheeyeo/FastAPI_S3_Uploads

APPLICATION CODE: [https://github.com/cheeyeo/FastAPI_S3_Uploads]

In a [Previous post], I discussed how to setup a mongodb change stream on a collection and using it through the pyMongo driver. In this post, I hope to explain how I used the same concept but in a FastAPI application to provide real-time updates on file uploads.

While working on a file upload example using FastAPI and S3, I came across the issue of how to report progress on a file upload via the API backend to a React dashboard. The solution I came up with was:
* Using the mongodb change stream to broadcast progress updates. Since each upload is already stored as a document, we can enable watch on the collection and publish an update to the progress as it progresses.

* Create a websocket backend endpoint, that allows the React UI to connect to whenever an upload occurs. This websocket will contain the progress updates.

* Update the React UI whenever progress is received to show a progress bar.


The change stream code is as shown below:
{% highlight python %}

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

            await broadcast(
                {
                    "event": change["operationType"],
                    "document_id": str(doc_id),
                    "before": before,
                    "after": after,
                }
            )
{% endhighlight %}

`async_uploads` refers to the uploads collection fetched by the pymongo async client. The watch loop monitors changes to the uploads collection for any inserts, updates and replacements. Within the async with loop, we iterate over the stream iterator for each change event as it arrives. The event is passed to the `broadcast` method which publishes the events to all subscribed websockets.

The websocket endpoint is as follows, which is a generic example from the documentation:

{% highlight python %}

CLIENTS = set()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    CLIENTS.add(websocket)

    try:
        while True:
            await websocket.receive_text()
    except WebSocketDisconnect:
        CLIENTS.remove(websocket)
{% endhighlight %}

We store each websocket into a `CLIENTS` set. Whenever we broadcast an event, we iterate over all the websockets as follows:

{% highlight python %}
async def broadcast(data):
    text = json.dumps(data, default=str)
    for client in CLIENTS.copy():
        try:
            await client.send_text(text)
        except Exception:
            CLIENTS.remove(client)
{% endhighlight %}

While this is not the optimal solution, it works well for this example application.

Within the typescript code for the UI, we created an upload list and an upload card component for each upload initiated. Within each `UploadCard` object, we make a websocket connection to the backend. If the id in the data received matches the ID of the upload card and the event from the change stream is of type 'update', we start to update the progress bar component.

{% highlight javascript %}
import { useState, useEffect, useRef } from 'react';
import ProgressBar from "./ProgressBar";

const WS_URL = 'ws://localhost:8000/ws';

export type UploadCardProps = {
  uploadId: string;
  filename: string;
  initialProgress: number;
};

const UploadCard = ({ uploadId, filename, initialProgress }: UploadCardProps) => {
  const [progress, setProgress] = useState<number>(initialProgress);
  const ws = useRef<WebSocket | null>(null);
  const [message, setMessage] = useState<string>('');

  useEffect(() => {
    ws.current = new WebSocket(WS_URL);

    if (!ws.current || ws.current.readyState === WebSocket.CLOSED) {
      const ws2 = new WebSocket('ws://localhost:8001')
      ws.current = ws2
    }

    ws.current.onopen = () => {
      console.log(`WebSocket for ${filename} connected`);
      // Optionally send a message to subscribe to updates for this uploadId
      // ws.current?.send(JSON.stringify({ subscribe_to_upload: uploadId }));
    };

    ws.current.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);

        // if error exists close WS and show error
        if (data.event === 'update' && data.document_id === uploadId && data.after.status === 'error') {
          console.log(data.after.exception);
          setMessage(data.after.exception);
          ws.current?.close()
        }

        if (data.event === 'update' && data.document_id === uploadId && typeof data.after.percentage === 'number') {
          setProgress(data.after.percentage);
        }

        if (data.after.percentage === 100) {
          // close websocket after completion
          setMessage(data.after.s3_key + '\n' + data.after.s3_url)
          ws.current?.close()
        }

      } catch (err) {
        console.error("Error parsing websocket message for uploadId", uploadId, err);
      }

    };

    ws.current.onerror = (error) => {
      console.error(`WebSocket error for ${filename}:`, error);
    };

    ws.current.onclose = () => {
      console.log(`WebSocket for ${filename} closed`);
    };

    return () => {
      if (ws.current && ws.current.readyState === WebSocket.OPEN) {
        ws.current.close()
      }
    };
  }, [uploadId, filename]);

  return (
    <div className="upload-card">
      <div className="upload-card-filename">{filename}</div>
      <ProgressBar 
        progress={progress} 
        progressText={progress > 0 ? `${Math.round(progress)}%` : 'Initializing...'} 
      />
      <div>
        <pre>
          {message}
        </pre>
      </div>
    </div>
  );
};

export default UploadCard;
{% endhighlight %}

Lastly, we need to start the watcher loop on application startup. We use the `asynccontextmanager` to define a lifespan context function which allows you to perform startup and shutdown tasks:

{% highlight python %}
@asynccontextmanager
async def lifespan(app: FastAPI):
    task = asyncio.create_task(watch_uploads_changes())
    yield
    task.cancel()
    # close db clients on app exit
    await async_client.close()

app = FastAPI(
    title="File uploads example",
    description="Example of file uploads in FastAPI",
    version="1.0.0",
    lifespan=lifespan,
)
{% endhighlight %}

We pass the `watch_uploads_changes` function into the `asyncio.create_task` method which runs in its own coroutine. The `yield` passes control back to the FastAPI application. When the application exits, we perform cleanup by calling `task.cancel()` and close the mongodb client connection.

The `boto3` client was used for uploads to S3. I used the `upload_fileobj` method as it allows you to pass extra config and a callback to report progress:

{% highlight python %}
config = TransferConfig(
    multipart_chunksize=CHUNK_SIZE,
    multipart_threshold=5 * GB,
)

S3_CLIENT.upload_fileobj(
    Fileobj=file.file,
    Bucket=S3_BUCKET,
    Key=filename,
    ExtraArgs={"ContentType": file.content_type},
    Config=config,
    Callback=ProgressPercentage(file, upload_id),
)
{% endhighlight %}

The `TransferConfig` object from boto3 enables you to switch on multipart upload if the threshold is exceeded. In this example, we set it to a maximum of 5 GB with a chunk size of 1024MB. 

We define a `ProgressPercentage` object to track the progress. This is defined in the boto3 docs as a function but you can also provide your own classes so long as it implements __call__:

{% highlight python %}
class ProgressPercentage:
    def __init__(self, file: UploadFile, id: str):
        self._filename = file.filename
        self._size = float(file.size)
        self._seen_so_far = 0
        self._lock = threading.Lock()
        self._id = id

    def __call__(self, bytes_amount):
        with self._lock:
            self._seen_so_far += bytes_amount
            percentage = (self._seen_so_far / self._size) * 100

            # Below uses non-async mongo client as the callback is non async
            nonasync_uploads.update_one(
                {"_id": ObjectId(self._id)},
                {
                    "$set": UpdateUploadSchema(
                        current=self._seen_so_far,
                        percentage=percentage,
                        status="in progress",
                        updated_at=datetime.now(),
                    ).model_dump(exclude_unset=True)
                },
            )
{% endhighlight %}

Within the `__call__` function, we used the byes_amount parameter which is passed by the boto3 library to calculate the percentage of how much has been uploaded. We use the non-async client to update the associated record of its progress.

Since we are writing to the collection for each update, we trigger the watch loop for change streams which then publishes data to the UI via websockets.

Using the hybrid approach, I was able to track and display progress bars for S3 Uploads.

You can test it out at [https://github.com/cheeyeo/FastAPI_S3_Uploads]


