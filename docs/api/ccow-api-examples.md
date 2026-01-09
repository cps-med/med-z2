# med-z8 CCOW API Examples

This document provides concrete request/response examples for the med-z8 CCOW service API, including real JWT tokens (sanitized), HTTP headers, and error scenarios.

**Purpose:** Help developers understand the exact data flow, token structure, and API contracts when integrating with med-z8.

---

## Table of Contents

1. [Authentication Overview](#authentication-overview)
2. [JWT Token Structure](#jwt-token-structure)
3. [Common HTTP Headers](#common-http-headers)
4. [API Endpoint Examples](#api-endpoint-examples)
   - [Set Active Patient](#set-active-patient)
   - [Get Active Patient](#get-active-patient)
   - [Clear Active Patient](#clear-active-patient)
   - [Get Context History](#get-context-history)
   - [Get All Active Patients (Admin)](#get-all-active-patients-admin)
   - [Cleanup Stale Contexts (Admin)](#cleanup-stale-contexts-admin)
   - [Health Check](#health-check)
5. [Error Scenarios](#error-scenarios)
6. [Client Integration Examples](#client-integration-examples)

---

## Authentication Overview

**All `/ccow/*` endpoints require JWT authentication.**

**Authentication Flow:**
1. User authenticates with Keycloak (outside med-z8 scope)
2. Client application receives JWT access token
3. Client includes token in `Authorization` header: `Bearer <token>`
4. med-z8 validates JWT using Keycloak's public keys (JWKS)
5. med-z8 extracts `user_id` from JWT `sub` claim
6. Request proceeds with authenticated user context

**Token Lifespan:**
- Access tokens typically expire after 5-15 minutes
- Clients must implement token refresh flow
- Expired tokens return `401 Unauthorized`

---

## JWT Token Structure

**Example JWT Access Token from Keycloak:**

```
Header:
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "FJ86GcF3jTbNLOco4NvZkUCIUmfYCqoqtOQeMfbhNlE"
}

Payload:
{
  "exp": 1736356800,
  "iat": 1736352800,
  "jti": "e6f7a8b9-c0d1-2e3f-4a5b-6c7d8e9f0a1b",
  "iss": "http://localhost:8080/realms/va-healthcare",
  "aud": "med-z8",
  "sub": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "typ": "Bearer",
  "azp": "med-z1",
  "session_state": "12345678-1234-1234-1234-123456789012",
  "acr": "1",
  "scope": "openid email profile",
  "sid": "12345678-1234-1234-1234-123456789012",
  "email_verified": true,
  "name": "John Doe",
  "preferred_username": "john.doe",
  "given_name": "John",
  "family_name": "Doe",
  "email": "john.doe@va.gov"
}

Signature: <cryptographic signature>
```

**Key Claims Used by med-z8:**
- **`sub`**: User ID (UUID) - PRIMARY KEY for context storage
- **`email`**: User email - stored in context for display
- **`name`**: Full name - used in audit trail
- **`iss`**: Issuer - validated against `OIDC_ISSUER` config
- **`aud`**: Audience - must be "med-z8"
- **`exp`**: Expiration - must be in future

**Full JWT Token (Base64-encoded, for testing):**

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV
tZllDcW9xdE9RZWZ2aE5sRSJ9.eyJleHAiOjE3MzYzNTY4MDAsImlhdCI6MTczNjM1MjgwMCwianRp
IjoiZTZmN2E4YjktYzBkMS0yZTNmLTRhNWItNmM3ZDhlOWYwYTFiIiwiaXNzIjoiaHR0cDovL2xvY2
FsaG9zdDo4MDgwL3JlYWxtcy92YS1oZWFsdGhjYXJlIiwiYXVkIjoibWVkLXo4Iiwic3ViIjoiYTFi
MmMzZDQtZTVmNi03ODkwLWFiY2QtZWYxMjM0NTY3ODkwIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoibW
VkLXoxIiwic2Vzc2lvbl9zdGF0ZSI6IjEyMzQ1Njc4LTEyMzQtMTIzNC0xMjM0LTEyMzQ1Njc4OTAx
MiIsImFjciI6IjEiLCJzY29wZSI6Im9wZW5pZCBlbWFpbCBwcm9maWxlIiwic2lkIjoiMTIzNDU2Nz
gtMTIzNC0xMjM0LTEyMzQtMTIzNDU2Nzg5MDEyIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsIm5hbWUi
OiJKb2huIERvZSIsInByZWZlcnJlZF91c2VybmFtZSI6ImpvaG4uZG9lIiwiZ2l2ZW5fbmFtZSI6Ik
pvaG4iLCJmYW1pbHlfbmFtZSI6IkRvZSIsImVtYWlsIjoiam9obi5kb2VAdmEuZ292In0.SIGNATURE
```

**Note:** The signature portion is not shown in full as it's cryptographic binary data. Real tokens are ~1200-1500 characters.

---

## Common HTTP Headers

**Required Headers:**

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
Content-Type: application/json
```

**Optional Headers:**

```http
User-Agent: med-z1/1.0.0
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
```

---

## API Endpoint Examples

### Set Active Patient

**Purpose:** Set the active patient for the authenticated user.

**Endpoint:** `PUT /ccow/active-patient`

**Request:**

```http
PUT /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
Content-Type: application/json

{
  "patient_id": "1012853550V207686",
  "set_by": "med-z1"
}
```

**Request Body Schema:**

```json
{
  "patient_id": "string (required) - ICN, DFN, or application-specific patient ID",
  "set_by": "string (required) - Application name that initiated context change"
}
```

**Response (200 OK):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "email": "john.doe@va.gov",
  "patient_id": "1012853550V207686",
  "set_by": "med-z1",
  "set_at": "2026-01-08T18:30:45.123456",
  "last_accessed_at": "2026-01-08T18:30:45.123456"
}
```

**Response Schema:**

```json
{
  "user_id": "string - UUID from JWT sub claim",
  "email": "string - User email from JWT",
  "patient_id": "string - Active patient ID",
  "set_by": "string - Application that set context",
  "set_at": "string (ISO 8601) - When context was set",
  "last_accessed_at": "string (ISO 8601) - When context was last accessed"
}
```

**cURL Example:**

```bash
curl -X PUT http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV..." \
  -H "Content-Type: application/json" \
  -d '{
    "patient_id": "1012853550V207686",
    "set_by": "med-z1"
  }'
```

**Python Example:**

```python
import httpx

async def set_active_patient(access_token: str, patient_id: str):
    async with httpx.AsyncClient() as client:
        response = await client.put(
            "http://localhost:8001/ccow/active-patient",
            json={"patient_id": patient_id, "set_by": "med-z1"},
            headers={"Authorization": f"Bearer {access_token}"}
        )
        response.raise_for_status()
        return response.json()

# Usage
context = await set_active_patient(
    access_token="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...",
    patient_id="1012853550V207686"
)
print(f"Active patient: {context['patient_id']}")
```

**JavaScript Example:**

```javascript
async function setActivePatient(accessToken, patientId) {
  const response = await fetch('http://localhost:8001/ccow/active-patient', {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      patient_id: patientId,
      set_by: 'med-z1'
    })
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  return await response.json();
}

// Usage
const context = await setActivePatient(
  'eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...',
  '1012853550V207686'
);
console.log(`Active patient: ${context.patient_id}`);
```

---

### Get Active Patient

**Purpose:** Retrieve the current active patient for the authenticated user.

**Endpoint:** `GET /ccow/active-patient`

**Request:**

```http
GET /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Response (200 OK):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "email": "john.doe@va.gov",
  "patient_id": "1012853550V207686",
  "set_by": "med-z1",
  "set_at": "2026-01-08T18:30:45.123456",
  "last_accessed_at": "2026-01-08T18:35:12.789012"
}
```

**Response (404 Not Found) - No Active Patient:**

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "detail": "No active patient context"
}
```

**cURL Example:**

```bash
curl -X GET http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV..."
```

**Python Example:**

```python
import httpx

async def get_active_patient(access_token: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "http://localhost:8001/ccow/active-patient",
            headers={"Authorization": f"Bearer {access_token}"}
        )

        if response.status_code == 404:
            return None  # No active patient

        response.raise_for_status()
        return response.json()

# Usage
context = await get_active_patient("eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...")

if context:
    print(f"Active patient: {context['patient_id']}")
else:
    print("No active patient selected")
```

---

### Clear Active Patient

**Purpose:** Clear the active patient context for the authenticated user.

**Endpoint:** `DELETE /ccow/active-patient`

**Request:**

```http
DELETE /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Response (200 OK):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "Patient context cleared",
  "user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

**Response (404 Not Found) - No Active Patient:**

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "detail": "No active patient context"
}
```

**cURL Example:**

```bash
curl -X DELETE http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV..."
```

**Python Example:**

```python
async def clear_active_patient(access_token: str):
    async with httpx.AsyncClient() as client:
        response = await client.delete(
            "http://localhost:8001/ccow/active-patient",
            headers={"Authorization": f"Bearer {access_token}"}
        )

        if response.status_code == 404:
            return None  # No context to clear

        response.raise_for_status()
        return response.json()

# Usage
result = await clear_active_patient("eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...")
print(result["message"])
```

---

### Get Context History

**Purpose:** Retrieve context change history. Admin users can see all events, regular users see their own events.

**Endpoint:** `GET /ccow/history`

**Query Parameters:**
- `user_id` (optional): Filter history for specific user (admin only)
- `limit` (optional): Limit number of entries (default: 100)

**Request (All Events):**

```http
GET /ccow/history HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Request (User-Specific):**

```http
GET /ccow/history?user_id=a1b2c3d4-e5f6-7890-abcd-ef1234567890 HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Response (200 OK):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "total": 5,
  "entries": [
    {
      "action": "set",
      "user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "email": "john.doe@va.gov",
      "patient_id": "1012853550V207686",
      "actor": "med-z1",
      "timestamp": "2026-01-08T18:30:45.123456"
    },
    {
      "action": "access",
      "user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "email": "john.doe@va.gov",
      "patient_id": "1012853550V207686",
      "actor": "med-z2",
      "timestamp": "2026-01-08T18:35:12.789012"
    },
    {
      "action": "clear",
      "user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "email": "john.doe@va.gov",
      "patient_id": null,
      "actor": "med-z1",
      "timestamp": "2026-01-08T18:40:00.000000"
    },
    {
      "action": "set",
      "user_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "email": "jane.smith@va.gov",
      "patient_id": "5000000341V359724",
      "actor": "med-z1",
      "timestamp": "2026-01-08T18:28:30.456789"
    },
    {
      "action": "access",
      "user_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "email": "jane.smith@va.gov",
      "patient_id": "5000000341V359724",
      "actor": "imaging-viewer",
      "timestamp": "2026-01-08T18:32:15.987654"
    }
  ]
}
```

**Action Types:**
- `"set"`: User set a new active patient
- `"access"`: User accessed existing active patient
- `"clear"`: User cleared active patient

**cURL Example:**

```bash
# All events
curl -X GET http://localhost:8001/ccow/history \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV..."

# User-specific
curl -X GET "http://localhost:8001/ccow/history?user_id=a1b2c3d4-e5f6-7890-abcd-ef1234567890" \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV..."
```

---

### Get All Active Patients (Admin)

**Purpose:** Retrieve all active patient contexts across all users (admin endpoint).

**Endpoint:** `GET /ccow/active-patients`

**Request:**

```http
GET /ccow/active-patients HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Response (200 OK):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "total": 2,
  "contexts": [
    {
      "user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "email": "john.doe@va.gov",
      "patient_id": "1012853550V207686",
      "set_by": "med-z1",
      "set_at": "2026-01-08T18:30:45.123456",
      "last_accessed_at": "2026-01-08T18:35:12.789012"
    },
    {
      "user_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "email": "jane.smith@va.gov",
      "patient_id": "5000000341V359724",
      "set_by": "med-z1",
      "set_at": "2026-01-08T18:28:30.456789",
      "last_accessed_at": "2026-01-08T18:32:15.987654"
    }
  ]
}
```

**cURL Example:**

```bash
curl -X GET http://localhost:8001/ccow/active-patients \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV..."
```

**Use Cases:**
- Admin dashboard showing all active patient contexts
- Monitoring/observability
- Debugging context synchronization issues

---

### Cleanup Stale Contexts (Admin)

**Purpose:** Remove context entries not accessed in last 24 hours (admin endpoint).

**Endpoint:** `POST /ccow/cleanup`

**Request:**

```http
POST /ccow/cleanup HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
Content-Type: application/json

{
  "threshold_hours": 24
}
```

**Request Body (Optional):**

```json
{
  "threshold_hours": "number (default: 24) - Remove contexts older than this many hours"
}
```

**Response (200 OK):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "Cleanup completed",
  "removed_count": 5,
  "threshold_hours": 24
}
```

**cURL Example:**

```bash
curl -X POST http://localhost:8001/ccow/cleanup \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV..." \
  -H "Content-Type: application/json" \
  -d '{"threshold_hours": 24}'
```

---

### Health Check

**Purpose:** Verify service health and dependency status (no authentication required).

**Endpoint:** `GET /health`

**Request:**

```http
GET /health HTTP/1.1
Host: localhost:8001
```

**Response (200 OK - Healthy):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "healthy",
  "timestamp": "2026-01-08T18:45:00.123456",
  "version": "1.0.0",
  "checks": {
    "keycloak_jwks": "healthy",
    "storage": "healthy"
  }
}
```

**Response (503 Service Unavailable - Degraded):**

```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json

{
  "status": "degraded",
  "timestamp": "2026-01-08T18:45:00.123456",
  "version": "1.0.0",
  "checks": {
    "keycloak_jwks": "unhealthy: Connection timeout",
    "storage": "healthy"
  }
}
```

**cURL Example:**

```bash
curl -X GET http://localhost:8001/health
```

**Use Cases:**
- Kubernetes liveness/readiness probes
- Monitoring and alerting
- Manual health verification

---

## Error Scenarios

### 401 Unauthorized - Missing Token

**Request:**

```http
GET /ccow/active-patient HTTP/1.1
Host: localhost:8001
```

**Response:**

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "detail": "Not authenticated"
}
```

---

### 401 Unauthorized - Invalid Token

**Request:**

```http
GET /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer invalid-token-12345
```

**Response:**

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "detail": "Invalid authentication credentials: Invalid token signature"
}
```

---

### 401 Unauthorized - Expired Token

**Request:**

```http
GET /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Response:**

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "detail": "Invalid authentication credentials: Token expired"
}
```

**Client Action:**
- Refresh access token using refresh token
- Retry request with new access token

---

### 401 Unauthorized - Wrong Audience

**Token with `aud: "med-z1"` instead of `aud: "med-z8"`:**

**Request:**

```http
GET /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Response:**

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "detail": "Invalid authentication credentials: Invalid token audience"
}
```

**Fix:**
- Ensure Keycloak client scope includes "med-z8" audience
- Request token with correct audience claim

---

### 404 Not Found - No Active Patient

**Request:**

```http
GET /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Response:**

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "detail": "No active patient context"
}
```

**Client Action:**
- This is normal if user hasn't selected a patient yet
- Prompt user to select a patient
- Call `PUT /ccow/active-patient` to set context

---

### 422 Unprocessable Entity - Invalid Request Body

**Request:**

```http
PUT /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
Content-Type: application/json

{
  "patient_id": "",
  "set_by": null
}
```

**Response:**

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "detail": [
    {
      "loc": ["body", "patient_id"],
      "msg": "patient_id cannot be empty",
      "type": "value_error"
    },
    {
      "loc": ["body", "set_by"],
      "msg": "none is not an allowed value",
      "type": "type_error.none.not_allowed"
    }
  ]
}
```

**Fix:**
- Provide valid `patient_id` (non-empty string)
- Provide valid `set_by` (non-null string)

---

### 500 Internal Server Error - Storage Backend Failure

**Request:**

```http
GET /ccow/active-patient HTTP/1.1
Host: localhost:8001
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkZKODZHY0YzalRiTkxPY280TnZaa1VDSVV...
```

**Response:**

```http
HTTP/1.1 500 Internal Server Error
Content-Type: application/json

{
  "detail": "Internal server error"
}
```

**Possible Causes:**
- Redis connection failure (if using Redis backend)
- Database connection failure (if using PostgreSQL backend)
- Unexpected exception in storage layer

**Client Action:**
- Retry with exponential backoff
- Check service health: `GET /health`
- Alert operations team if persistent

---

## Client Integration Examples

### Python FastAPI Client (med-z1)

```python
# app/utils/ccow_client.py
import httpx
from fastapi import HTTPException

class CCOWClient:
    def __init__(self, base_url: str = "http://localhost:8001"):
        self.base_url = base_url

    async def set_active_patient(
        self,
        access_token: str,
        patient_id: str,
        set_by: str = "med-z1"
    ):
        """Set active patient context."""
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                response = await client.put(
                    f"{self.base_url}/ccow/active-patient",
                    json={"patient_id": patient_id, "set_by": set_by},
                    headers={"Authorization": f"Bearer {access_token}"}
                )
                response.raise_for_status()
                return response.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 401:
                raise HTTPException(status_code=401, detail="CCOW authentication failed")
            elif e.response.status_code == 422:
                raise HTTPException(status_code=400, detail="Invalid patient_id")
            else:
                raise HTTPException(status_code=500, detail="CCOW service error")
        except httpx.RequestError as e:
            raise HTTPException(status_code=503, detail="CCOW service unavailable")

    async def get_active_patient(self, access_token: str):
        """Get current active patient context."""
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                response = await client.get(
                    f"{self.base_url}/ccow/active-patient",
                    headers={"Authorization": f"Bearer {access_token}"}
                )

                if response.status_code == 404:
                    return None  # No active patient

                response.raise_for_status()
                return response.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 401:
                raise HTTPException(status_code=401, detail="CCOW authentication failed")
            else:
                raise HTTPException(status_code=500, detail="CCOW service error")
        except httpx.RequestError as e:
            raise HTTPException(status_code=503, detail="CCOW service unavailable")

    async def clear_active_patient(self, access_token: str):
        """Clear active patient context."""
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                response = await client.delete(
                    f"{self.base_url}/ccow/active-patient",
                    headers={"Authorization": f"Bearer {access_token}"}
                )

                if response.status_code == 404:
                    return None  # No context to clear

                response.raise_for_status()
                return response.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 401:
                raise HTTPException(status_code=401, detail="CCOW authentication failed")
            else:
                raise HTTPException(status_code=500, detail="CCOW service error")
        except httpx.RequestError as e:
            raise HTTPException(status_code=503, detail="CCOW service unavailable")

# Usage in med-z1 routes
from app.utils.ccow_client import CCOWClient

ccow_client = CCOWClient(base_url="http://med-z8:8001")

@app.get("/patient/{icn}")
async def view_patient(icn: str, request: Request):
    # Get access token from request cookies (set during OIDC login)
    access_token = request.cookies.get("access_token")

    if not access_token:
        raise HTTPException(status_code=401, detail="Not authenticated")

    # Update CCOW context
    await ccow_client.set_active_patient(
        access_token=access_token,
        patient_id=icn,
        set_by="med-z1"
    )

    # Load patient data...
    return templates.TemplateResponse("patient.html", {...})
```

---

### JavaScript Browser Client

```javascript
// ccow-client.js
class CCOWClient {
    constructor(baseUrl = 'http://localhost:8001') {
        this.baseUrl = baseUrl;
    }

    async setActivePatient(accessToken, patientId, setBy = 'web-app') {
        const response = await fetch(`${this.baseUrl}/ccow/active-patient`, {
            method: 'PUT',
            headers: {
                'Authorization': `Bearer ${accessToken}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                patient_id: patientId,
                set_by: setBy
            })
        });

        if (!response.ok) {
            if (response.status === 401) {
                throw new Error('CCOW authentication failed');
            } else if (response.status === 422) {
                throw new Error('Invalid patient ID');
            } else {
                throw new Error(`CCOW error: ${response.statusText}`);
            }
        }

        return await response.json();
    }

    async getActivePatient(accessToken) {
        const response = await fetch(`${this.baseUrl}/ccow/active-patient`, {
            headers: {
                'Authorization': `Bearer ${accessToken}`
            }
        });

        if (response.status === 404) {
            return null;  // No active patient
        }

        if (!response.ok) {
            throw new Error(`CCOW error: ${response.statusText}`);
        }

        return await response.json();
    }

    async clearActivePatient(accessToken) {
        const response = await fetch(`${this.baseUrl}/ccow/active-patient`, {
            method: 'DELETE',
            headers: {
                'Authorization': `Bearer ${accessToken}`
            }
        });

        if (response.status === 404) {
            return null;  // No context to clear
        }

        if (!response.ok) {
            throw new Error(`CCOW error: ${response.statusText}`);
        }

        return await response.json();
    }

    async pollForContextChanges(accessToken, currentPatientId, callback) {
        try {
            const context = await this.getActivePatient(accessToken);

            if (context && context.patient_id !== currentPatientId) {
                // Context changed by another application
                callback(context.patient_id, context.set_by);
            }
        } catch (error) {
            console.error('Error polling CCOW context:', error);
        }
    }
}

