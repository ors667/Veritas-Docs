# Runbook — Dead-Letter Queue Triage

**Trigger:** either dead-letter-queue depth alarm has fired.
**Severity:** highest. A message in a dead-letter queue means a trade was not
processed, which is a potential reporting gap, not merely an operational one.

## The rule before every other rule

Nothing is deleted from a dead-letter queue until the reason it failed is
understood and the message has been reprocessed. Every trade has to be accounted
for. A dead-letter queue that has been "cleaned up" is worse than a full one,
because the gap is now invisible.

## Which queue fired

There are two, and they mean different things.

**Trade ingestion dead-letter queue.** The event never reached the ledger. The
trade is not recorded anywhere downstream. Common causes, in the order they
usually turn out to be true: schema validation failure (a field is missing or
malformed), counterparty identifier lookup failure, ledger write failure.

**Submission dead-letter queue.** The trade is recorded but the regulatory
submission failed five times. This one is on a clock — the submission still has
to land before the next-day deadline, and five failures means something
systematic rather than transient.

## Triage

1. Read the message bodies. They carry both the trade data and the error.
2. Group by error. A single malformed trade and a systematic schema mismatch look
   identical at depth one and completely different at depth two hundred.
3. Fix the underlying cause before redriving. Redriving into an unfixed cause
   just refills the queue and resets everyone's sense of when it started.

## Escalation

The submission queue's alarm routes to Regulatory Affairs as well as on-call, and
that is deliberate: the deadline is theirs to manage, not the on-call engineer's.
Bring them in when you see the alarm, not when you have finished triaging.

The ingestion alarm is never silenced during trading hours. If it is noisy, the
noise is the finding.
