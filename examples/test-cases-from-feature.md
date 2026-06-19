# Example: feature description -> test cases

This is a real-ish example of how this workflow goes. I've cleaned it up but the general shape - what AI generates, what I change - is accurate.

## Input: feature description

From the ticket (paraphrased):

> **Phone login with OTP**
> 
> User enters their phone number on the login screen. System sends a 6-digit OTP via SMS (Twilio). User enters OTP. On success, system returns a JWT session token (expires 24h) and a refresh token (expires 30d).
>
> OTP validity: 5 minutes. Single use - verified OTP is deleted immediately.
> Rate limiting: max 3 OTP send attempts per phone number per 10 minutes. After 3 failed verify attempts: lock account for 15 minutes.
>
> Endpoints:
> - `POST /auth/otp/send` - body: `{ phone: string }` - returns `{ message: "OTP sent" }` or error
> - `POST /auth/otp/verify` - body: `{ phone: string, otp: string }` - returns `{ access_token, refresh_token }` or error

## What AI generated (raw output)

After feeding the prompt from `prompts/test-design.md`:

```
TC-001: Successful OTP send to valid phone number
Preconditions: User exists with verified phone number in system
Steps:
  1. POST /auth/otp/send with body {"phone": "+12025551234"}
  2. Verify HTTP 200 response
  3. Check response body contains "OTP sent" message
Expected: 200 OK, message confirming OTP sent. OTP stored in Redis with 5-min TTL.

TC-002: Successful login - OTP verify
Preconditions: User exists, OTP sent (TC-001 complete), OTP not expired
Steps:
  1. POST /auth/otp/verify with {"phone": "+12025551234", "otp": "<valid OTP>"}
Expected: 200 OK, response contains access_token and refresh_token

TC-003: OTP send to unregistered phone number
Preconditions: Phone number not in user table
Steps:
  1. POST /auth/otp/send with {"phone": "+19999999999"}
Expected: [AI note: unclear - does the system reveal that the phone isn't registered, or return 200 to avoid enumeration? Please specify.]

TC-004: OTP send with invalid phone format
Preconditions: none
Steps:
  1. POST /auth/otp/send with {"phone": "not-a-phone"}
  2. Also test {"phone": ""}, {"phone": null}, missing field
Expected: 400 Bad Request, validation error

TC-005: OTP expired
Preconditions: OTP sent, 5 minutes elapsed (or TTL manually expired in test setup)
Steps:
  1. POST /auth/otp/verify with correct OTP after TTL has passed
Expected: 401, error indicating OTP expired

TC-006: Wrong OTP
Preconditions: OTP sent
Steps:
  1. POST /auth/otp/verify with {"phone": "+12025551234", "otp": "000000"}
Expected: 401, error indicating invalid OTP. Attempt counter incremented.

TC-007: OTP already used (replay)
Preconditions: OTP sent, successfully verified once (TC-002)
Steps:
  1. POST /auth/otp/verify with the same OTP again
Expected: 401, OTP invalid (already consumed)

TC-008: Account lockout after 3 failed verify attempts
Preconditions: OTP sent
Steps:
  1. POST /auth/otp/verify with wrong OTP x 3
  2. POST /auth/otp/verify with correct OTP
Expected: After 3rd failure, account locked. Correct OTP also rejected with lockout error. Lockout lasts 15 min.
[AI note: What HTTP status for lockout? 401, 423, or 429?]

TC-009: OTP send rate limit
Preconditions: User exists
Steps:
  1. POST /auth/otp/send x 3 within 10 minutes
  2. POST /auth/otp/send (4th attempt)
Expected: First 3 succeed (200). 4th attempt rejected.
[AI note: What status code for rate limit hit? 429 most likely. Also - do the first 3 each invalidate the previous OTP?]

TC-010: Twilio service unavailable
Preconditions: Twilio mocked to return error
Steps:
  1. POST /auth/otp/send with valid phone
Expected: [AI note: unclear what the API should return here. 502? 503? Or does it queue for retry? Depends on implementation.]

TC-011: Access token format and claims
Preconditions: Successful login (TC-002)
Steps:
  1. Decode access_token (JWT)
  2. Verify exp claim = ~24h from issue time
  3. Verify sub claim = user ID
  4. Verify token is signed correctly
Expected: Valid JWT, correct claims
[AI note: I don't know what claims your JWT includes beyond standard ones. Check with dev team.]
```

## What I changed after review

**TC-003 (unregistered phone):** AI correctly flagged this as ambiguous. Checked with dev - the API returns 200 to avoid phone enumeration. Updated expected result.

**TC-005 (OTP expired):** Added test setup note: in integration tests we use a Redis helper to manually expire the key, not `time.sleep(300)`. AI didn't know this, had to add it.

**TC-008 (lockout):** Status code is 429 in our implementation, not 423. Also added: OTP send also blocked during lockout (AI missed this - it's in the business rules but not in the endpoint description).

**TC-009 (send rate limit):** Confirmed 429. Added clarification: each new OTP invalidates the previous one - this is tested in a separate case AI didn't generate. I added TC-012 manually.

**TC-010 (Twilio down):** Our service returns 503 and queues for retry with a 30s delay. Added that and a note about mocking strategy.

**TC-011 (JWT claims):** Added actual claims our tokens include: `sub`, `exp`, `iat`, `phone_verified: true`, `device_id`. AI couldn't know these.

**Added manually:**
- TC-012: Second OTP send invalidates first OTP
- TC-013: Concurrent verify requests with same OTP (race condition - only one should succeed)
- TC-014: Phone number normalization ("+1 202 555 1234" vs "+12025551234")

## Takeaway

AI gave me 11 test cases in about 2 minutes. 7 of them were usable with minor edits. 3 had gaps flagged correctly (AI said "I'm not sure, check with team"). 1 missed a business rule that wasn't explicitly stated.

I added 3 cases it didn't think of - the race condition, OTP invalidation on resend, and phone normalization. Those came from my own domain knowledge.

Total time from ticket to draft test plan: ~20 minutes instead of an hour. The reviewing and filling in gaps is still manual, and that's where the actual thinking is.