// Usage example
const ccow = new CCOWClient('http://localhost:8001');
let currentPatientId = null;

// When user selects a patient
async function selectPatient(patientId) {
    const accessToken = getAccessTokenFromCookie();

    try {
        const context = await ccow.setActivePatient(accessToken, patientId, 'med-z1');
        currentPatientId = context.patient_id;
        loadPatientData(patientId);
    } catch (error) {
        alert(`Failed to set active patient: ${error.message}`);
    }
}

// Poll for context changes (every 5 seconds)
setInterval(async () => {
    const accessToken = getAccessTokenFromCookie();

    await ccow.pollForContextChanges(
        accessToken,
        currentPatientId,
        (newPatientId, setBy) => {
            alert(`Patient context changed to ${newPatientId} by ${setBy}`);
            currentPatientId = newPatientId;
            loadPatientData(newPatientId);
        }
    );
}, 5000);
```

---

## Testing with cURL

### Complete Workflow

```bash
# 1. Get JWT token from Keycloak (outside med-z8)
ACCESS_TOKEN=$(curl -X POST http://localhost:8080/realms/va-healthcare/protocol/openid-connect/token \
  -d "client_id=med-z1" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "grant_type=password" \
  -d "username=john.doe" \
  -d "password=password123" \
  -d "scope=openid email profile" \
  | jq -r '.access_token')

echo "Access Token: $ACCESS_TOKEN"

# 2. Set active patient
curl -X PUT http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"patient_id": "1012853550V207686", "set_by": "med-z1"}' \
  | jq .

# 3. Get active patient
curl -X GET http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  | jq .

# 4. Get history
curl -X GET http://localhost:8001/ccow/history \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  | jq .

# 5. Clear active patient
curl -X DELETE http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  | jq .

# 6. Verify cleared (should return 404)
curl -X GET http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  | jq .
```

---

**Document Version:** v1.0
**Last Updated:** 2026-01-08
**Author:** VA Healthcare Engineering Team
