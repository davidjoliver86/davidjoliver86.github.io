---
title: "Part 3/4 - Create and deploy an automated, serverless Newegg stock checker using Python, Lambda, and Terraform"
date: 2020-12-03
draft: false
tags:
  - aws
  - python
  - terraform
  - lambda
  - cloudwatch
  - sns
---
> ⚠️ This tutorial is a relic of its time period, back when the COVID-19 pandemic was
  beginning to surge. While the information is still relevant and factual, this tool
  was designed with once-in-a-lifetime supply chain conditions in mind, and the end
  result may not be as useful today as it was back when this article was written.

We've got a working Lambda function now, so let's start adding in the other pieces of
the AWS puzzle to turn this into a lean, mean, automated stock-checking machine.

## How often should we run it?

Very good question. For a hot product that may literally last a few minutes on the
"shelf", it sounds like we'd want to check pretty frequently. Ideally within timeframes
of a minute. But there's one all-important thing to consider that holds true more than
anything else in AWS, or cloud computing in general.

Money. Moolah. Dinero. Cash money. Cash rules everything around me - C.R.E.A.M.

There are two components to Lambda pricing - number of requests, and the total duration
of time spent in the function. In us-east-1, these costs are as follows:

| | Price |
| --- | --- |
| Requests | $0.20 per 1M requests |
| Duration | $0.0000166667 for every GB-second |

And if you're wondering "what in the hell is a GB-second", you're probably not alone.
Each function allows you to allocate an amount of memory to its execution environment.
Obviously, the more memory you allocate, the more you should have to pay. If your code
runs for exactly 1 second in an environment with 1024 MB, you are charged
$0.0000166667 for the duration. The vast majority of Lambda functions are well-suited
to run in an environment with the minimum amount of 128 MB, or 1/8th that.

As you may recall from the Terraform code in part 2, we created this function with
the minimum memory allocation of 128 MB. Unless you know you're working on something
that may become memory or CPU intensive, stick to the minimum at first. After invoking
this function a few times, it does appear to have a duration of 1 second. So what would
it cost to run this every minute?

## EventBridge - because too many things already started with "Cloud"

What used to be called CloudWatch Events is now Amazon EventBridge. I'm okay with that,
because frankly, too many AWS offerings already start with "Cloud". EventBridge is -
quite figuratively - a bridge between events emitted by other AWS services, and targets
to catch them and perform actions. Those events can also simply be cron tasks, and you
set them to run as frequently as once every minute. One of the actions we can perform
is to invoke our stock checker Lambda function. So we can set up our stock checker to
run every minute.

## Doing the math

Let's assume a 31-day month. 60 minutes/hour x 24 hours/day x 31 days/month = 44,640.
And if every invocation averages out to be 1 second, 44,600 x 0.000166667 GB/second
x 1/8 of a GB = $0.93. And the 44,640 invocations x 0.20/1,000,000 = $0.008928. Round
up to a penny, so that's $0.94/month. Here you go.

![94 cents](/img/94cents.jpg)

## Terraform the EventBridge Rule

We'll need to Terraform both sides of the equation - the event *rule* for the "every
minute" event, and the event *target* to point to the Lambda function to invoke. Also
note that for now, Terraform still refers to these resources through their old name,
but the CloudWatch console also does too for now. So, yay consistency. Or CloudConsistency.
```hcl
resource "aws_cloudwatch_event_rule" "every_minute" {
  name_prefix         = "every-minute-"
  schedule_expression = "rate(1 minute)"
  description         = "Runs every minute"
}

resource "aws_cloudwatch_event_target" "newegg_stock_checker" {
  rule = aws_cloudwatch_event_rule.every_minute.id
  arn  = aws_lambda_function.newegg_stock_checker.arn

  input = jsonencode({
    url = var.url  // defined in variables.tf
  })
}
```
Also note the use of `jsonencode` instead of hard-coding raw JSON in heredoc format.
This allows you to use the language primitives in a way that seamlessly translate to
JSON. Looks like our plan and apply succeeded, so we should be seeing some action from
this function very shortly.

... I hope.

\*crickets\*

Okay.. we waited a few minutes, but it looks like there haven't been any invocations of
this function. Well, it turns out this is a common "gotcha" about Terraforming a Lambda
function to run on a schedule. From the [Terraform aws_cloudwatch_event_rule docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_event_target):

> In order to be able to have your AWS Lambda function or SNS topic invoked by an EventBridge rule, you must setup the right permissions using aws_lambda_permission or aws_sns_topic.policy. More info [here](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/resource-based-policies-cwe.html).

