---
title: "Part 4/4 - Create and deploy an automated, serverless Newegg stock checker using Python, Lambda, and Terraform"
date: 2020-12-07
draft: false
tags:
  - aws
  - python
  - terraform
  - lambda
  - cloudwatch
  - s3
---
> ⚠️ This tutorial is a relic of its time period, back when the COVID-19 pandemic was
  beginning to surge. While the information is still relevant and factual, this tool
  was designed with once-in-a-lifetime supply chain conditions in mind, and the end
  result may not be as useful today as it was back when this article was written.

Alright.. the home stretch. At the end of the last part, we mentioned that we need a
stateful data store for our Lambda function, otherwise it would keep wanting to send
alerts every minute that there's no stock, and that's not very useful. The stateful
data store we're going to use is S3.

Of course, you don't *have* to use S3 - you could spin up a DynamoDB or RDS instance,
and there's the relatively new EFS on Lambda which I haven't personally tested, but
for our use case - storing an arbitrary file - S3 is my pick.

## Creating the S3 bucket

There's a pretty wide array of options for the `aws_s3_bucket` resource in the
[Terraform docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket)
but we only need two:
```hcl
resource "aws_s3_bucket" "newegg" {
  bucket_prefix = "io-github-davidjoliver86-newegg-"
  acl           = "private"
}
```
I use `bucket_prefix` in my case because all S3 buckets share a global namespace, and
the combination of reverse domain notation, combined with the random numbers appended
to the end, ought to ensure a globally unique name. You also can technically omit `acl`
and it'll default to `private`, but it's still a good habit to remind youself not to *ever*
use any of the `public` options without a good reason! Otherwise, you could end up on a
[BitDefender list](https://businessinsights.bitdefender.com/worst-amazon-breaches) that
you *really* don't want to be on.

Also - I'm intentionally omitting another crucial thing we'll need to Terraform. But
it'll all come together.

## Updating our stock checker

Before we update our code, let's think about when *exactly* we'd want a notification.
Obviously, we don't want a notification every time it runs, although I suppose there's
some comfort in knowing that it's at least running. We want a notification when something
lands in stock. But suppose one product is back in stock, and stays in stock for about 5
minutes. If we run this every minute, we now have 5 notifications telling us the same
thing. So really, we only want a notification when there's a change in the "in stock"
offerings vs. the last time it was run.

First, we're changing our `get_items_in_stock` function to just `get_items` so that we
can do some analysis later:
```py
def get_items(self) -> Dict[str, bool]:
    """
    After parsing the page, return a dict of the items and their stock status.
    """
    return self.items
```
And now the meat of the comparison:
```py
def compare_to_s3(
    current_items: Dict[str, bool], s3_bucket: str, s3_obj: str
) -> Tuple[Status, Collection[str]]:
    """
    Diff the results against the previous state in S3, then update S3.
    """
    s3 = boto3.resource("s3")
    status = Status.NO_CHANGE

    # Retrieve existing dict, or instantiate an empty one if it doesn't exist in S3.
    obj = s3.Bucket(s3_bucket).Object(s3_obj)
    try:
        existing_items = json.load(obj.get()["Body"])
    except exceptions.ClientError as e:
        if e.response["Error"]["Code"] == "NoSuchKey":
            existing_items = {}
            status = Status.INITIALIZED
        else:
            raise e

    # If our items haven't changed, just exit. A PutObject operation is more costly.
    if current_items == existing_items:
        return (status, set())

    # Are there new in-stock items compared to last time?
    in_stock_current = set(
        [item for item, in_stock in current_items.items() if in_stock]
    )
    in_stock_previous = set(
        [item for item, in_stock in existing_items.items() if in_stock]
    )

    # Need to update our object now.
    with io.BytesIO(json.dumps(current_items).encode("utf-8")) as fp:
        obj.upload_fileobj(fp)

    # Return a set of any new items.
    diff = in_stock_current - in_stock_previous
    if diff and (status != Status.INITIALIZED):
        status = Status.ITEMS_IN_STOCK
    if (not diff) and in_stock_previous:
        status = Status.ITEMS_GONE
    return (status, diff)
```
What we're storing in S3 is a JSON file of the `items` dictionary - that is, the name
of each item, and whether it was in stock. We can diff against the previous run by
downloading our file from S3 to identify what's changed, then we upload the current
items back to S3. To save money on unnecessary `PutObject` calls, we only do this
if there's been a definitive change.

We're interested in the four different possibilities:
* `NO_CHANGE` - There's been no change to the offerings that are *in stock*. It's
  possible that an item can be added to the page, but have no stock initially. This
  won't trigger a message.
* `INITIALIZED` - The stock checker was run for the first time. Send a message to
  confirm it works, and print the current stock list.
* `ITEMS_IN_STOCK` - Okay.. this is the only one we're *really* interested in. The
  only way `diff` is non-empty is if there's a new in-stock item vs. the last run.
* `ITEMS_GONE` - We were too slow (or asleep). Whatever was in stock is now gone.

Now to make changes to our handler and create some SNS sending functions:
```py
def send_init_message(
    sns, topic_arn: str, checker_name: str, stock_changes: Collection[str]
):
    stock_changes_lines = "\n".join(stock_changes)
    subject = f"[{checker_name}] - First run successful"
    message = f"Stock checker operational. Items in stock:\n{stock_changes_lines}"
    sns.publish(TopicArn=topic_arn, Message=message, Subject=subject)


def send_in_stock_message(
    sns, topic_arn: str, checker_name: str, stock_changes: Collection[str]
):
    stock_changes_lines = "\n".join(stock_changes)
    subject = f"[{checker_name}] - New items in stock!"
    message = f"Hurry up! New items in stock:\n{stock_changes_lines}"
    sns.publish(TopicArn=topic_arn, Message=message, Subject=subject)


def send_gone_message(sns, topic_arn: str, checker_name: str):
    subject = f"[{checker_name}] - All items gone!"
    message = f"Too slow..."
    sns.publish(TopicArn=topic_arn, Message=message, Subject=subject)


def lambda_handler(event, context):
    parser = NeweggParser(event["url"])
    parser.fetch()
    items = parser.get_items()

    # Continue to print in-stock items to the logs.
    for item, in_stock in items.items():
        if in_stock:
            LOGGER.info(item)

    # Compare to S3 and update.
    status, stock_changes = compare_to_s3(items, event["s3Bucket"], event["s3Object"])

    # Return immediately if no change.
    if status == Status.NO_CHANGE:
        return

    # Otherwise send SNS message.
    sns = boto3.client("sns", region_name="us-east-1")
    if status == Status.INITIALIZED:
        send_init_message(sns, event["topicArn"], event["s3Object"], stock_changes)
    if status == Status.ITEMS_IN_STOCK:
        send_in_stock_message(sns, event["topicArn"], event["s3Object"], stock_changes)
    if status == Status.ITEMS_GONE:
        send_gone_message(sns, event["topicArn"], event["s3Object"])
```
We're now adding `s3Bucket` and `s3Object` attributes that should be part of the `event`.
Just like with `topicArn`, we can have Terraform fill in the `s3Bucket` value. Also, note
that `s3Object` is passed in as a `checker_name` variable to these functions. This
attribute does double-duty in that it gives this instance of the stock checker a name.
Useful, because we can deploy more than one in parallel.

Let's go to Terraform and add those attributes to our `event` in `cloudwatch.tf`:
```hcl
input = jsonencode({
  url      = var.url
  topicArn = aws_sns_topic.newegg_stock_checker.arn
  s3Bucket = aws_s3_bucket.newegg.id
  s3Object = "gtx3070"
})
```

Also, we need to flip the `is_enabled` switch back on - remember that we stopped it at
the end of the last part. Zip up the code, run Terraform, and now that it's deployed, we
should see an email come very shortly...

## It's borked...

Okay.. so this might be something to add to the list of "things we want to be emailed
about". Now we're no stranger to running into errors - we deliberately stumbled upon
them earlier as a learning experience - but it's not all "no news is good news". We
need to know if something's broken. Enter: *CloudWatch Alarms.*

## Understanding CloudWatch Alarms

To understand what a CloudWatch Alarm is, it's important to be aware of CloudWatch
Metrics. Almost every AWS service reports metrics that span across your resources,
always tabluating things that you may want to monitor. In the CloudWatch metrics
console, we can drill down to "Lambda" > "By Function Name", and we can graph the
number of errors our function is generating. Sure enough, it's not zero.

![CloudWatch Metrics](/img/cloudwatch_metrics.png)

An Alarm is simply a way to define thresholds of a metric and act on them through
an SNS notification (or autoscaling change). We would want to create an alarm
whenever there is at least 1 error. Here's how that would look in Terraform:
```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name  = "${var.function_name}-errors"
  namespace   = "AWS/Lambda"
  metric_name = "Errors"

  # Looking over the last 60 seconds.
  period             = 60
  evaluation_periods = 1

  # Was there >= 1 error?
  comparison_operator = "GreaterThanOrEqualToThreshold"
  threshold           = 1
  statistic           = "Sum"

  alarm_actions = [aws_sns_topic.newegg_stock_checker.id]
  ok_actions    = [aws_sns_topic.newegg_stock_checker.id]

  dimensions = {
    "FunctionName" = var.function_name
  }
}
```
This alarm is basically stating that if there's been at least one error within the last
60-second window, trigger the ALARM state. Once we're in an ALARM state, the SNS topic
is sent a notification. Notice that we just reused the very same SNS topic we used to
get our normal updates. Also, once the errors have been fixed, the alarm should drop
below the triggering threshold, and we should get a confirmation email that the alarm
has been resolved. Terraform it, and yep, sure enough, I get an email:

> ALARM: "newegg-stock-checker-errors" in US East (N. Virginia)

## Ever get that feeling of deja vu?

Okay, so let's fix our error. CloudWatch logs.. survey says... `[ERROR] ClientError:
An error occurred (AccessDenied) when calling the GetObject operation: Access Denied`.
A classic permissions error, since now the Lambda function wants to write to the S3
bucket. So let's grant it.

```hcl
# Allow S3 access

data "aws_iam_policy_document" "s3_access" {
  statement {
    actions   = ["s3:GetObject*", "s3:PutObject*"]
    resources = ["${aws_s3_bucket.newegg.arn}/*"]
  }

  statement {
    actions   = ["s3:ListBucket"]
    resources = [aws_s3_bucket.newegg.arn]
  }
}

resource "aws_iam_policy" "s3_access" {
  name_prefix = "s3-access-"
  policy      = data.aws_iam_policy_document.s3_access.json
}

resource "aws_iam_policy_attachment" "s3_access" {
  name       = "s3-access"
  policy_arn = aws_iam_policy.s3_access.arn
  roles = [
    aws_iam_role.newegg_stock_checker.id
  ]
}
```
The `GetObject` and `PutObject` are self-explanitory, but why the `ListBucket`? I
actually just noticed as of this very second - without the `ListBucket` permission,
attempting to call `GetObject` on a non-existent object will actually return a 403
and give an "Access Denied" error. With `ListBucket` in place, AWS will correctly
return a 404 `NoSuchKey` error, which is specifically what we're trying to catch.
Okay, let's Terraform this, and...

## VERY NICE!! GREAT SUCCESS!!

![VERY NICE I LIKE](/img/great_success.png)

Well, yes, it's still out of stock - as it has been the whole time creating this
tutorial - but at this point, we've fully created an automated serverless Newegg
stock checker, and we hardly had to do any clicking and pointing in the AWS console.
We've seen the firsthand benefits of Terraform and infrastructure-as-code to create
a nifty project in AWS.

## But you promised we could deploy more than one stock checker?

Yes you can, and I'll show off a neat way to do that in a "bonus" part. For now, let's
hope I can snag a GTX 3070...
