# About

EC2 instances are volatile and can be recycled at any time without warning.
Amazon recommends running them under Auto Scaling Groups to ensure overall
service availability, but it's easy to forget that instances can suddenly fail
until it happens in the early hours of the morning during a holiday.

Chaos Lambda increases the rate at which these failures occur during business
hours, helping teams to build services that handle them gracefully.

Whenever the lambda is triggered it will potentially terminate one instance per
Auto Scaling Group in the region.  By default the probability of terminating an
ASG's instance is 1 in 6.  This probability can be overridden by setting a
`chaos-lambda-termination` tag on the ASG with a value between 0.0 and 1.0,
where 0.0 means never terminate and 1.0 means always terminate.


# Quick setup

Run `make zip` to create a `chaos-lambda.zip` file containing the lambda
function.  Upload it to a S3 bucket in your account, taking note of the bucket
name (eg `my-bucket`) and the path (eg `lambdas/chaos-lambda.zip`).

Create the lambda function via CloudFormation using the
`cloudformation/templates/lambda.json` template, entering the bucket name and
path.

In the AWS console go to the `Lambda` service, select the newly created lambda
function, go into the `Event sources` tab, and click `Add event source`.
Choose `Scheduled Event` for the `Event source type`, then fill in the fields
as follows (enter the `Name` first as changing it resets the other fields):
* Name: `business-hours`
* Description: `Assume things fail`
* Schedule expression: `cron(0 10-16 ? * MON-FRI *)`

Ensure `Enable now` is selected and click the `Submit` button.  You can disable
or re-enable the hourly trigger at any time by clicking the
`Enabled`/`Disabled` link in the `State` column.

To receive notifications if the lambda function fails for any reason, create
another stack using the `cloudformation/templates/alarms.json` template.  This
takes the lambda function name (something similar to
`chaos-lambda-ChaosLambdaFunction-EM2XNWWNZTPW`) and the email address to
send the alerts to.


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