So now let's add our `aws_lambda_permission` to allow our ~~CloudWatch~~ EventBridge rule to
invoke it.
```hcl
resource "aws_lambda_permission" "newegg_stock_checker_every_minute" {
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.newegg_stock_checker.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.every_minute.arn
}
```
Terraform plan/apply again, and now we see the function getting executed! Let's take a look
at the logs:
```
START RequestId: ecaad111-c954-4fae-a35d-0247c924cdaf Version: $LATEST
[]
END RequestId: ecaad111-c954-4fae-a35d-0247c924cdaf
REPORT RequestId: ecaad111-c954-4fae-a35d-0247c924cdaf	Duration: 274.30 ms	Billed Duration: 275 ms	Memory Size: 128 MB	Max Memory Used: 57 MB
```
So yes, unfortunately there's still no stock, but it does look to be running every minute. It
also looks like I dramatically overestimated the duration - I was getting my figure from
invoking it with the test event. And it appears to be right-sized for memory utilization.
But come on... wasn't the point of this stock checker to make it so I don't have to keep my
eyes glued on a page 24/7? Luckily we don't need to.

## Setting up SNS for email notifications

What we really want to do is have it send an email as soon as there's some stock
available. And the best way to do that in an environment where we're in full control
over who to send emails to (me) is through SNS.

It's pretty simple, as the first "S" in the name implies. We'll create an SNS topic
for our Lambda function to publish messages to, then we'll add my email as a
subscription endpoint. It's basically a classic pub/sub implementation. In `sns.tf`:
```hcl
resource "aws_sns_topic" "newegg_stock_checker" {
  name_prefix = "newegg-stock-checker-"
}
```
Once we've created our topic with a plan/apply, we'll add an email endpoint. While
it is technically possible to Terraform it, the [aws_sns_topic_subscription docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sns_topic_subscription)
advise against it:

> These are unsupported because the endpoint needs to be authorized and does not generate an ARN until the target email address has been validated. This breaks the Terraform model and as a result are not currently supported.

So we'll just go in the console and add our subscription, then click the confirmation
link in the email that follows:

![SNS topic subscription](/img/sns_topic_subscription.png)

## Integrating SNS

Now we'll need to actually integrate SNS into the workflow. The execution environment
for our function just so happens to include `boto3`, which is what we'll use to have
our function send messages to the SNS topic, from which the SNS topic relays that
message to all its subscribers (i.e. me). In `newegg.py`, we'll add two imports:
```py
import boto3
import json
```

And add some more to the `lambda_handler`:
```py
def lambda_handler(event, context):
    parser = NeweggParser(event["url"])
    parser.fetch()
    items_in_stock = parser.get_items_in_stock()

    # Continue to print it to the logs.
    LOGGER.info(items_in_stock)

    # Send an SNS message too.
    sns = boto3.client("sns", region_name="us-east-1")
    sns.publish(
        TopicArn=event["topicArn"],
        Subject="Newegg Stock Check Alert",
        Message=json.dumps(items_in_stock),
    )
```
Notice that we're getting the `topicArn` item from the `event` dictionary, even though
we didn't define it before. Luckily, we can derive that ARN from within Terraform. This
is a much better practice in my opinion than simply hardcoding the `topicArn` in the
function itself. In `cloudwatch.tf`:
```hcl
resource "aws_cloudwatch_event_target" "newegg_stock_checker" {
  rule = aws_cloudwatch_event_rule.every_minute.id
  arn  = aws_lambda_function.newegg_stock_checker.arn

  input = jsonencode({
    url      = var.url
    topicArn = aws_sns_topic.newegg_stock_checker.arn
  })
}
```
Now when we re-zip our function code, we hope that'll trigger an update. Let's see if
Terraform wants to:
```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_cloudwatch_event_target.newegg_stock_checker will be updated in-place
  ~ resource "aws_cloudwatch_event_target" "newegg_stock_checker" {
        arn            = "arn:aws:lambda:us-east-1:MY_ACCOUNT_ID:function:newegg-stock-checker"
        event_bus_name = "default"
        id             = "every-minute-20201204060835803400000001-terraform-20201204060837710900000002"
      ~ input          = jsonencode(
          ~ {
              + topicArn = "arn:aws:sns:us-east-1:MY_ACCOUNT_ID:newegg-stock-checker-20201204064825927800000001"
                url      = "https://www.newegg.com/p/pl?d=gtx+3070&N=100007709&isdeptsrh=1&PageSize=96"
            }
        )
        rule           = "every-minute-20201204060835803400000001"
        target_id      = "terraform-20201204060837710900000002"
    }

  # aws_lambda_function.newegg_stock_checker will be updated in-place
  ~ resource "aws_lambda_function" "newegg_stock_checker" {
        arn                            = "arn:aws:lambda:us-east-1:MY_ACCOUNT_ID:function:newegg-stock-checker"
        filename                       = "newegg.zip"
        function_name                  = "newegg-stock-checker"
        handler                        = "newegg.lambda_handler"
        id                             = "newegg-stock-checker"
        invoke_arn                     = "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:MY_ACCOUNT_ID:function:newegg-stock-checker/invocations"
      ~ last_modified                  = "2020-12-04T06:20:15.206+0000" -> (known after apply)
        layers                         = []
        memory_size                    = 128
        publish                        = false
        qualified_arn                  = "arn:aws:lambda:us-east-1:MY_ACCOUNT_ID:function:newegg-stock-checker:$LATEST"
        reserved_concurrent_executions = -1
        role                           = "arn:aws:iam::MY_ACCOUNT_ID:role/newegg-stock-checker"
        runtime                        = "python3.8"
      ~ source_code_hash               = "OglgTDSdxTM/32aDilEJlSzYmTLT/ubrEg3sWSe9jmw=" -> "/Wi4bcZA4A6ziNkHDTqLw1GPXZDLGgDp14uBud/JwAA="
        source_code_size               = 1419
        tags                           = {}
        timeout                        = 3
        version                        = "$LATEST"

        tracing_config {
            mode = "PassThrough"
        }
    }

Plan: 0 to add, 2 to change, 0 to destroy.
```
Looks good, so let's apply, and play the waiting game... eh, the waiting game sucks,
let's play Hungry Hungry Hippos instead.

