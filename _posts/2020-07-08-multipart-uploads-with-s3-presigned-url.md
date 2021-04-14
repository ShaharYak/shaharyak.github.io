---
layout: post
title: Multipart uploads with S3 pre-signed URLs
description: Breaking an object into pieces, uploading it to S3 in a secure way and constructing it to one piece
date: 2020-07-08
img: multipart-uploads-with-s3-presigned-url.jpeg
tags: [AWS, S3, Presigned Url, Multipart Upload]
---
## Breaking an object into pieces, uploading it to S3 in a secure way and constructing it to one piece

<p align="center">
  <img src="https://miro.medium.com/max/700/1*ywkGJ-Zhohw3fsQ2iL3fwg.jpeg">
</p>

**Originally published on [Altostra](https://www.altostra.com/blog/multipart-uploads-with-s3-presigned-url)**

## Let’s get on the same page

In my previous post, [Working with S3 pre-signed URLs](https://www.altostra.com/blog/pre-signed-urls), I showed you how and why I used pre-signed URLs.
This time I faced another problem: I had to upload a large file to S3 using pre-signed URLs. To be able to do so I had to use multipart upload, which is basically uploading a single object as a set of parts, with the advantage of parallel uploading.
In the following article, I’ll describe how I combined and used multipart upload with pre-signed URLs.

A normal multipart upload looks like this:

1. Initiate a multipart upload

1. Upload the object’s parts

1. Complete multipart upload

For that to work with pre-signed URLs, we should add one more stage: generating pre-signed URLs for each part.
The final process looks like this:

1. Initiate a multipart upload

1. Create pre-signed URLs for each part

1. Upload the object’s parts

1. Complete multipart upload

Stages 1,2, and 4 are server-side stages, which require an AWS access key id & secret, where stage number 3 is client-side.

Let’s explore each stage.

### Before we begin

As in my previous post, in the examples below, I used the constants ```BUCKET_NAME``` and ```OBJECT_NAME``` to represent my bucket and object - replace these with your own as you see fit.
Also, there are placeholders for the ```accesskeyId``` and ```secretAccessKey```, you can read about them [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html).

### Stage one — Initiate a multipart upload

At this stage, we request from AWS S3 to initiate multipart upload, in response, we will get the ```UploadId``` which will associate each part to the object they are creating.

```typescript

import AWS from 'aws-sdk'
import cuid from 'cuid'

async function initiateMultipartUpload() {
  const s3 = new AWS.S3({
  accessKeyId: /* Bucket owner access key id */,
  secretAccessKey: /* Bucket owner secret */,
  sessionToken: `session-${cuid()}`
  })

  const params = {
    Bucket: BUCKET_NAME,
    Key: OBJECT_NAME
  }

  const res = await s3.createMultipartUpload(params).promise()

  return res.UploadId
}
```

### Stage Two — Create pre-signed URLs for each part

This stage is responsible for creating a pre-signed URL for each part. Each pre-signed URL is being signed with the ```UploadId``` and the ```PartNumber```, because of that, we have to create a separate pre-signed URL for each part. The pre-signed URLs are being generated with the ```uploadPart``` method, which grants the user access to, obviously, upload a part.

```typescript
import AWS from 'aws-sdk'

async function generatePresignedUrlsParts(s3: AWS.S3, uploadId: string, parts: number) {
  const baseParams = {
    Bucket: BUCKET_NAME,
    Key: OBJECT_NAME,
    UploadId: uploadId
  }

  const promises = []

  for (let partNo = 1; partNo < parts + 1; partNo++) {
    promises.push(s3.getSignedUrlPromise('uploadPart', {
      ...baseParams,
      PartNumber: partNo
    }))
  }

  const res = await Promise.all([
    ...promises
  ])

  const urls: Record<number, string> = {}

  for (let partNo = 0; partNo < parts; partNo++) {
    urls[partNo + 1] = res[partNo]
  }

  return urls
}
```

### Stage Three — Upload the object’s parts

At this stage, we will upload each part using the pre-signed URLs that were generated in the previous stage. You can see each part is set to be 10MB in size. S3 Multipart upload doesn't support parts that are less than 5MB (except for the last one).

After uploading all parts, the ```etag``` of each part that was uploaded needs to be saved. We will use the ```etag``` in the next stage to complete the multipart upload process.

```typescript
import Axios from 'axios'

interface Part {
  ETag: string
  PartNumber: number
}

const FILE_CHUNK_SIZE = 10_000_000

async function uploadParts(file: Buffer, urls: Record<number, string>) {
  const axios = Axios.create()
  delete axios.defaults.headers.put['Content-Type']

  const keys = Object.keys(urls)
  const promises = []

  for (const indexStr of keys) {
    const index = parseInt(indexStr) - 1
    const start = index * FILE_CHUNK_SIZE
    const end = (index + 1) * FILE_CHUNK_SIZE
    const blob = index < keys.length
      ? file.slice(start, end)
      : file.slice(start)

    promises.push(axios.put(urls[index], blob))
  }

  const resParts = await Promise.all(promises)

  let parts: Part[] = []
  let i = 1

  for (let part of resParts) {
    parts.push({
      ETag: (part as any).headers.etag,
      PartNumber: i
    })

    i++
  }

  return parts
}
```

### Stage Four — Complete multipart upload

The last stage’s job is to inform to S3 that all the parts were uploaded. By giving the ```completeMultipartUpload``` function the ```etag``` of each part, S3 knows how to constructs the object from the uploaded parts.

```typescript

import AWS from 'aws-sdk'
import cuid from 'cuid'

interface Part {
  ETag: string
  PartNumber: number
}

async function completeMultiUpload(uploadId: string, parts: Part[]) {
  const s3 = new AWS.S3({
    accessKeyId: /* Bucket owner access key id */,
    secretAccessKey: /* Bucket owner secret */,
    sessionToken: `session-${cuid()}`
  })

  const params = {
    Bucket: BUCKET_NAME,
    Key: OBJECT_NAME,
    UploadId: uploadId,
    MultipartUpload: { Parts: parts }
  }

  await s3.completeMultipartUpload(params).promise()
}
```

### Extra Stage — Avoid extra charges

If at any stage you want to abort the upload, you can do it with the function ```abortMultipartUpload``` (you can read about it [here](https://docs.aws.amazon.com/AmazonS3/latest/API/API_AbortMultipartUpload.html)).

Be aware that after you initiate a multipart upload and upload one or more parts, to stop from being charged for storing the uploaded parts, you must either complete or abort the multipart upload. Amazon S3 frees up the space used to store the parts and stop charging you for storing them only after you either complete or abort a multipart upload.

As a best practice, Amazon recommends you configure a lifecycle rule (using the ```AbortIncompleteMultipartUpload``` action) to minimize your storage costs.
