# Prompt: rough notes -> bug report

When I'm doing exploratory testing I keep raw notes - timestamps, what I clicked, what looked wrong, curl output, random observations. This prompt turns that into a structured bug report.

## The prompt

```
I'm a QA engineer and I found a bug during testing. I have rough notes and need help turning them into a proper bug report.

My raw notes:
[paste your notes here - messy is fine, include any logs, responses, screenshots descriptions]

Environment info (fill in what you know):
- Service/component: 
- Environment: [staging / prod / local]
- Build/version: 
- Date/time of observation:

Please write a bug report with:
- Title: short, specific, describes the problem not the symptom
- Severity: your suggestion with reasoning (I'll override if needed)
- Summary: 2-3 sentences, what's broken and why it matters
- Steps to reproduce: numbered, precise enough that someone else can follow them
- Expected result
- Actual result
- Additional context: logs, response bodies, anything relevant from my notes

Where my notes are ambiguous or something's missing from the steps, note it in brackets so I know to fill it in. Don't make up steps or details that aren't in my notes.
```

## Example input (raw notes)

```
testing order refund endpoint
2024-11-14 ~14:30
POST /orders/ORD-4821/refund
full refund, amount = original order total (450.00)

got 200 back, response says refund_status: "processing"
checked DB - refund record created, looks fine

but kafka event never showed up in refund-processor topic
waited ~5 min, nothing
checked consumer group - offset not moving

tried again with partial refund (200.00) - same thing, 200 ok, no kafka event
tried on different order (ORD-4822) - same

so looks like refund endpoint returns success but doesn't actually publish to kafka?
note: this worked last week, might be related to the kafka config change on tuesday
```

## Example output

**Title:** Refund endpoint returns 200 but does not publish event to kafka refund-processor topic

**Severity:** High - refunds appear successful to the API caller but are not processed downstream

**Summary:**
When a refund is submitted via `POST /orders/{id}/refund`, the endpoint returns 200 and creates a refund record in the database, but no event is published to the `refund-processor` Kafka topic. Refunds are silently stuck in "processing" state. The issue affects both full and partial refunds and was observed across multiple orders.

**Steps to reproduce:**
1. Send `POST /orders/ORD-4821/refund` with `amount: 450.00` (full refund)
2. Verify response: 200 OK, `refund_status: "processing"` [attach response body]
3. Check database - confirm refund record was created
4. Monitor `refund-processor` Kafka topic for ~5 minutes
5. Check consumer group offset - confirm it does not advance

**Expected result:**
After step 2, a refund event is published to `refund-processor`. Consumer group offset advances within expected lag time.

**Actual result:**
No event appears in `refund-processor`. Consumer group offset stays unchanged. Refund remains in "processing" indefinitely.

**Additional context:**
- Tested on orders ORD-4821 (full refund, 450.00) and ORD-4822 - same behavior
- Partial refund (200.00) on same order also affected - not specific to full refunds
- Behavior changed recently - reportedly worked last week
- Possible correlation with Kafka config change on [Tuesday, date TBD - confirm with infra team]
- [Attach: DB refund record, Kafka consumer group status output]

## Notes

The main thing this saves is the formatting time. The actual content - what happened, what the expected behavior was - that's still on you. AI can't tell from your notes whether "refund_status: processing" is the right field or not, or what the Kafka consumer lag threshold should be.

The "don't make up steps" instruction is important. Without it, Claude fills in gaps with plausible-sounding steps that may not be reproducible.
