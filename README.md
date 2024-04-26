# Recording Pipeline

Code not available for public viewing. This is a private repository.

![image 3](https://github.com/blekmus/ace-recording-pipeline/assets/47277246/fcabb52d-e0f3-4eb5-b5d5-70e7ace93261)


## Description

The recording pipeline, located at is the service that’s responsible for handling all course recording uploads for Ace Academy. It consists of three main functions: handling Zoom recordings, handling manual recording uploads by instructors on the Ace Academy website, and handling uploads from custom scripts, such as when trying to upload videos downloaded from YouTube.

The primary objective of the pipeline is to receive videos, upload them to the Backblaze B2 archive, transfer them to Bunny Stream, and then notify the Ace Academy website upon completion. However, the traditional approach of using a simple webhook and a bash script encountered a challenge when multiple zoom recordings or large files (around 4GB each) were uploaded simultaneously. This led to memory constraints on the server during the processing phase.

To address this issue, the recording pipeline adopts a queue-based approach, allowing only one task (upload) to be processed at a time, regardless of the number of upload requests received. By queuing the tasks and processing them sequentially, the pipeline ensures that all tasks are executed without overwhelming the server's memory resources. In the event of a failure during the upload process (e.g., if Backblaze experiences a temporary downtime), the failed upload is retried multiple times with staggered delays, while the rest of the tasks continue to progress unaffected.

## Processes

The recording pipeline comprises of three processes, each managed by pm2, a reliable process manager that automatically handles application crashes and restarts. To configure and facilitate easy restarts of the pipeline, we utilize the pm2 ecosystem file located at `ecosystem.config.js`. Let's take a closer look at the three processes involved:

**Celery**: A Python-based job queue service, and its configuration can be found in the file `celery_app.py`. Celery handles the processing and management of background tasks efficiently.

**Pipeline**: This process is a Python FastAPI server, exposing three endpoints to the network: `/zoom-upload`, `/manual-upload`, and `/custom-upload`. These endpoints provide functionality for various types of uploads within the recording pipeline.

**TUS**: A Node.js resumable file upload server, written in TypeScript. It offers a single endpoint, `/tusupload`, enabling resumable file uploads.

#### Zoom Upload Workflow
1. **Webhook Processing**: Zoom processes the video and sends a webhook with the download link and meeting details. The `/zoom-upload` endpoint on the Pipeline server receives the webhook request.
2. **Authentication and Task Delegation**: The Pipeline endpoint verifies the authenticity of the request. If the request is valid, tasks are delegated to the Celery job queue for further processing.
3. **Task Queuing**: The Celery job queue manages the tasks and stores them in a Redis database running on port 6379. Redis here acts as a fail safe. Even if Celery fails, when it restarts Redis has all of the tasks stored in it to resume from when it failed. The tasks remain in the queue until their scheduled execution time arrives.
4. **Task Execution**: When a task's scheduled execution time is reached, the Celery process retrieves it from the queue. The corresponding function in the `celery_app.py` file is executed. Each task is executed sequentially to ensure proper order and prevent race conditions.
5. **Download Recording**: The first task in the queue is to download the recording from the provided download link. The recording is stored temporarily in the `/tmp` directory.
6. **Upload to B2 Bucket:** The second task is to upload the downloaded recording to the Backblaze B2 `ace-academy-recordings` bucket. This bucket acts as a video archive.
7. **Upload to Bunny:** Ace Academy hosts all its recordings on the video streaming platform Bunny  Stream. The recordings is uploaded to their services using their own TUS file uploading api.
8. **Notification and Cleanup**: Once all the tasks are completed, a notification containing the Bunny Stream and B2 information is sent to Ace. The video file is deleted to free up storage resources.
    
#### Manual Upload Workflow
1. **Tutor Uploading:** Tutors can manually upload a video from the Ace Academy website where it is uploaded to `/tusupload`. A unique `ace_id` is assigned to the video to uniquely identify each recording uploaded. This is to link the recordings uploaded through the Ace Academy website to the TUS server. 
2. **TUS Server:** Nodejs TUS server handles the upload in the pipeline. This is used to allow tutors to upload videos that allow them to resume if the upload fails. This acts as a way to save them data and time if the upload stops halfway by any chance. After the recording has been uploaded, a request is sent to the pipeline to be requested to be uploaded into `/manual-upload`.
3. **Pipeline Notification:**  A notification is sent by the TUS server to the pipeline with the location of the video file along with its metadata, this notification is sent through the local network `http://localhost:9009/manual-upload` where it is then added to the task queue.
4. The rest is the same as the Zoom upload workflow starting from task queuing.

#### Custom Upload Workflow

This workflow exists for edge cases. Say you have a youtube link and you want to send that video through the pipeline. One way is to locally download the video and manually upload it through the website using the Manual Upload endpoint. However, imagine you have 10 videos to upload and this is Sri Lanka we're talking about so the bandwidth costs would be immense. The solution is to download all of the videos to the recording server and to tell the pipeline to handle those files.
1. **Download File:** Download the file to the pipeline-server manually. Doesn’t matter where it is saved.
2. **Notify Pipeline:** A request should be sent to the pipeline through the `/custom-upload` endpoint. This request contains metadata about the uploaded file and the file path where the video is saved on the server. This is done through the local network `http://localhost:9009/custom-upload`. 
3. The rest is the same as the Zoom upload workflow starting from task queuing.

## Running

All three of the processes are run using the node-based process manager `pm2`. It's currently running on node version `18`. The server is using `nvm` to manage the node versions.

```bash
nvm use 18 // switch node version

pm2 restart ecosystem.config.js // to restart everything
pm2 restart pipeline // to restart a single process
```
