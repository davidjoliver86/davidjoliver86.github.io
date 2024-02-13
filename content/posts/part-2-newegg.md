---
title: "Part 2/4 - Create and deploy an automated, serverless Newegg stock checker using Python, Lambda, and Terraform"
date: 2020-11-30
draft: false
tags:
  - aws
  - python
  - terraform
  - lambda
---
> ⚠️ This tutorial is a relic of its time period, back when the COVID-19 pandemic was
  beginning to surge. While the information is still relevant and factual, this tool
  was designed with once-in-a-lifetime supply chain conditions in mind, and the end
  result may not be as useful today as it was back when this article was written.

In this part, we're going to focus on getting a Terraform environment setup. Instead
of clicking around in the AWS web console, we're going to be writing declarative config
files that will define our "infrastructure-as-code". We're going to also make some
small changes to our parser from the last part to get it ready to deploy as a Lambda
function.

## We're transforming Mars into a habitible planet?

I wish. But no, we're just going to be writing code in which we define the AWS
resources we want to create, and Terraform will take that code and translate it into
AWS API calls such that what's deployed matches how we've configured it in code. This
being code means that we can treat it as code and reap the benefits. Need to revert
a change? Concerned about what nosey auditors want to see? Version control. Creating
a bunch of similar resources? For-loops.

## Setting up your Terraform environment

Download Terraform CLI from https://terraform.io and extract it, ideally somewhere on
your PATH. Yes, it's a single binary. The wonders of Golang. :) Next, you'll want to
setup your AWS credentials. If you haven't already, my preferred practice is to define
them in your `~/.aws/credentials` file as such:
```
[default]
aws_access_key_id=<YOUR ACCESS KEY>
aws_secret_access_key=<YOUR SECRET KEY>
```

## Hello world of sorts

The first thing you should do is verify that you're able to download the Terraform AWS
provider. Create a `modules.tf` file with the following contents:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}
```

Then run `terraform init`. If it doesn't error out, you're good to go. Next, we'll
declare our AWS provider. Also in `modules.tf`:
```hcl
provider "aws" {
  region = var.region
}
```

Notice that we've also declared a variable; it's generally a good practice to use
variables. We'll define that, and some others, in `variables.tf`:
```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "function_name" {
  type    = string
  default = "newegg-stock-checker"
}

variable "function_zipfile_name" {
  type    = string
  default = "newegg.zip"
}
```

Now let's define the resources we need to create a Lambda function, and then turn our
stock checker into one.

## Okay, what's this "Lambda function" you keep talking about?

AWS Lambda is a service that allows you to execute code without worrying about having
to maintain the underlying server. Think "functions-as-a-service". It's probably my
favorite AWS service due to its flexiblity; many popular AWS services can hook into
Lambda, and it's a crucial part of an event-driven architectures.

In the case of our stock checker, we can deploy the code and setup an event handler
to run it on a schedule without having to spin up an EC2 resource and define a cron
job. And it's likely more efficient from a cost perspective too.

## Making our stock checker a Lambda function

AWS Lambda supports many different languages, and each of them has their own process
to define your function's entrypoint. In the case of Python, we'll be adding a
`lambda_handler(event, context)` function. `event` is usually a dictionary, containing
arbitrary data from the event that invoked it, and `context` is an object containing
metadata about the execution context.

In our case, we can basically copy the `if __name__ == '__main__'` into this handler:
```python
def lambda_handler(event, context):
    parser = NeweggParser(event["url"])
    parser.fetch()
    print(parser.get_items_in_stock())
```
Notice that we moved the `base_url` argument of the `NeweggParser` into an attribute
of the event. When we define the "cron" aspect of this later, we can pass in the URL
we want to check as part of whatever arbitrary JSON we want to pass into the `event`.
So even though we're stock checking GTX 3070s, maybe I might want a Radeon instead?

## I'm afraid I can't let you do that...

We'll now need to define an IAM role for this function in order to allow it to fully
execute. This will allow us to define precisely how it can interact with other AWS
services. Following the principle of least privilege, if we don't explicitly grant
permissions to do something, they're not allowed.

All Lambda functions require a minimal set of permissions for them to log their
activity. This comes in the form of an AWS-managed policy that we can attach to a role.

What we'll also need to do is define that the role itself is one for use with AWS
Lambda. So let's write our first Terraform to create our role, in an `iam.tf` file:

```hcl
data "aws_iam_policy_document" "lambda_trust_policy" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "newegg_stock_checker" {
  name               = "newegg-stock-checker"
  assume_role_policy = data.aws_iam_policy_document.lambda_trust_policy.json
}

