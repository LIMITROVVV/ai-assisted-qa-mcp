# A day in the workflow

Not every day, but a representative one. I'm not going to pretend this is some perfect optimized system - it's a bunch of habits that stuck because they actually saved time.

## Morning: new feature in the sprint

Let's say there's a ticket for a new endpoint - something like a payment status webhook receiver. Before AI tooling, I'd read the spec, read the existing tests for similar endpoints, start writing test cases more or less from scratch.

Now:

1. Open Claude Code with Postman and filesystem MCPs connected
2. Ask Claude to read the OpenAPI spec file from the project directory (filesystem MCP - no copy-pasting)
3. "Here's the spec. Generate test case descriptions covering happy path, validation errors, and integration failures."
4. Get back a draft in ~2 minutes
5. Spend 15-20 minutes reviewing: fixing wrong assumptions, adding domain-specific cases, removing things that don't apply

The reviewing is not optional. It's the actual work. But it's faster to review than to write from scratch, especially for the boring structural stuff (missing field tests, wrong type tests).

## Mid-morning: test documentation in Allure

We write test cases in Allure before automating them. This used to be the most tedious part of my job - lots of copy-paste, formatting, adding the same metadata fields over and over.

With Allure MCP connected, I can tell Claude: "Here are 8 test cases I wrote. Format them for Allure with proper step descriptions, expected results, and add these labels: feature=payments, layer=api, severity based on your judgment."

Output isn't perfect - severity assessments are often wrong, and Claude doesn't know which labels we use internally. But the structure is right and the steps are formatted. I correct severity and add any missing labels in a few minutes.

## After standup: picking up an automation task

A test case that was manual needs to be automated. I have pytest and an existing test file for the service.

I ask Claude to read the existing test file (filesystem MCP) and the OpenAPI spec, then generate a test function following the same patterns as the existing code. 

The result: function structure is usually correct, fixtures are named right (because it read the existing file), happy path assertion is there. The edge case assertions are often wrong or incomplete. I spend 20-30 minutes fixing assertions and adding what it missed.

This used to take 45-60 minutes including the mental overhead of context-switching into "test code" mode. Now it's 20-30 minutes of focused review.

## Afternoon: reviewing a PR

Colleague opened a PR adding a new Kafka consumer. I'm reviewing for testability and checking if there's anything we should test that isn't covered.

I ask Claude (with GitHub MCP) to read the diff and list what behaviors are introduced that need testing. It gives a list. Some are obvious (it's going to list "the consumer processes messages" type things), but occasionally it catches something I'd have glossed over - like a dead letter queue path that's implemented but has no test.

I don't take the list at face value. I use it as a checklist to check against what's actually in the PR.

## Late afternoon: bug report from exploratory session

I ran 2 hours of exploratory testing on a new payment flow. I have notes in a scratch file - rough, out of order, includes dead ends and things that turned out to be fine.

Feed notes to the bug report prompt (`prompts/bug-report.md`). Get back a formatted bug report. Check it against my notes - sometimes Claude synthesizes steps in a slightly wrong order or softens a description in a way that makes the bug sound less severe. Fix those. Done in 5 minutes instead of 20.

## What I don't use AI for

- Deciding what's important to test. That requires knowing the product.
- Load testing strategy. AI has no idea what "acceptable" looks like for our system.
- Root cause analysis of test failures. It can help read logs, but guessing why something fails in our specific infrastructure is mostly noise.
- Contract testing setup. The tooling (Pact, etc.) has enough nuance that AI-generated config leads to false security.
- Anything where I need to be right. For exploratory testing notes or rough drafts, being 80% right is fine. For a migration test plan or a go/no-go test suite, I write it myself.

## Rough time savings estimate

Honest rough numbers from my own work, not a controlled study:

| Task | Before | With AI assist |
|------|--------|----------------|
| First draft test cases (medium feature) | 45-60 min | 20-25 min |
| Allure documentation for 10 test cases | 30 min | 10 min |
| Pytest scaffold for new endpoint | 40-50 min | 20-25 min |
| Formatting bug reports | 15-20 min per report | 5 min |

The savings are real but not dramatic. It's not "10x faster" - it's "not dreading the tedious stuff." Which actually matters for consistency.
