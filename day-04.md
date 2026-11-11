# Day 4: CloudWatch Alarm + SNS Email

**Date:** 11 Nov 2026 | **Region:** Mumbai (ap-south-1) | **Status:** Completed

## Goal

Get an email automatically when an EC2 instance's CPU goes high: build a CloudWatch alarm, send it to an SNS topic, trigger it on purpose, and confirm the email arrives.

## The idea in simple words

| Piece | Think of it as | Job |
|---|---|---|
| CloudWatch | Smoke detector | Watches a number (CPU) and compares it with a limit |
| Alarm | The rule | "If average CPU is above 70% for 5 minutes, sound the alarm" |
| SNS topic | A mailing list | Receives the alarm message and delivers it to every subscriber |
| SNS subscription | One address on the list | My email, confirmed through a link |

**Why the alarm sends to a topic and not straight to email:** the alarm only has to know one thing, the topic. Later I can add more emails, a phone number, or a chat channel to the topic without touching the alarm.

## Flow

```
EC2 instance --CPU data every 5 min--> CloudWatch
                                            |
                         average CPU > 70%  |  (alarm state: ALARM)
                                            v
                                      SNS topic: cpu-alerts
                                            |
                                            v
                                       Email to me
```

## What I built

| Component | Setting |
|---|---|
| EC2 instance | `84DayChallenge`, t3.small (2 vCPUs), Ubuntu |
| SNS topic | `cpu-alerts`, type Standard |
| Subscription | Email, status **Confirmed** (confirmation link clicked) |
| Alarm | `ec2-high-cpu` |
| Metric | `AWS/EC2` > `CPUUtilization`, per instance |
| Statistic / Period | Average, 5 minutes (300 s) |
| Condition | Greater than 70 |
| Datapoints to alarm | 1 out of 1 |
| Alarm action | State **In alarm** sends to `cpu-alerts` |

## How I triggered it

On the instance:

```bash
sudo apt update
sudo apt install -y stress-ng
nproc                                   # 2 CPUs
stress-ng --cpu 2 --timeout 600s &      # one worker per CPU, stops after 10 min
top                                     # both stress-ng-cpu workers at ~100%
```

I used `--cpu 2` because the average only passes 70% if **all** CPUs are busy. Loading one of two CPUs gives about 50%.

## Result

- The alarm moved **OK to ALARM**.
- The 5-minute window starting 06:26 UTC averaged **97.5%** CPU, above the 70% threshold.
- The email arrived about 30 seconds after that window closed (06:31 UTC).
- The email shows the state change, the reason, the threshold, the period, and the monitored metric.

Alarms are never instant: CloudWatch only receives one CPU value every 5 minutes (basic monitoring), so the alarm reacts after a window closes.

## Troubleshooting drill: alarm stuck in INSUFFICIENT_DATA

Right after I created the alarm, it showed `INSUFFICIENT_DATA`. It turned `OK` by itself after about 10 minutes, because CloudWatch needs at least one full period of data first.

If it stays stuck, check:

| Check | Should be |
|---|---|
| Namespace | `AWS/EC2` |
| Metric name | `CPUUtilization` |
| Dimension | `InstanceId` of the right instance |
| Period | 5 minutes |
| Instance state | Running (a stopped instance sends no data) |

## Observation: no email when the alarm recovered

After the load ended and I stopped the instance, I got no email for the OK state. This is expected:

- I only attached the SNS action to the **In alarm** state, so recovery sends nothing.
- A stopped instance sends no data, so the alarm goes to `INSUFFICIENT_DATA` and not to `OK`.

To also get a recovery email, edit the alarm and add a notification for the **OK** state using the same topic. The alarm's History tab lists every state change with a time.

## Key takeaways

- An alarm watches one metric, compared with one threshold, over one period.
- Each alarm state (OK, ALARM, INSUFFICIENT_DATA) can have its own action. Unset states send nothing.
- Confirm the SNS email subscription first, or no alarm email will arrive.
- INSUFFICIENT_DATA right after creation is normal. If it persists, check the namespace, metric, dimension and period.
- A load test must keep every CPU busy to push the average over the threshold.

## Cleanup

- `stress-ng` finished by itself after 10 minutes.
- Stopped `84DayChallenge` (stopped, not terminated).
- The SNS topic and alarm cost almost nothing, so I kept them.

## Still to do

- Add a second, smaller budget alarm (about Rs 100) on top of the Rs 500 budget from Day 1.
- Add an OK-state notification to the alarm.