Turns out the reason the waiting game sucks is.. we've hit an error.
```py
[ERROR] AuthorizationErrorException: An error occurred (AuthorizationError) when calling the Publish operation: User: arn:aws:sts::MY_ACCOUNT_ID:assumed-role/newegg-stock-checker/newegg-stock-checker is not authorized to perform: SNS:Publish on resource: arn:aws:sns:us-east-1:MY_ACCOUNT_ID:newegg-stock-checker-20201204064825927800000001
Traceback (most recent call last):
  File "/var/task/newegg.py", line 102, in lambda_handler
    sns.publish(TopicArn=event["topicArn"], Message=json.dumps(items_in_stock))
  File "/var/runtime/botocore/client.py", line 357, in _api_call
    return self._make_api_call(operation_name, kwargs)
  File "/var/runtime/botocore/client.py", line 676, in _make_api_call
    raise error_class(parsed_response, operation_name)
```
So it looks like the function doesn't have permission to publish to that topic. Here's
the principle of least privilege in action - grant only the bare minimum capabilities
you possibly can. This qualifies it as a bare minimum, so let's add a policy to `iam.tf`:
```tf
# Allow publishing to SNS

data "aws_iam_policy_document" "publish_to_sns" {
  statement {
    actions   = ["SNS:Publish"]
    resources = [aws_sns_topic.newegg_stock_checker.arn]
  }
}

resource "aws_iam_policy" "publish_to_sns" {
  name_prefix = "publish-to-sns-"
  policy      = data.aws_iam_policy.publish_to_sns.json
}

resource "aws_iam_policy_attachment" "publish_to_sns" {
  name       = "publish-to-sns"
  policy_arn = aws_iam_policy.publish_to_sns.arn
  roles = [
    aws_iam_role.newegg_stock_checker.id
  ]
}
```
And once we plan/apply these changes, we now see email alerts!

![SNS email alert](/img/sns_email.png)

## AAAAGH THE SPAM
So it works... a little too well. Okay, clearly there's an issue with these alerts,
since we're getting them every minute, and the results haven't changed. We only
want to get an alert when it matters - that is, when there's actually a change in
stock. In the next - and final - part, we'll introduce a stateful data store so that
we can get notifications when there's a meaningful stock change. Also, given the
Lambda errors we hit, it's probably also equally useful to get alerts if the function
errors out, so we'll add those too. But for right now...

## Cutting off the spam
The most efficient way to do this seems to be just adding a `is_disabled = false`
statement to the `aws_cloudwatch_event_rule` resource. So I'll do that for now, and
we can re-enable it once we've got better alerting in place.