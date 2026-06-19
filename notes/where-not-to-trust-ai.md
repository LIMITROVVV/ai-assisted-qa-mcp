# Where not to trust AI in QA work

This is the part I think is missing from most "AI for QA" content. Everyone's excited to show the prompt that generates 20 test cases. Fewer people talk about the test cases that look right but are wrong in ways that matter.

Here's what I've actually run into.

## It invents API fields

This is the most common problem. When you give Claude an endpoint description - especially in prose rather than a schema - it will generate assertions for fields that don't exist in your actual response.

Example: I described an order endpoint that returns status and order_id. Claude generated tests asserting `response["created_at"]`, `response["updated_at"]`, and `response["items"]`. None of those were in the response. The tests looked reasonable. They would have failed silently if I hadn't run them.

When you give it an actual OpenAPI schema, this is better but not solved. If the schema has examples, it sometimes asserts specific example values.

**What to do:** run every generated test against the real API before treating it as done. Obvious, but easy to skip when the output looks confident.

## It doesn't know what "correct" means for your domain

AI can tell you a phone number field is required. It can't tell you that +7 numbers should behave differently from +1 numbers in your system, or that certain prefixes are blocked, or that the normalization step strips spaces but not dashes.

Same with business rules. If the spec says "users with status SUSPENDED can't place orders," Claude will generate a test for that. But if there's also "users with status PENDING_VERIFICATION can't place orders, unless the order is below $50," that's probably not in the spec anywhere - it's in someone's head or buried in a two-year-old Slack thread.

Those are the bugs that actually matter. AI systematically misses them because they don't come from the spec.

## Confidence calibration is off

Claude sounds equally certain whether it's right or wrong. This is dangerous in a few specific situations:

**Error codes.** When there's no spec for error responses, Claude picks ones that seem reasonable (400 for validation, 422 for business rules, 409 for conflicts). Sometimes those are right. Sometimes your service returns 200 with an error field in the body, or 500 for things that should be 400. Unchecked, these become wrong assertions that silently pass against the wrong behavior.

**Default values.** "The field defaults to X" - Claude will sometimes state this with confidence. It might be guessing from convention, not from your actual code. I've been burned by this twice: once on a timeout field that defaulted to 0 (not null), once on a boolean flag that defaulted to false instead of true.

**What changed in a PR.** When I ask Claude to list what a PR tests, it sometimes includes things that aren't actually in the diff, or misses a subtle behavior change because the code looks similar to what it expected.

## It misses race conditions and timing issues

These tend to be the interesting bugs in distributed systems - and AI almost never generates test cases for them without explicit prompting.

I had a Kafka consumer that was idempotent-by-design. Claude generated a test that sent a message and checked the result. Never occurred to it to send the same message twice, check for duplicate processing, or test what happens when the consumer restarts mid-batch.

Those aren't obscure cases. They're just harder to derive from a spec.

## It can't assess risk

"What should we test?" requires knowing what's likely to break and what would hurt if it did. AI doesn't have that context. It'll cover the whole spec and still miss the fact that this endpoint is called from three legacy services with weird timeout behavior, or that the DB migration running alongside this change makes certain state transitions impossible right now.

Risk assessment is the core of senior QA work. AI can help you be more thorough about the cases you've already identified, but it can't identify the cases that come from institutional knowledge.

## Summary

Use AI for:
- First draft when you know the domain and will review the output
- Formatting and documentation work
- Getting unstuck on test structure

Don't use AI to:
- Define what's important to test
- Assert anything you haven't verified against the real system
- Replace someone who knows the product

The failure mode isn't that AI gives you obviously wrong output. It's that it gives you plausible-looking output that's subtly wrong, and you don't catch it because it looked fine. The reviewing step isn't a formality.
