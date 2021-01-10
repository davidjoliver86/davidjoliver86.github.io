---
title: "The hidden costs of S3 versioning"
date: 2021-01-09T12:56:58-08:00
draft: false
tags:
  - aws
  - s3
---

I'm going to start by proposing a riddle:

> How big is my S3 bucket?

The answer to that can be surprisingly ambiguous. The naive approach is simply to
iterate over all the objects in your bucket in order to sum up the sizes of each of
its objects. And that works perfectly fine if you have something on the order of
thousands, or perhaps tens of thousands of objects in a bucket. But should you really
need to do this? If you wanted to see how much space a folder in a Unix environment
uses, the `du` manpage doesn't tell you "eh, parse the output of `ls -l` and add it
up yourself".

So - no, you don't really need to sum this up yourself. The 'Metrics' tab in the S3
console exposes the CloudWatch metric that already computes this for you. So, problem
solved, right? Blog post over?

It depends.

## The wonders and woes of versioning

Versioning simply allows Amazon to store multiple variants of the same object in a bucket.
You can preserve and restore old versions at any time, which is a wonderful preventative
measure against unintended user or application error. Every object in your bucket has
a random version hash. If you overwrite an object in your bucket, a new version is created
with a different hash. This becomes the "current" version, and all other versions become
"noncurrent". Also, if you "delete" an object, a new version of that object acts as a
"delete marker" and becomes the current version - but this "delete marker" can, itself,
be deleted to effectively "undelete" the object.

## That doesn't sound bad at all.. so what's the problem with versioning?

I was on a Slack call with my manager where we were going over some of the S3 buckets
across our AWS estate. We were wanting to determine how much data we were storing across
our buckets, and for some of our buckets, we were running into numbers that raised a few
serious red flags.

There are some S3 buckets that are being used as a "file mirror" - effectively, an EC2
instance would have a cron job that runs `aws s3 sync` with its home folder. This was
destined toward an S3 bucket that has versioning. When we were determining how much
storage was used in that bucket, we were able to sum up each item, and it was on the
order of about 30 gigabytes. Pretty much what we were expecting. Cost Explorer and
CloudWatch, however, painted a vastly different picture... 15 *TERABYTES*.

Sure enough, we weren't being charged for the 30 gigabytes that we had on the surface.
We were being charged for the 15 terabytes. And the culprit that wound up using 14.97
terabytes? Versions. Old versions over the years.

## Always have a lifecycle rule!

By having a lifecycle rule in place, you can control the buildup of years' of stale
versions adding on to an ever-growing pile in your S3 bill. A lifecycle rule allows you
to define rules on how to manage old versions of objects, or objects that aren't frequently
accessed. There are more details about the lifecycle rules you can enable in the [official docs](https://docs.aws.amazon.com/AmazonS3/latest/dev/intro-lifecycle-rules.html). Specifically, in this case, we would want a `NoncurrentVersionExpiration` rule to
automatically delete old object versions after a certain number of days. This way, you
still have enough time to recover a deleted object, but for the vast majority of files
that were meant to be deleted (or constantly overridden), this will greatly help manage
your costs.

## The simple takeaway

If you have versioning enabled on your bucket, *always* be sure you have a good lifecycle
rule in place to handle object expirations. Otherwise, these costs will slowly eat away
at your S3 bill.