resource "aws_iam_role_policy_attachment" "newegg_stock_checker_base" {
  role       = aws_iam_role.newegg_stock_checker.id
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```
So a couple of practices should have stood out from this example:

First of all, normally an IAM policy document is just a JSON string. By using the
`aws_iam_policy_document` resource instead, we have a well-defined interface that can
generate this JSON for us. I'm not alone in speaking from past experience having been
bitten by simple JSON formatting errors before.

Second, note that we're referring to the exported `id` element in our role policy
attachment. Rather than having to repeatedly hardcode the name of the role in any
resource that will reference it, it's a far better practice to rely on attributes
that can be exported from it. Use `id` if possible. The Terraform docs are a great
resource on what you can utilize. But it gets better too - we can cross-reference it
in other files as well, and that's exactly what we're going to do with our Lambda
function.

## Creating the function

Using our variables, and our role, we can create our function in a `lambda.tf`:
```hcl
resource "aws_lambda_function" "newegg_stock_checker" {
  function_name    = var.function_name
  filename         = var.function_zipfile_name
  source_code_hash = filebase64sha256(var.function_zipfile_name)
  role             = aws_iam_role.newegg_stock_checker.arn
  handler          = "newegg.lambda_handler"
  runtime          = "python3.8"
}
```
If you're paying attention to what I'm calling the file, notice that I'm naming them
after the AWS service they represent. This is not a requirement, but an opinionated
recommendation. Let's take note of a few things:

* The deployment package is expected to be a zip file. For a simple function without
any third-party libraries, it's as simple as zipping up our `newegg.py` from earlier.
* It's expecting to find the zipfile in the same folder as all the `.tf` files. We can
alter this if needed, and in a larger-scale deployment, I would prefer using S3, but
this is fine for our example.
* Note that the handler is in the form of `<module.handler_function>`. So if we called
this `amazon.py` to check Amazon, it'd be `amazon.lambda_handler`. And you could
actually call `lambda_handler` whatever you want, but I prefer to just stick with that
by convention.

## Okay.. let's get terraforming
We're now ready to run Terraform and create our function and IAM role. And the beauty
of Terraform is that roughly 99% of the time, the dependency ordering just works. So
we don't *have* to explicitly tell it to create the role first, then the function. It
usually just knows. And in the rare event that it doesn't, you can override it.

My preferred method to run Terraform is to generate a plan first, then apply the plan
only if I'm satisfied with the changes. So let's run `terraform plan -out tf.plan`...

```
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_role.newegg_stock_checker will be created
  + resource "aws_iam_role" "newegg_stock_checker" {
      + arn                   = (known after apply)
      + assume_role_policy    = jsonencode(
            {
              + Statement = [
                  + {
                      + Action    = "sts:AssumeRole"
                      + Effect    = "Allow"
                      + Principal = {
                          + Service = "lambda.amazonaws.com"
                        }
                      + Sid       = ""
                    },
                ]
              + Version   = "2012-10-17"
            }
        )
      + create_date           = (known after apply)
      + force_detach_policies = false
      + id                    = (known after apply)
      + max_session_duration  = 3600
      + name                  = "newegg-stock-checker"
      + path                  = "/"
      + unique_id             = (known after apply)
    }

  # aws_iam_role_policy_attachment.newegg_stock_checker_base will be created
  + resource "aws_iam_role_policy_attachment" "newegg_stock_checker_base" {
      + id         = (known after apply)
      + policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
      + role       = (known after apply)
    }

  # aws_lambda_function.newegg_stock_checker will be created
  + resource "aws_lambda_function" "newegg_stock_checker" {
      + arn                            = (known after apply)
      + filename                       = "newegg.zip"
      + function_name                  = "newegg-stock-checker"
      + handler                        = "newegg.lambda_handler"
      + id                             = (known after apply)
      + invoke_arn                     = (known after apply)
      + last_modified                  = (known after apply)
      + memory_size                    = 128
      + publish                        = false
      + qualified_arn                  = (known after apply)
      + reserved_concurrent_executions = -1
      + role                           = (known after apply)
      + runtime                        = "python3.8"
      + signing_job_arn                = (known after apply)
      + signing_profile_version_arn    = (known after apply)
      + source_code_hash               = "OglgTDSdxTM/32aDilEJlSzYmTLT/ubrEg3sWSe9jmw="
      + source_code_size               = (known after apply)
      + timeout                        = 3
      + version                        = (known after apply)

      + tracing_config {
          + mode = (known after apply)
        }
    }

Plan: 3 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

This plan was saved to: tf.plan

To perform exactly these actions, run the following command to apply:
    terraform apply "tf.plan"
```

Looks good. Let's apply that plan now.
```
aws_iam_role.newegg_stock_checker: Creating...
aws_iam_role.newegg_stock_checker: Creation complete after 1s [id=newegg-stock-checker]
aws_iam_role_policy_attachment.newegg_stock_checker_base: Creating...
aws_lambda_function.newegg_stock_checker: Creating...
aws_iam_role_policy_attachment.newegg_stock_checker_base: Creation complete after 1s [id=newegg-stock-checker-20201202055221258600000001]
aws_lambda_function.newegg_stock_checker: Still creating... [10s elapsed]
aws_lambda_function.newegg_stock_checker: Creation complete after 16s [id=newegg-stock-checker]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed
```

Alright - now let's check it out.

## Testing our function
Now we can start clicking around the AWS console. Go to Lambda, and you should see your
function, complete with its code. Click on "Test" to test it, aaand.. nope.

![Lambda test event](/img/lambda_test_event.png)

Fortunately, this is exactly what we want. This is where you define what gets passed in
into the `event` argument of the `lambda_handler`. Notice that within the handler, it's
looking for `event["url"]`? So here's where we can create a new test event with a simple
JSON object with the URL we tried last time:
```json
{
  "url": "https://www.newegg.com/p/pl?d=gtx+3070&N=100007709&isdeptsrh=1&PageSize=96"
}
```
Create, test, and... 💥

![Lambda results](/img/lambda_results.png)

So the good news is we now have a working Lambda function that we created entirely
in Terraform (okay, and Python too). The bad news is - they're still out of stock.
Well, hopefully while we wait for stock to replenish, we can start adding more
functionality down the line. We can set it to run automatically, and email us whenever
there's a stock change. We'll do that in part 3.