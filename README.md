# Vector

Vector is a tool that augments your auto-scaling groups. The two
features currently offered are Predictive Scaling and Flexible Down
Scaling.

## Predictive scaling

Auto Scaling groups do a good job of responding to current
load conditions, but if you have a predictable load pattern,
it can be nice to scale up your servers a little bit *early*.
Some reasons you might want to do that are:

 * If it takes several minutes for an instance to fully boot
   and ready itself for requests.
 * If you have very serious (but predictable) spikes,
   it's nice to have the capacity in place before the spike
   starts.
 * To give yourself a buffer of time if AWS APIs start
   throwing errors. If scaling up is going to fail, you'd
   rather it start failing with a little bit of time before
   you actually need the capacity so you can begin evasive maneuvers.

Vector examines your existing CloudWatch alarms tied to your Auto
Scaling groups, and predicts if they will be triggered in the future
based on what happened in the past.

**Note:** This only works with metrics that are averaged across your group -
like CPUUtilization or Load. If you auto-scale based on something
like QueueLength, Predictive Scaling will not work right for you.

For each lookback window you specify, Vector will first check the
current value of the metric * the number of nodes, and the past value of
the metric * the past number of nodes. If those numbers are close enough
(within the threshold specified by `--ps-valid-threshold`), then it will
continue.

Vector will then go back to the lookback window specified, and then
forward in time based on the lookahead window (`--ps-lookahead-window`).
It will compute the metric * number of nodes then to get a predicted
aggregate metric value for the current future. It then divides that by
the current number of nodes to get a predicted average value for the
metric. That is then compared against the alarm's threshold.

For example:

> You have an alarm that checks CPUUtilization of your group, and will
> trigger the alarm if that goes above 70%. Vector is configured to use a
> 1 week lookback window, a 1 hour lookahead window, and a valid-threshold
> of 0.8.
> 
> The current value of CPUUtilization is 49%, and there are 2 nodes in the
> group. CPUUtilization 1 week ago was 53%, and there were 2 nodes in the
> group. Therefore, total current CPUUtilization is 98%, and 1 week ago was
> 106%. Those are within 80% of each other (valid-threshold), so we can
> continue with the prediction.
> 
> The value of CPUUtilization 1 week ago, *plus* 1 hour was 45%, and
> there were 4 nodes in the group. We calculate total CPUUtilization for
> that time to be 180%. Assuming no new nodes are launched, the predicted
> average CPUUtilization for the group 1 hour from now is 180% / 2 = 90%.
> 90% is above the alarm's 75% threshold, so we trigger the scaleup
> policy.


If you use Predictive Scaling, you probably also want to use Flexible
Down Scaling (below) so that after scaling up in prediction of load,
your scaledown policy doesn't quickly undo Vector's hard work. You
probably want to set `up-to-down-cooldown` to be close to the size of
your `lookahead-window`.

### Timezones

If you specify a timezone (either explicitly or via the system
timezone), Vector will use DST-aware time calculations when evaluating
lookback windows. If you don't specify a timezone and your system time
is UTC, then 8AM on Monday morning after DST ends will look back 168
hours - which is 7AM on the previous Monday.  Predictive scaling would
be off by one hour for a whole week in that case.

## Flexible Down Scaling

### Different Cooldown Periods

Auto Scaling Groups support the concept of "cooldown periods" - a window
of time after a scaling activity where no other activities should take
place. This is to give the group a chance to settle into the new
configuration before deciding whether another action is required.

However, Auto Scaling Groups only support specifying the cooldown period
*after* a certain activity - you can say "After a scale up, wait 5
minutes before doing anything else, and after a scale down, wait 15
minutes." What you can't do is say "After a scale up, wait 5 minutes for
another scale up, and 40 minutes for a scale down."

Vector lets you add custom up-to-down and down-to-down cooldown periods.
You create your policies and alarms in your Auto Scaling Groups like
normal, and then *disable* the alarms tied to the scale down policy.
Then you tell Vector what cooldown periods to use, and he does the rest.

