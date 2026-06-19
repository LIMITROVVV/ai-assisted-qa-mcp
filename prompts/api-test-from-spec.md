# Prompt: OpenAPI spec -> pytest or Postman test scaffold

This is for getting a first draft of automated test code. The generated code will have the right structure and cover obvious cases, but you'll need to fix assertions, add auth setup, and handle anything domain-specific.

## Prompt for pytest

```
I need a pytest test file for the following API endpoint. Generate a scaffold - I'll fill in the details.

Endpoint spec:
[paste OpenAPI YAML/JSON for the endpoint, or a text description with request/response schema]

Tech context:
- HTTP client: [httpx / requests / aiohttp]
- Auth method: [Bearer token / API key / Basic / none]
- Base URL is in an env var: [e.g. BASE_URL]
- Fixtures I already have: [e.g. auth_token, test_user, db_session - describe them briefly]

Please generate:
1. A pytest file with test functions for:
   - Main success case
   - Missing required fields (one test per required field if there are multiple)
   - Invalid field values (pick the most obviously testable ones)
   - Unauthorized access (if auth is required)
   - Any status-specific behavior mentioned in the spec

2. Use parametrize for cases that vary only by input data
3. Add TODO comments where I need to fill in assertions or setup
4. Don't generate fixture definitions - I'll wire in my existing ones

Be conservative with assertions: only assert what the spec explicitly states. Don't assert response fields that aren't in the spec.
```

## Prompt for Postman collection

```
I need Postman test scripts for the following API endpoint.

Endpoint spec:
[paste spec or description]

Generate:
1. A request definition (method, URL with path variables, headers, example body)
2. Pre-request script if needed for auth or setup
3. Test script (pm.test blocks) covering:
   - Status code check for success case
   - Response body schema validation
   - Key field checks
   - Error status codes from the spec

Use {{base_url}} as the URL variable. Assume there's a {{auth_token}} collection variable for auth.

Add comments for anything that needs manual setup (e.g. "replace with valid order ID from your test data").
```

## Example input (pytest prompt)

Endpoint spec:
```yaml
POST /api/v1/payments
Summary: Create a payment
Auth: Bearer token required

Request body (application/json):
  amount: number, required, > 0
  currency: string, required, enum: [USD, EUR, GBP]
  order_id: string, required
  idempotency_key: string, optional, max 64 chars

Responses:
  201: Payment created
    body: { payment_id: string, status: "pending", created_at: datetime }
  400: Validation error
    body: { error: string, field: string }
  401: Unauthorized
  409: Duplicate idempotency_key
  422: amount must be > 0
```

Fixtures I have: `auth_headers` (dict with Authorization header), `test_order_id` (str)

## Example output (excerpt)

```python
import pytest
import httpx
import os
from uuid import uuid4

BASE_URL = os.environ["BASE_URL"]
PAYMENTS_URL = f"{BASE_URL}/api/v1/payments"


def make_payload(**overrides):
    """Base valid payload. Override fields to test variations."""
    return {
        "amount": 100.00,
        "currency": "USD",
        "order_id": "TODO: inject from test_order_id fixture",
        "idempotency_key": str(uuid4()),
        **overrides,
    }


class TestCreatePaymentSuccess:
    def test_creates_payment_returns_201(self, auth_headers):
        payload = make_payload()
        response = httpx.post(PAYMENTS_URL, json=payload, headers=auth_headers)

        assert response.status_code == 201
        data = response.json()
        assert "payment_id" in data
        assert data["status"] == "pending"
        assert "created_at" in data
        # TODO: validate created_at format if needed


class TestCreatePaymentValidation:
    @pytest.mark.parametrize("missing_field", ["amount", "currency", "order_id"])
    def test_missing_required_field_returns_400(self, auth_headers, missing_field):
        payload = make_payload()
        del payload[missing_field]
        response = httpx.post(PAYMENTS_URL, json=payload, headers=auth_headers)

        assert response.status_code == 400
        data = response.json()
        assert "error" in data
        assert data.get("field") == missing_field  # TODO: confirm your API returns this

    def test_amount_zero_returns_422(self, auth_headers):
        payload = make_payload(amount=0)
        response = httpx.post(PAYMENTS_URL, json=payload, headers=auth_headers)
        assert response.status_code == 422
```

## What to expect from the output

The structure is usually solid - class grouping, parametrize for similar cases, TODO comments in the right places. The assertions are the weak point. Claude doesn't know how your error responses are actually formatted, whether your idempotency handling returns 409 or replays the 201, or what "created_at" format you use. Check every assertion against actual API responses before treating this as done.

The `make_payload` helper pattern saves time when you're generating lots of variation tests - worth keeping even if you change the assertions.
