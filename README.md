# About

EC2 instances are volatile and can be recycled at any time without warning.
Amazon recommends running them under Auto Scaling Groups to ensure overall
service availability, but it's easy to forget that instances can suddenly fail
until it happens in the early hours of the morning during a holiday.

Chaos Lambda increases the rate at which these failures occur during business
hours, helping teams to build services that handle them gracefully.


# Quick setup

Create the lambda function in the region you want it to target using the
`cloudformation/templates/lambda_standalone.json` CloudFormation template.
There are two parameters you may want to change:
* `Schedule`: change if the default run times don't suit you (once per hour
  between 10am UTC and 4pm UTC, Monday to Friday); see
  http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html
  for documentation on the syntax.
* `DefaultMode`: by default all Auto Scaling Groups in the region are targets;
  set this to `off` to switch to an opt-in mode, where only ASGs with a
  `chaos-lambda-termination` tag (see below) are affected.


# Notifications

To receive notifications if the lambda function fails for any reason, create
another stack using the `cloudformation/templates/alarms.json` template.  This
takes the lambda function name (something similar to
`chaos-lambda-ChaosLambdaFunction-EM2XNWWNZTPW`) and the email address to
send the alerts to.


# Probability of termination

Whenever the lambda is triggered it will potentially terminate one instance per
Auto Scaling Group in the region.  By default the probability of terminating an
ASG's instance is 1 in 6.  This probability can be overridden by setting a
`chaos-lambda-termination` tag on the ASG with a value between 0.0 and 1.0,
where 0.0 means never terminate and 1.0 means always terminate.


# Enabling/disabling

The lambda is triggered by a CloudWatch Events rule, the name of which can be
found from the `ChaosLambdaFunctionOutput` output of the lambda stack.  Locate
this rule in the AWS console under the Rules section of the CloudWatch service,
and you can disable or enable it via the `Actions` button.


# Regions

By default the lambda will target instances running in the same region.  It's
generally a good idea to avoid cross-region actions, but at the time of writing
lambda functions cannot be run in some regions.

To override the default choice of region you will need to create a
`src/regions.txt` file containing one or more newline-separated AWS region
names, eg:

```
ap-south-1
us-west-1
```

Blank lines and any whitespace surrounding the names are ignored.

Generate a `chaos-lambda.zip` file containing both the region list and the code
by running `make zip`, and upload this to a S3 bucket that was created in the
same region you plan to run the lambda from.

Create a stack using the `cloudformation/templates/lambda.json` CloudFormation
template (not the "standalone" one mentioned in the quick setup).  This
requires two additional parameters for locating the zip file: `S3Bucket` and
`S3Key`.


# Log messages

Chaos Lambda log lines always start with a timestamp and a word specifying the
event type.  The timestamp is of the form `YYYY-MM-DDThh:mm:ssZ`, eg
`2015-12-11T14:00:37Z`, and the timezone will always be `Z`.  The different
event types are described below.

## bad-probability

`<timestamp> bad-probability [<value>] in <asg name>`

Example:

`2015-12-11T14:07:21Z bad-probability [not often] in test-app-ASG-7LJI5SY4VX6T`

If the value of the `chaos-lambda-termination` tag isn't a number between `0.0`
and `1.0` inclusive then it will be logged in one of these lines.  The square
brackets around the value allow CloudWatch Logs to find the full value even if
it contains spaces.

## result

`<timestamp> result <instance id> is <state>`

Example:

`2015-12-11T14:00:40Z result i-fe705d77 is shutting-down`

After asking EC2 to terminate each of the targeted instances the new state of
each is logged with a `result` line.  The `<state>` value is taken from the
`code` property of the `InstanceState` AWS type described at
http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_InstanceState.html

## targeting

`<timestamp> targeting <instance id> in <asg name>`

Example:

`2015-12-11T14:00:38Z targeting i-168f9eaf in test-app-ASG-1LOMEKEVBXXXS`

The `targeting` lines list all of the instances that are about to be
terminated, before the `TerminateInstances` call occurs.

## triggered

`<timestamp> triggered <region>`

Example:

`2015-12-11T14:00:37Z triggered eu-west-1`

Generated when the lambda is triggered, indicating the region that will be
affected.
