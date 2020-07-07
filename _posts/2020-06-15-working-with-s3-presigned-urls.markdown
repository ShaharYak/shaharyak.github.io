---
layout: post
title: Working With S3 Presigned Urls
date: 2020-06-15
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: presigned.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [AWS, S3, Presigned Url]
---

Grant temporary access to objects in AWS S3 buckets without the need to grant explicit permissions

![](https://cdn-images-1.medium.com/max/3494/1*4ZaSKTqyUqtbqkHRlh7qIQ.png)
**Originally posted at [Altostra blog](https://altosta.com/blog/path-to-the-post)**

## TL;DR

S3 pre-signed URLs grant temporary access to objects in AWS S3 buckets without the need to grant explicit permissions.

* Supports only GET and PUT requests.

* Request headers must exactly match both when creating and using the URLs.

## Motivation

It’s a best practice to keep S3 buckets private and only grant public access when absolutely required. Then how can you grant your client access to an object without changing the bucket ACL, creating roles, or providing a user on your account? That’s where S3 pre-signed URLs come in.

## Pre-Signed URLs

S3 pre-signed URLs are a form of an S3 URL that temporarily grants **restricted access** to a **single** S3 object to perform a **single operation** — either PUT or GET — for a predefined **time limit**.

To break it down:

* It is **secure** — the URL is signed using an AWS access key

* It grants **restricted access** — only one of **GET** or **PUT** is allowed for a single URL

* Only to a **single** object — each pre-signed URL corresponds to one object

* With a **time-constrained** — the URL expires after a set timeout

In the next section, we’ll use these properties to generate an S3 pre-signed URL.
So let’s jump over to some code samples.

## Key Points

Building a solution with S3 pre-signed URLs is not without its pitfalls. Here are some pointers to save you valuable time:

1. You must send the same HTTP headers — when accessing a pre-signed URL — as you used when you generated it. For example, if you generate a pre-signed URL with the Content-Type header, then you must also provide this header when you access the pre-signed URL. Beware that some libraries - for example, Axios - attach default headers, such as Content-Type, if you don't provide your own.

1. The default pre-signed URL expiration time is 15 minutes. Make sure to adjust this value to your specific needs. Security-wise, you should keep it to the minimum possible — eventually, it depends on your design.

1. To upload a large file — larger than 10MB — you need to use multi-part upload. I’ll cover this topic in my next post.

1. Pre-signed URLs support only the getObject, putObject and uploadPart functions from the AWS SDK for S3. It's impossible to grant any other access to an object or a bucket, such as listBucket.

1. Because of the previously mentioned AWS SDK functions limitation, you can’t use pre-signed URLs as Lambda function sources, since Lambda requires both listBucket and getObject access to an S3 object to use as a source.

## Implementation

Now that we have all the information we require let’s get to coding.

Each code sample section contains two parts, the producer of the URL and the consumer. I use the JavaScript AWS SDK to generate the pre-signed URLs and the Axios NPM package to access these URLs from the consumer using basic HTTP requests.

In the examples, I use the constants BUCKET_NAME and OBJECT_NAME to represent my bucket and object - replace these with your own as you see fit.
Also, there is a placeholder for theaccesskeyId and secretAccessKey, you can read about it [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html).

### **Generating S3 pre-signed URLs — The Producer**

<script src="https://gist.github.com/ShaharYak/bcd8ebb8099e4d5a540027b365033815.js"></script>

### Accessing S3 objects using pre-signed URLs — The Consumer

<script src="https://gist.github.com/ShaharYak/cc8c4b2caf1c51350769fcf5931703c6.js"></script>

**Further Reading:**
* [Using Signed URLs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-signed-urls.html)
* [Share an Object with Others](https://docs.aws.amazon.com/AmazonS3/latest/dev/ShareObjectPreSignedURL.html)

