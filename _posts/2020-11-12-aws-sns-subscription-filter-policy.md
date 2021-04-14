---
layout: post
title: How to reduce Lambda invocations using SNS filter-policy
description: Let AWS filter your messages and save you time and money
date: 2020-11-12
img: aws-sns-subscription-filter-policy.jpg
tags: [AWS, Lambda, Serverless]
---

## Let AWS filter your messages and save you time and money

<p align="center">
  <img src="https://miro.medium.com/max/700/1*ceTdFZnQqpLD3CNp7I0NTA.png">
</p>

**Originally published on [Altostra](https://www.altostra.com/blog/aws-sns-subscription-filter-policy)**


## Motivation
At Altostra, we work with a serverless distributed system and use a message bus for communication between our system components. Early on, we implemented our message bus using an SNS topic. See our [previous post] (https://www.altostra.com/blog/aws-pub-sub) for other potential solutions.

By design, SNS sends messages to all subscribers - but at some point, we wanted to limit that each subscriber will get only the messages relevant to its functionality.

## How it works
We want our system components to receive only relevant messages and nothing more. That's when the filter-policy feature of SNS got into the picture.

SNS filter-policy enables you to filter messages by setting a set of filter-rules on every subscription. We move the message filtering responsibility from our code to AWS, thus reducing Lambda invocations, costs and code(switch cases, over-abstraction and so on). The filter-policy filters the messages according to the `MessageAttributes` attribute of the SNS message.

For example, a filter-policy which accepts only messages with an `event_type` of `deployment_started` looks like this:

```json
{
  "event_type": ["deployment_started"]
}
```

The filter-policy introduces a set of rules, restricted to the types `String`, `String.Array` and `Number`, and several operators such as `equals`, `in`, `not in`, mathematical operators (only for the`Number` type), and more. Check out the full list [here](https://docs.aws.amazon.com/sns/latest/dg/sns-subscription-filter-policies.html#subscription-filter-policy-constraints).

A filter-policy can be a simple condition on a single property or a complex condition, which then operates with the `AND` operator between properties and the `OR` operator between the property values.
For example, this filter policy:

```json
{
  "event_type": ["deployment_started", "deployment_finshed",],
  "vendor_id": [{"anything-but": 101}]
}
```
accepts messages where the `event_type` is `deployment_started` **or** `deployment_finshed` **AND** the `vendor_id` is not `101`.

An SNS message publishing code that passes the filter above would be:

```js
const AWS = require('aws-sdk');

async function publishMessage(message, topicArn) {
  const sns = new AWS.SNS()

  const messageParams = {
    Message: message,
    TopicArn: topicArn,
    MessageAttributes: {
      event_type: {
        DataType: 'String',
        StringValue: 'deployment_started'
      },
      vendor_id: {
        DataType: 'Number',
        StringValue: '200'
      }
    }
  }

  await sns.publish(messageParams).promise()
}
```

## Lets get our hands dirty
In the example below, I'll demonstrate a full flow of:

1. Subscribing Lambdas to SNS
2. Add a filter-policy to each subscription
3. Publish messages according to the filter policy, and show you the logs of each Lambda

Using Altostra designer, I will quickly build the architecture, subscribe Lambdas to SNS and add filter policy to each subscription. I will show you the logs in the AWS web console.

### Subscribe Lambdas to an SNS
First, let's create two Lambda functions and connect them to an SNS topic.

LambdaA handler will print `LambdaA: + the message`

LambdaB handler will print `LambdaB: + the message`

![Subscribe](../images/blog/aws-sns-subscription-filter-policy/lambda-sns-creation.gif)

### Add the filter policy
Now, let's add a filter-policy to each Lambda, LambdaA accepts messages when the `type` attribute equals `hello` and LambdaB when the `type` attribute equals `world`.

![Filters](../images/blog/aws-sns-subscription-filter-policy/create-filter.gif)

### Publish!
Finally, let's publish two messages to the SNS topic, one with the `hello` type and the other with the `world` type. After that, I'll show you the logs for each Lambda.

![Publish](../images/blog/aws-sns-subscription-filter-policy/sns-filter-send-messages.gif)