### Multiple Alarms

Another benefit to Flexible Down Scaling is the ability to specify
multiple alarms for a scaling down policy and require *all* alarms to
trigger before scaling down. With Vector, you can add multiple
(disabled) alarms to a policy, and Vector will trigger the policy only
when *both* alarms are in ALARM state. This lets you do something like
"only scale down when CPU utilization is < 30% and there is not a
backlog of requests on any instances".

### Max Sunk Cost

Vector also lets you specify a "max sunk cost" when scaling down a node.
Amazon bills on hourly increments, and you pay a full hour for every
partial hour used, so you want your instances to terminate as close to
their hourly billing renewal (without going past it).

For example, if you specify `--fds-max-sunk-cost 15m` and have two nodes
in your group - 47 minutes and 32 minutes away from their hourly billing
renewals - the group will not be scaled down.

(You should make sure to run Vector on an interval smaller than this
one, or else it's possible Vector may never find eligible nodes for
scaledown and never scaledown.)

### Variable Thresholds

When deciding to scale down, a static CPU utilization threshold may be
inefficient. For example, if there are 2 nodes running, and you have a
minimum of 2, and the average CPU is 75%, removing 1 node would
theoretically result in the remaining 2 nodes running at > 100%. However,
with 20 nodes running, at an average CPU of 75%, removing 1 node will
only result in an average CPU of 79% across the remaining 19 nodes.

When there are more nodes running, you can be more aggressive about
removing nodes without overloading the remaining nodes. Variable
thresholds allow you to express this.

You can enable variable thresholds with `--fds-variable-thresholds`.

### Integration with Predictive Scaling

Before scaling down, and if Predictive Scaling is in effect, Vector will
check to see if the size **after** scaling down would trigger Predictive
Scaling. If it would, the scaling policy will not be executed.

## Requirements

 * Auto Scaling groups must have the GroupInServiceInstances metric
   enabled.
 * Auto Scaling groups must have at least one scaling policy with a
   positive adjustment, and that policy must have at least one
   CloudWatch alarm with a CPUUtilization metric.

## Installation

```bash
$ gem install vector
```

## Usage

Typically vector will be invoked via cron periodically (every 10 minutes
is a good choice.)

```
Usage: vector [options]
DURATION can look like 60s, 1m, 5h, 7d, 1w
        --timezone TIMEZONE          Timezone to use for date calculations (like America/Denver) (default: system timezone)
        --region REGION              AWS region to operate in (default: us-east-1)
        --groups group1,group2       A list of Auto Scaling Groups to evaluate
        --fleet fleet                An AWS ASG Fleet (instead of specifying --groups)
    -v, --[no-]verbose               Run verbosely

Predictive Scaling Options
        --[no-]ps                    Enable Predictive Scaling
        --ps-lookback-windows DURATION,DURATION
                                     List of lookback windows
        --ps-lookahead-window DURATION
                                     Lookahead window
        --ps-valid-threshold FLOAT   A number from 0.0 - 1.0 specifying how closely previous load must match current load for Predictive Scaling to take effect
        --ps-valid-period DURATION   The period to use when doing the threshold check

Flexible Down Scaling Options
        --[no-]fds                   Enable Flexible Down Scaling
        --fds-up-to-down DURATION    The cooldown period between up and down scale events
        --fds-down-to-down DURATION  The cooldown period between down and down scale events
        --fds-max-sunk-cost DURATION Only let a scaledown occur if there is an instance this close to its hourly billing point
```

# Questions

### Why not just predictively scale based on the past DesiredInstances?

If we don't look at the actual utilization and just look
at how many instances we were running in the past, we will end up
scaling earlier and earlier, and will never re-adjust and not scale up
if load patterns change, and we don't need so much capacity.

### What about high availability? What if the box Vector is running on dies?

Luckily Vector is just providing optimizations - the critical component
of scaling up based on demand is still provided by the normal Auto
Scaling service. If Vector does not run, you just don't get the
predictive scaling and down scaling.

