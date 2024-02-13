---
title: "Post-mortem and bonus: serverless Newegg stock checker"
date: 2020-12-09
draft: false
tags:
  - aws
  - terraform
---
> ⚠️ This tutorial is a relic of its time period, back when the COVID-19 pandemic was
  beginning to surge. While the information is still relevant and factual, this tool
  was designed with once-in-a-lifetime supply chain conditions in mind, and the end
  result may not be as useful today as it was back when this article was written.

Quick update - in the last approx. 24-hour period since I deployed my stock checker
to check on GTX 3070 stock, I actually did get a few hits back. Unfortunately, most
of those wound up being combo packs that included a motherboard. Many of them were
Intel motherboards, whereas I'm actually targeting a Ryzen build. Oh, and they all
went back out of stock within about 2 minutes. So yeah - 60-second granularity can make
the difference between seeing an item go in and out of stock, and never getting to see
the item even hit the virtual shelf!

## Oops..
It turns out I actually forgot to add one crucial thing that can make the difference
between actually being able to get a product - a link back in the email to the search
page. Since we're already passing that URL in the `event`, we just need to pass it
to the SNS email functions.
```py
def send_in_stock_message(
    sns, topic_arn: str, url: str, checker_name: str, stock_changes: Collection[str]
):
    stock_changes_lines = "\n".join(stock_changes)
    subject = f"[{checker_name}] - New items in stock!"
    message = f"Hurry up! New items in stock:\n{stock_changes_lines}\n{url}"
    sns.publish(TopicArn=topic_arn, Message=message, Subject=subject)
```

And this may come particularly handy when we're able to deploy more than one stock
checker, in case there's a second product we wanted to monitor. And in my case,
that'll be a Radeon 6800.

## Bonus section: deploying multiple instances

The nice thing about designing the function around an `event` object is that we've
basically abstracted the same logic that can be applied to searching for a different
product on Newegg. So in theory, all we actually need to do is create a separate
`aws_cloudwatch_event_target` with a different input. So let's copy this from
`cloudwatch.tf`:
```hcl
resource "aws_cloudwatch_event_target" "newegg_stock_checker" {
  rule = aws_cloudwatch_event_rule.every_minute.id
  arn  = aws_lambda_function.newegg_stock_checker.arn

  input = jsonencode({
    url      = var.url
    topicArn = aws_sns_topic.newegg_stock_checker.arn
    s3Bucket = aws_s3_bucket.newegg.id
    s3Object = "gtx3070"
  })
}
```
And.. there's kind of a small problem. We have `url` as a Terraform variable. Now we
could remove the `url` variable and resume copy-pasting this resource.. but that's also
going to create potentially a lot of repetitive code. Do we have any better options?

## For each of us: a better option...

The `for_each` construct allows us to iterate over a data structure to create multiple
"duplicates" of the same resource, with different variables that can be substituted as
needed. Terraform has provided a `count` resource for a long time, but in general I
prefer the `for_each` since it does a better job encapsulting the individual resources.
Having run Terraform plans with large resources under a `count`, the plan gets very
messy when something in the middle gets removed for whatever reason.

We can define a single Terraform variable that stores our projects in a map - this
effectively replaces the single `url` variable:
```hcl
variable "checkers" {
  type        = map(string)
  description = "URLs for stock checkers; key is used as the scraper name and S3 object"
  default = {
    gtx3070    = "https://www.newegg.com/p/pl?d=gtx+3070&N=100007709&isdeptsrh=1&PageSize=96"
    radeon6800 = "https://www.newegg.com/p/pl?d=radeon+6800&N=100007709&isdeptsrh=1&PageSize=96"
  }
}
```

In `cloudwatch.tf`, we can redefine the event target with a `for_each` expression:
```hcl
resource "aws_cloudwatch_event_target" "newegg_stock_checker" {
  for_each = var.checkers
  rule     = aws_cloudwatch_event_rule.every_minute.id
  arn      = aws_lambda_function.newegg_stock_checker.arn

  input = jsonencode({
    url      = each.value
    topicArn = aws_sns_topic.newegg_stock_checker.arn
    s3Bucket = aws_s3_bucket.newegg.id
    s3Object = each.key
  })
}
```

And doing so causes a small change behind the scenes - the original resource is
destroyed to make way for the new "multi-resource":
```
Terraform will perform the following actions:

  # aws_cloudwatch_event_target.newegg_stock_checker will be destroyed
  - resource "aws_cloudwatch_event_target" "newegg_stock_checker" {
  ...

  # aws_cloudwatch_event_target.newegg_stock_checker["gtx3070"] will be created
  + resource "aws_cloudwatch_event_target" "newegg_stock_checker" {
  ...

  # aws_cloudwatch_event_target.newegg_stock_checker["radeon6800"] will be created
  + resource "aws_cloudwatch_event_target" "newegg_stock_checker" {
  ...

Plan: 2 to add, 0 to change, 1 to destroy.
```
Also, since we modified our function, zip it up again and run Terraform to update the
code. After I applied the plan, it only took one minute for the `[radeon6800] - First run successful`
email to appear! Creating or modifying a new stock checker now just comes down to
adding one line of code to generate a new event target.

I hope this has been an informative, and at times, entertaining series. Now I await
the very next follow-up - actually purchasing a new graphics card and getting my new
gaming system setup. But I might blog about a few other things before then. :)
