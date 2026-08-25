# Runbook — End-of-Day Batch

**Trigger:** the `veritas-eod-report` CronJob did not complete, or did not start.
**Severity:** high during a trading week; the batch feeds a T+1 obligation.

## What the batch does

At 18:00 UTC on each trading day the batch produces the consolidated end-of-day
report covering any trade that did not flow through the real-time path. It has a
two-hour deadline and only one instance may run at a time.

## First checks

Look at the CronJob's own history before anything else:

```
kubectl describe cronjob veritas-eod-report -n veritas
kubectl get jobs -n veritas --sort-by=.metadata.creationTimestamp
```

Three outcomes, three different responses:

- **No job was created.** The CronJob was suspended, or the scheduler did not
  fire. Check `.spec.suspend` first — a suspended CronJob is silent, and it is
  the most common cause of a batch that "did not run".
- **A job was created and is still running.** Check how long. Past two hours it
  will be killed by its deadline; decide whether to let it die and re-trigger, or
  to let it finish if it is close.
- **A job was created and failed.** Read its pod logs. The batch is read-heavy on
  the Aurora read replica, so replica load is the usual cause.

## Re-triggering

```
kubectl create job --from=cronjob/veritas-eod-report manual-eod-$(date +%Y%m%d) -n veritas
```

Only one instance may run at a time. Confirm nothing else is running before you
create the manual job, or the concurrency policy will reject it and you will
conclude, wrongly, that the trigger did not work.

## When to escalate

Escalate to Regulatory Affairs as soon as it is clear the batch will not complete
before the T+1 deadline — not once the deadline has passed. They need the warning
to prepare a late-reporting notification, and that preparation takes longer than
the batch does.
