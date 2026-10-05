# Cron in AWS: EventBridge Scheduler, not Rules

*Worked on: September 2026*

I needed a nightly job to fire at 2 a.m. Pacific: a Lambda that kicks off a data sync. I went looking for the familiar "EventBridge Rule with a cron expression" and couldn't find it where I expected.

## What tripped me up

In the current AWS console, the EventBridge **Rules** page is built around event-pattern triggers: "when something happens, run this." That's not what I wanted. I wanted "when the clock says so, run this," and that now lives in **EventBridge Scheduler**, a separate service with its own page. If you click through Rules looking for a schedule option, you can lose time wondering whether you're missing a permission.

## Why Scheduler is the better fit anyway

The real reason to use it is time zones. Cron expressions on the older scheduled-rule path are evaluated in UTC. Pacific time shifts between UTC-8 and UTC-7 with daylight saving, so a rule written as "10:00 UTC" runs at 2 a.m. in winter and 3 a.m. in summer. For a job that has to finish before staff start their day, an hour of drift twice a year is exactly the kind of bug that shows up months later.

Scheduler lets you attach an IANA time zone to the schedule itself, so the expression is written in local time and AWS handles the daylight saving change:

```python
import boto3

scheduler = boto3.client("scheduler")

scheduler.create_schedule(
    Name="nightly-sync",
    ScheduleExpression="cron(0 2 * * ? *)",   # 2:00 AM, local to the zone below
    ScheduleExpressionTimezone="America/Los_Angeles",
    FlexibleTimeWindow={"Mode": "OFF"},
    Target={
        "Arn": "arn:aws:lambda:REGION:ACCOUNT_ID:function:nightly-sync",
        "RoleArn": "arn:aws:iam::ACCOUNT_ID:role/scheduler-invoke-role",
    },
)
```

Use the zone name (`America/Los_Angeles`), not an abbreviation like PST, which would pin you to one side of the daylight saving change.

## Details worth remembering

- **Cron syntax has six fields**, and one of day-of-month or day-of-week must be `?`. That trips people up if they paste a five-field Unix cron line.
- **Scheduler needs its own execution role** that is allowed to invoke the target. This is separate from the Lambda's own role.
- **Turn the flexible time window off** if you want it to fire at the stated minute rather than somewhere inside a window.
- **Check the next run times.** The console previews upcoming trigger times in the zone you chose. I compared them against a couple of dates on either side of the daylight saving change before trusting it.

## Takeaway

For anything time-based in AWS, start with EventBridge Scheduler, and set the time zone explicitly instead of doing UTC arithmetic in your head. Rules are for reacting to events; Scheduler is for the clock.
