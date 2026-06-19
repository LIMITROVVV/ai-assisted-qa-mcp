# Prompt: feature description -> test case draft

Use this when you have a spec or ticket and need a first cut of test cases quickly. The output will need review - treat it as a starting point, not a finished doc.

## The prompt

```
I'm a QA engineer working on backend/integration testing. I need help drafting test cases for a new feature.

Feature description:
[paste description, ticket, or spec here]

Context (fill in what's relevant):
- API endpoint(s) involved: [e.g. POST /api/v1/auth/otp]
- Data store: [e.g. PostgreSQL, Redis for session]
- Related services: [e.g. SMS gateway, user service]
- Known constraints: [rate limits, expiry times, business rules]

Please generate test cases covering:
1. Happy path (the main success scenario)
2. Negative cases (invalid input, missing fields, wrong state)
3. Boundary conditions (limits, edge values)
4. If there are integrations - what happens when a downstream service fails

For each test case, give me:
- Test case ID (TC-001, TC-002, etc.)
- Short title
- Preconditions
- Steps
- Expected result

Don't generate test code yet, just the test case descriptions. Flag anything where you're not sure what the expected behavior should be - I'd rather know than have you guess.
```

## Example input

Feature description:
> User can log in using their phone number. They enter their phone, receive an OTP via SMS, enter the OTP, and get a session token. OTP is valid for 5 minutes and can be used once. After 3 failed attempts the account is locked for 15 minutes.

Context:
- API endpoints: `POST /auth/otp/send`, `POST /auth/otp/verify`
- SMS gateway: Twilio
- Session stored in Redis, OTP stored in Redis with TTL
- User table in PostgreSQL

## Example output (excerpt - full example in examples/test-cases-from-feature.md)

```
TC-001: Successful OTP login
Preconditions: User exists with verified phone number
Steps:
  1. POST /auth/otp/send with valid phone number
  2. Retrieve OTP from Redis (or mock SMS)
  3. POST /auth/otp/verify with correct OTP within 5 minutes
Expected: 200 OK, session token returned, OTP deleted from Redis

TC-005: OTP expired
Preconditions: User exists, OTP sent
Steps:
  1. POST /auth/otp/send
  2. Wait for TTL to expire (or manually expire the Redis key in test setup)
  3. POST /auth/otp/verify with correct but expired OTP
Expected: 401, error message indicating OTP expired (not "invalid OTP")

TC-008: Third failed attempt triggers lockout
Preconditions: User exists, OTP sent
Steps:
  1. POST /auth/otp/verify with wrong OTP (attempt 1) - expect 401
  2. POST /auth/otp/verify with wrong OTP (attempt 2) - expect 401
  3. POST /auth/otp/verify with wrong OTP (attempt 3) - expect 401 or 423
  4. POST /auth/otp/verify with correct OTP - expect 423 Locked
Expected: After attempt 3, account locked for 15 min. Correct OTP also rejected during lockout.

[AI note: Not sure what error code you use for lockout - 423, 429, or something custom? Marking this for review.]
```

## Notes on using this prompt

The "flag anything uncertain" instruction matters. Without it, Claude will happily invent expected behavior for things it doesn't know. You want it to surface gaps, not paper over them.

If you have an existing OpenAPI spec, paste it in instead of a text description - the output is usually more accurate because it has concrete field names and response schemas to work from.

After you get the output, go through each test case and ask: does this match how our system actually works? That's where the real work is.
