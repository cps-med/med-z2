# CCOW Context Vault Service Design Specification

**Document Version:** v1.1
**Date:** 2026-01-09
**Status:** 🟡 DESIGN PHASE (Not yet implemented)
**Target Repository:** `med-z8` (new standalone repository)

**Recent Updates (v1.1):**
- Added token refresh handling patterns (Section 5.4)
- Added client application identity extraction from JWT `azp` claim (Section 5.6)
- Added user identity migration strategy (Section 11.2)
- Added development mode with JWT bypass (Section 10.4)
- Updated implementation checklist with revised approach

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Background and Motivation](#2-background-and-motivation)
3. [Requirements](#3-requirements)
4. [Architecture](#4-architecture)
5. [Authentication and Authorization](#5-authentication-and-authorization)
6. [Data Model](#6-data-model)
7. [API Specification](#7-api-specification)
8. [Deployment](#8-deployment)
9. [Security Considerations](#9-security-considerations)
10. [Testing Strategy](#10-testing-strategy)
11. [Migration from med-z1 CCOW](#11-migration-from-med-z1-ccow)
12. [Future Enhancements](#12-future-enhancements)
13. [Implementation Checklist](#13-implementation-checklist)

---

## 1. Executive Summary

### 1.1 Purpose

**med-z8** is a standalone CCOW (Clinical Context Object Workgroup) Context Vault service that provides patient context synchronization across multiple clinical applications in the VA healthcare ecosystem. It replaces the embedded CCOW service in med-z1 with an enterprise-grade, multi-application shared service.

### 1.2 Key Features

- **Single Sign-On (SSO)**: Integrates with Keycloak identity provider for OAuth 2.0 / OIDC authentication
- **Multi-Application Support**: Supports med-z1, CPRS simulator, imaging applications, and future med-* applications
- **Stateless JWT Authentication**: No database dependency for session validation
- **Production-Ready**: Designed for horizontal scaling, high availability, and enterprise deployment
- **Standards-Based**: Follows OAuth 2.0, OIDC, and simplified CCOW patterns

### 1.3 Success Criteria

| Criterion | Target |
|-----------|--------|
| **Authentication** | JWT validation < 50ms (p95) |
| **Context Operations** | < 100ms (p95) for get/set/clear |
| **Availability** | 99.9% uptime (3-nines) |
| **Concurrent Users** | Support 1000+ concurrent users |
| **Horizontal Scaling** | Support 3+ instances behind load balancer |
| **SSO Integration** | Seamless login across all med-* applications |

### 1.4 Design Principles

1. **Stateless First**: JWT validation requires no database lookups
2. **Standards-Based**: OAuth 2.0, OIDC, REST API
3. **Keycloak-Native**: Designed for Keycloak but adaptable to other OIDC providers
4. **Cloud-Ready**: Container-native, 12-factor app principles
5. **Observable**: Comprehensive logging, metrics, and health checks

---

## 2. Background and Motivation

### 2.1 Current State (med-z1 Embedded CCOW)

The current CCOW implementation lives inside the med-z1 repository:

**Limitations:**
- ❌ Tied to med-z1's custom session-based authentication
- ❌ PostgreSQL dependency for session validation (auth.sessions table)
- ❌ Cannot be used by non-med-z1 applications (CPRS, imaging)
- ❌ No SSO support
- ❌ Single application scope

**Architecture:**
```
med-z1 (Port 8000)
  ├── app/ (main application)
  └── ccow/ (embedded CCOW service, Port 8001)
       └── auth_helper.py → PostgreSQL (auth.sessions)
```

### 2.2 Target State (med-z8 Standalone Service)

med-z8 becomes a standalone, enterprise-grade CCOW service:

**Benefits:**
- ✅ **SSO Integration**: Keycloak-based authentication, shared across all applications
- ✅ **Multi-Application**: med-z1, CPRS, imaging, future med-* apps
- ✅ **Stateless**: JWT validation (no database dependency)
- ✅ **Independent Deployment**: Separate repository, versioning, scaling
- ✅ **Production-Ready**: Designed for VA enterprise deployment

**Architecture:**
```
┌──────────────────────────────────────────────────┐
│        Keycloak Identity Provider (Port 8080)    │
│  Realm: va-development                           │
│  Issues: JWT access tokens (15 min expiration)   │
└────────────────────┬─────────────────────────────┘
                     │
        ┌────────────┼────────────┬─────────────┐
        │ OIDC       │ OIDC       │ OIDC        │
        ▼            ▼            ▼             ▼
   ┌────────┐  ┌────────┐  ┌─────────┐  ┌──────────┐
   │med-z1  │  │ CPRS   │  │Imaging  │  │Future app│
   │(8000)  │  │(8002)  │  │(8004)   │  │          │
   └───┬────┘  └───┬────┘  └────┬────┘  └────┬─────┘
       │ JWT       │ JWT        │ JWT        │ JWT
       └───────────┼────────────┼────────────┘
                   ▼
            ┌──────────────┐
            │   med-z8     │
            │    CCOW      │
            │  (Port 8001) │
            │              │
            │ ✅ Stateless │
            │ ✅ JWT only  │
            │ ✅ No DB     │
            └──────────────┘
```

### 2.3 Strategic Alignment

**Alignment with VA Enterprise Architecture:**
- Uses **OAuth 2.0 / OIDC** (same as VA IAM, Azure AD in production)
- **Keycloak** in development mirrors Azure AD patterns in production
- **JWT bearer tokens** align with VA enterprise security standards
- **Microservices architecture** enables independent scaling and deployment

**Alignment with med-* Family Vision:**
- **Shared identity provider** (Keycloak) for all med-* applications
- **Shared context service** (med-z8) for patient context synchronization
- **Reusable patterns** documented in template design spec

---

## 3. Requirements

### 3.1 Functional Requirements

**FR-1: Multi-User Context Isolation**
Each authenticated user SHALL maintain an independent active patient context, identified by `user_id` (from JWT `sub` claim).

**FR-2: Context Persistence**
User contexts SHALL persist in memory and optionally in persistent storage (Redis/PostgreSQL) for recovery after service restart.

**FR-3: JWT-Based Authentication**
med-z8 SHALL validate JWT access tokens issued by Keycloak and extract user identity (`sub`, `email`, `name`) from token claims.

**FR-4: Multi-Application Support**
med-z8 SHALL support context synchronization for multiple client applications (med-z1, CPRS, imaging) authenticated by Keycloak.

**FR-5: Stateless Operation**
JWT validation SHALL NOT require database lookups (stateless validation using Keycloak public keys).

**FR-6: User-Scoped History**
med-z8 SHALL maintain context change history with user/global scope filtering.

**FR-7: Health and Metrics**
med-z8 SHALL expose health check and metrics endpoints for monitoring and observability.

**FR-8: CORS Support**
med-z8 SHALL support Cross-Origin Resource Sharing (CORS) for browser-based applications.

### 3.2 Non-Functional Requirements

**NFR-1: Performance**
- JWT validation: < 50ms (p95)
- Context operations (get/set/clear): < 100ms (p95)
- Health check: < 10ms

**NFR-2: Scalability**
- Support 1000+ concurrent users
- Horizontal scaling (3+ instances behind load balancer)
- Stateless design enables unlimited scaling

**NFR-3: Availability**
- 99.9% uptime (3-nines)
- No single point of failure (with persistent storage)
- Graceful degradation if Keycloak unavailable (cached JWKS)

**NFR-4: Security**
- TLS/HTTPS in production
- JWT signature validation
- CORS policy enforcement
- Audit logging for all context changes

**NFR-5: Observability**
- Structured JSON logging
- Prometheus-compatible metrics
- OpenTelemetry instrumentation (future)

**NFR-6: Maintainability**
- Comprehensive unit and integration tests (80%+ coverage)
- OpenAPI/Swagger documentation
- Clear error messages and diagnostics

### 3.3 Out of Scope (Phase 1)

- ❌ Full CCOW standard compliance (transaction protocol, voting)
- ❌ Real-time push notifications (WebSocket/SSE)
- ❌ Multi-session per user (single session enforcement)
- ❌ Role-based access control (all authenticated users have equal access)
- ❌ Context encryption at rest

---

## 4. Architecture

### 4.1 System Context Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    VA Healthcare Ecosystem                       │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │          Keycloak Identity Provider (Port 8080)           │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ Realm: va-development                              │  │  │
│  │  │ Users:                                             │  │  │
│  │  │  - clinician.alpha@va.gov                         │  │  │
│  │  │  - clinician.bravo@va.gov                         │  │  │
│  │  │  - admin.charlie@va.gov                           │  │  │
│  │  │                                                     │  │  │
│  │  │ Clients:                                           │  │  │
│  │  │  - med-z1 (OIDC authorization code flow)          │  │  │
│  │  │  - med-z8-ccow (JWT bearer validation)            │  │  │
│  │  │  - cprs-simulator (OIDC authorization code flow)  │  │  │
│  │  │                                                     │  │  │
│  │  │ Issues: JWT Access Tokens (15 min expiration)     │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         │                                       │
│        ┌────────────────┼────────────────┬──────────────┐      │
│        │ OIDC Login     │ OIDC Login     │ OIDC Login   │      │
│        ▼                ▼                ▼              ▼      │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐  ┌─────────┐ │
│   │ med-z1  │      │  CPRS   │      │ Imaging │  │Future   │ │
│   │ (8000)  │      │ (8002)  │      │ (8004)  │  │med-*    │ │
│   └────┬────┘      └────┬────┘      └────┬────┘  └────┬────┘ │
│        │ JWT            │ JWT            │ JWT        │ JWT   │
│        │ Bearer         │ Bearer         │ Bearer     │ Bearer│
│        └────────────────┼────────────────┼────────────┘       │
│                         ▼                                       │
│              ┌──────────────────────┐                          │
│              │      med-z8 CCOW     │                          │
│              │  Context Vault API   │                          │
│              │    (Port 8001)       │                          │
│              │                       │                          │
│              │ ┌───────────────────┐│                          │
│              │ │ JWT Validator     ││                          │
│              │ │ - Fetch JWKS      ││                          │
│              │ │ - Verify signature││                          │
│              │ │ - Extract claims  ││                          │
│              │ └───────────────────┘│                          │
│              │                       │                          │
│              │ ┌───────────────────┐│                          │
│              │ │ Context Vault     ││                          │
│              │ │ - In-memory       ││                          │
│              │ │ - Per-user ctx    ││                          │
│              │ │ - History tracking││                          │
│              │ └───────────────────┘│                          │
│              │                       │                          │
│              │ Optional:             │                          │
│              │ ┌───────────────────┐│                          │
│              │ │ Persistent Store  ││                          │
│              │ │ - Redis (cache)   ││                          │
│              │ │ - PostgreSQL (DB) ││                          │
│              │ └───────────────────┘│                          │
│              └──────────────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Component Architecture

```
med-z8/
├── ccow/
│   ├── __init__.py
│   ├── main.py                  # FastAPI application
│   ├── models.py                # Pydantic models
│   ├── vault.py                 # Context storage (in-memory)
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── jwt_validator.py    # JWT validation logic
│   │   └── dependencies.py     # FastAPI dependencies (get_current_user)
│   ├── storage/
│   │   ├── __init__.py
│   │   ├── memory.py           # In-memory storage (default)
│   │   ├── redis.py            # Redis storage (optional)
│   │   └── postgres.py         # PostgreSQL storage (optional)
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── cors.py             # CORS configuration
│   │   └── logging.py          # Request logging
│   └── tests/
│       ├── __init__.py
│       ├── test_jwt_auth.py
│       ├── test_vault.py
│       ├── test_api.py
│       └── test_multiuser.py
├── config.py                    # Configuration management
├── requirements.txt             # Python dependencies
├── Dockerfile                   # Container image
├── docker-compose.yml           # Local development setup
├── .env.template                # Environment variables template
├── README.md                    # Getting started guide
└── docs/
    ├── spec/
    │   └── med-z8-ccow-service-design.md  # This document
    ├── architecture.md          # Architecture decisions
    └── api.md                   # API reference

```

### 4.3 Technology Stack

| Component | Technology | Rationale |
|-----------|------------|-----------|
| **Language** | Python 3.11+ | Consistency with med-z1, VA Python standards |
| **Web Framework** | FastAPI | Async, OpenAPI, Pydantic validation |
| **JWT Library** | python-jose[cryptography] | Robust JWT/JWS/JWE implementation |
| **HTTP Client** | httpx | Async HTTP for JWKS fetching |
| **Testing** | pytest, pytest-asyncio | Comprehensive test coverage |
| **Container** | Docker | Portability, deployment |
| **Orchestration** | Docker Compose (dev), Kubernetes (prod) | Local dev + production deployment |
| **Storage (optional)** | Redis or PostgreSQL | Persistent context storage |

### 4.4 Data Flow Diagrams

#### 4.4.1 User Login Flow (OIDC)

```
┌────────┐                 ┌────────┐                 ┌──────────┐
│ User   │                 │ med-z1 │                 │ Keycloak │
└───┬────┘                 └───┬────┘                 └────┬─────┘
    │                          │                           │
    │ 1. Visit /dashboard      │                           │
    ├─────────────────────────>│                           │
    │                          │                           │
    │ 2. Redirect to Keycloak  │                           │
    │<─────────────────────────┤                           │
    │                          │                           │
    │ 3. Show login page       │                           │
    ├────────────────────────────────────────────────────> │
    │                          │                           │
    │ 4. Enter credentials     │                           │
    ├────────────────────────────────────────────────────> │
    │                          │                           │
    │ 5. Redirect with code    │                           │
    │<────────────────────────────────────────────────────┤
    │                          │                           │
    │ 6. Exchange code for JWT │                           │
    ├─────────────────────────>│────────────────────────>  │
    │                          │                           │
    │ 7. Return JWT tokens     │                           │
    │                          │<─────────────────────────┤
    │                          │                           │
    │ 8. Set access_token cookie                           │
    │<─────────────────────────┤                           │
    │                          │                           │
    │ 9. Redirect to /dashboard│                           │
    │<─────────────────────────┤                           │
    │                          │                           │
```

#### 4.4.2 CCOW Context Operation Flow

```
┌────────┐         ┌────────┐         ┌────────┐         ┌──────────┐
│ med-z1 │         │ med-z8 │         │ Vault  │         │ Keycloak │
└───┬────┘         └───┬────┘         └───┬────┘         └────┬─────┘
    │                  │                  │                    │
    │ 1. PUT /ccow/active-patient        │                    │
    │    Authorization: Bearer <JWT>     │                    │
    ├─────────────────>│                  │                    │
    │                  │                  │                    │
    │ 2. Extract JWT from header         │                    │
    │                  │                  │                    │
    │ 3. Fetch JWKS (cached)             │                    │
    │                  ├─────────────────────────────────────> │
    │                  │<───────────────────────────────────── │
    │                  │                  │                    │
    │ 4. Validate JWT signature          │                    │
    │                  │ (verify exp, iss, aud)               │
    │                  │                  │                    │
    │ 5. Extract user_id from 'sub' claim│                    │
    │                  │                  │                    │
    │ 6. Set context for user_id         │                    │
    │                  ├─────────────────>│                    │
    │                  │                  │                    │
    │ 7. Return PatientContext           │                    │
    │                  │<─────────────────┤                    │
    │                  │                  │                    │
    │ 8. HTTP 200 OK                      │                    │
    │<─────────────────┤                  │                    │
    │                  │                  │                    │
```

---

## 5. Authentication and Authorization

### 5.1 Authentication Model

**OAuth 2.0 / OIDC with JWT Bearer Tokens**

med-z8 uses **JWT bearer token authentication** where:
1. Client applications (med-z1, CPRS) authenticate users via Keycloak OIDC
2. Keycloak issues JWT access tokens (short-lived, 15 minutes)
3. Clients send JWT in `Authorization: Bearer <token>` header
4. med-z8 validates JWT signature and claims (stateless, no database)

### 5.2 JWT Validation Process

```python
# ccow/auth/jwt_validator.py

from authlib.jose import jwt, JoseError
import httpx
import logging
from typing import Optional, Dict, Any
from config import OIDC_CONFIG

logger = logging.getLogger(__name__)

# Cache for JWKS (Keycloak's public keys)
_jwks_cache: Optional[Dict] = None


async def get_jwks() -> Dict:
    """
    Fetch JSON Web Key Set (JWKS) from Keycloak.

    JWKS contains public keys used to verify JWT signatures.
    Cached to avoid repeated HTTP requests.

    Returns:
        Dictionary with 'keys' array
    """
    global _jwks_cache

    if _jwks_cache is None:
        async with httpx.AsyncClient() as client:
            response = await client.get(OIDC_CONFIG["jwks_uri"])
            response.raise_for_status()
            _jwks_cache = response.json()
            logger.info("Fetched JWKS from Keycloak")

    return _jwks_cache


async def validate_jwt(token: str) -> Dict[str, Any]:
    """
    Validate JWT access token issued by Keycloak.

    Validation steps:
    1. Fetch JWKS (Keycloak public keys)
    2. Decode JWT and verify signature
    3. Validate claims (exp, nbf, iss, aud)
    4. Extract user info (sub, email, name)

    Args:
        token: JWT access token string

    Returns:
        Dictionary with JWT claims:
        {
            "sub": "user-uuid",           # User ID
            "email": "user@va.gov",       # Email
            "name": "Dr. Alice Anderson", # Display name
            "preferred_username": "alice",# Username
            "iss": "http://localhost:8080/realms/va-development",
            "exp": 1234567890,            # Expiration timestamp
            "iat": 1234567000,            # Issued at timestamp
            # ... other claims
        }

    Raises:
        JoseError: If token is invalid, expired, or signature verification fails

    Security:
        - Stateless (no database lookup)
        - Uses Keycloak's public keys (asymmetric crypto)
        - Validates expiration, issuer, audience
    """
    try:
        # Fetch JWKS (cached)
        jwks = await get_jwks()

        # Decode and verify JWT
        claims = jwt.decode(token, jwks)

        # Validate standard claims (exp, nbf, iat)
        claims.validate()

        # Validate issuer (prevent token substitution attacks)
        if claims.get("iss") != OIDC_CONFIG["issuer"]:
            raise JoseError(f"Invalid issuer: {claims.get('iss')}")

        logger.debug(f"JWT validated for user: {claims.get('email')}")
        return claims

    except JoseError as e:
        logger.warning(f"JWT validation failed: {e}")
        raise
    except Exception as e:
        logger.error(f"Unexpected error validating JWT: {e}")
        raise JoseError(f"JWT validation error: {e}")


async def get_user_from_jwt(token: str) -> Optional[Dict[str, Any]]:
    """
    Validate JWT and extract user information.

    Wrapper around validate_jwt() that returns None on error
    instead of raising exceptions (more convenient for API use).

    Returns:
        {
            "user_id": str,        # From 'sub' claim
            "email": str,          # From 'email' claim
            "display_name": str,   # From 'name' or 'preferred_username'
        }
        Returns None if token is invalid.
    """
    try:
        claims = await validate_jwt(token)
        return {
            "user_id": claims["sub"],
            "email": claims.get("email"),
            "display_name": claims.get("name") or claims.get("preferred_username"),
        }
    except JoseError:
        return None
```

### 5.3 FastAPI Dependency

```python
# ccow/auth/dependencies.py

from fastapi import Request, HTTPException, status, Depends
from typing import Dict, Any
from ccow.auth.jwt_validator import get_user_from_jwt


async def get_current_user(request: Request) -> Dict[str, Any]:
    """
    FastAPI dependency to extract and validate user from JWT.

    Looks for JWT in:
    1. Authorization: Bearer <token> header (preferred)
    2. access_token cookie (fallback for web apps)

    Usage:
        @app.get("/ccow/active-patient")
        async def get_patient(user: Dict = Depends(get_current_user)):
            # user["user_id"] is authoritative user ID from JWT
            context = vault.get_context(user["user_id"])
            ...

    Raises:
        HTTPException(401) if token is missing or invalid

    Returns:
        {
            "user_id": str,
            "email": str,
            "display_name": str,
        }
    """
    # Check Authorization header first (API clients)
    auth_header = request.headers.get("Authorization")
    if auth_header and auth_header.startswith("Bearer "):
        token = auth_header.split(" ")[1]
    else:
        # Fallback to cookie (web app sessions)
        token = request.cookies.get("access_token")

    if not token:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Missing JWT access token",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # Validate JWT
    user = await get_user_from_jwt(token)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired JWT",
            headers={"WWW-Authenticate": "Bearer"},
        )

    return user
```

### 5.4 Token Refresh Handling

**Critical Client Requirement:** Access tokens expire after 5-15 minutes (Keycloak default). Clients MUST implement token refresh logic.

**Client-Side Token Storage:**
```python
# Client applications (e.g., med-z1) must store both tokens in session
session['access_token'] = token_response['access_token']
session['refresh_token'] = token_response['refresh_token']
session['token_expires_at'] = datetime.utcnow() + timedelta(seconds=token_response['expires_in'])
```

**Proactive Token Refresh Pattern:**
```python
# ccow_client.py (client-side helper)
async def get_valid_token(request: Request) -> str:
    """Ensure token is valid, refresh if expiring soon."""
    access_token = request.session.get("access_token")
    expires_at = request.session.get("token_expires_at")

    # Refresh if token expires in <1 minute
    if datetime.utcnow() > expires_at - timedelta(minutes=1):
        refresh_token = request.session.get("refresh_token")
        new_tokens = await refresh_token_with_keycloak(refresh_token)

        # Update session with new tokens
        request.session["access_token"] = new_tokens["access_token"]
        request.session["token_expires_at"] = datetime.utcnow() + timedelta(seconds=new_tokens["expires_in"])
        access_token = new_tokens["access_token"]

    return access_token
```

**Keycloak Token Refresh Endpoint:**
```http
POST /realms/va-healthcare/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
client_id=med-z1&
client_secret=<secret>&
refresh_token=<refresh_token>
```

### 5.5 Security Properties

| Property | Implementation |
|----------|----------------|
| **Confidentiality** | TLS/HTTPS in production |
| **Integrity** | JWT signature validation (RS256) |
| **Authentication** | JWT `sub` claim (user_id) |
| **Authorization** | All authenticated users have equal access (Phase 1) |
| **Non-Repudiation** | Audit logs with user_id, timestamp, action |
| **Expiration** | JWT `exp` claim (5-15 min default) |
| **Token Refresh** | Client implements proactive refresh (see 5.4) |

### 5.6 Client Application Identity (set_by Field)

**Security Requirement:** The `set_by` field identifies which application set the context. This must be trustworthy for audit purposes.

**Recommended Approach: Extract from JWT `azp` Claim**

```python
# ccow/auth/dependencies.py
async def get_current_user(request: Request) -> Dict[str, Any]:
    """
    Extract user AND application identity from JWT.

    Returns:
        {
            "user_id": str,        # From 'sub' claim
            "email": str,          # From 'email' claim
            "display_name": str,   # From 'name' claim
            "client_id": str,      # From 'azp' (authorized party) claim
        }
    """
    # ... JWT validation ...

    claims = await validate_jwt(token)
    return {
        "user_id": claims["sub"],
        "email": claims.get("email"),
        "display_name": claims.get("name"),
        "client_id": claims.get("azp", "unknown"),  # ✅ Trusted application ID
    }

# ccow/routes/context.py
@router.put("/ccow/active-patient")
async def set_active_patient(
    request: SetPatientRequest,
    user: dict = Depends(get_current_user)
):
    # ✅ Use client_id from validated JWT (cannot be spoofed)
    # Fallback to request.set_by if azp claim not present
    set_by = user["client_id"] if user["client_id"] != "unknown" else request.set_by

    context = PatientContext(
        user_id=user["user_id"],
        patient_id=request.patient_id,
        set_by=set_by,  # ✅ From JWT azp claim (trusted)
        # ...
    )
```

**Why This Matters:**
- **Without `azp` extraction:** A compromised med-z1 instance could claim `set_by="cprs"` in audit logs
- **With `azp` extraction:** Application identity comes from cryptographically signed JWT (trustworthy)

**Request Model Update:**
```python
class SetPatientContextRequest(BaseModel):
    patient_id: str = Field(..., min_length=1)
    set_by: Optional[str] = Field(default=None, min_length=1)  # ✅ Optional fallback
```

### 5.7 Authorization Model (Phase 1)

**Phase 1: Flat Authorization**
- All authenticated users can:
  - Get/set/clear their own patient context
  - View their own context history
  - View admin endpoints (all contexts, cleanup)

**Future: Role-Based Access Control (RBAC)**
- Extract `roles` claim from JWT
- Restrict admin endpoints to `admin` role
- Add read-only role for observers

---

## 6. Data Model

### 6.1 Core Models (Pydantic)

```python
# ccow/models.py

from datetime import datetime, timezone
from typing import Optional, Literal, List
from pydantic import BaseModel, Field


class PatientContext(BaseModel):
    """Active patient context for a specific user."""

    user_id: str = Field(..., description="User UUID (from JWT 'sub' claim)")
    email: Optional[str] = Field(None, description="User email")
    patient_id: str = Field(..., description="Patient ICN")
    set_by: str = Field(..., description="Application name (e.g., 'med-z1', 'cprs')")
    set_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),
        description="UTC timestamp when context was set"
    )
    last_accessed_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),
        description="UTC timestamp of last access (for cleanup)"
    )

    class Config:
        json_schema_extra = {
            "example": {
                "user_id": "550e8400-e29b-41d4-a716-446655440000",
                "email": "clinician.alpha@va.gov",
                "patient_id": "1012845331V153053",
                "set_by": "med-z1",
                "set_at": "2026-01-08T15:30:00.000Z",
                "last_accessed_at": "2026-01-08T15:45:00.000Z"
            }
        }


class ContextHistoryEntry(BaseModel):
    """Historical context change event."""

    action: Literal["set", "clear"] = Field(..., description="Action type")
    user_id: str = Field(..., description="User UUID")
    email: Optional[str] = Field(None, description="User email")
    patient_id: Optional[str] = Field(None, description="Patient ICN (None if cleared)")
    actor: str = Field(..., description="Application that performed action")
    timestamp: datetime = Field(..., description="UTC timestamp")

    class Config:
        json_schema_extra = {
            "example": {
                "action": "set",
                "user_id": "550e8400-e29b-41d4-a716-446655440000",
                "email": "clinician.alpha@va.gov",
                "patient_id": "1012845331V153053",
                "actor": "med-z1",
                "timestamp": "2026-01-08T15:30:00.000Z"
            }
        }


# Request/Response models
class SetPatientContextRequest(BaseModel):
    patient_id: str = Field(..., min_length=1)
    set_by: str = Field(default="unknown", min_length=1)


class ClearPatientContextRequest(BaseModel):
    cleared_by: Optional[str] = Field(default=None)


class GetAllActiveContextsResponse(BaseModel):
    contexts: List[PatientContext]
    total_count: int


class GetHistoryResponse(BaseModel):
    history: List[ContextHistoryEntry]
    scope: Literal["user", "global"]
    total_count: int
    user_id: Optional[str] = None


class CleanupResponse(BaseModel):
    removed_count: int
    message: str


class HealthCheckResponse(BaseModel):
    status: Literal["healthy", "unhealthy"]
    service: str
    version: str
    timestamp: str
    active_contexts: int
```

### 6.2 Storage Interface

```python
# ccow/storage/__init__.py

from abc import ABC, abstractmethod
from typing import Optional, List
from ccow.models import PatientContext, ContextHistoryEntry


class ContextStorage(ABC):
    """Abstract interface for context storage backends."""

    @abstractmethod
    async def get_context(self, user_id: str) -> Optional[PatientContext]:
        """Get active context for user."""
        pass

    @abstractmethod
    async def set_context(self, context: PatientContext) -> PatientContext:
        """Set active context for user."""
        pass

    @abstractmethod
    async def clear_context(self, user_id: str) -> bool:
        """Clear active context for user. Returns True if existed."""
        pass

    @abstractmethod
    async def get_all_contexts(self) -> List[PatientContext]:
        """Get all active contexts (admin)."""
        pass

    @abstractmethod
    async def add_history_entry(self, entry: ContextHistoryEntry) -> None:
        """Add entry to history."""
        pass

    @abstractmethod
    async def get_history(
        self,
        user_id: Optional[str] = None,
        scope: str = "user"
    ) -> List[ContextHistoryEntry]:
        """Get history entries (filtered by user or global)."""
        pass

    @abstractmethod
    async def cleanup_stale_contexts(self, threshold_hours: int = 24) -> int:
        """Remove stale contexts. Returns count removed."""
        pass
```

---

## 7. API Specification

### 7.1 Endpoints Summary

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/` | GET | ❌ | Service info |
| `/health` | GET | ❌ | Health check |
| `/ccow/active-patient` | GET | ✅ JWT | Get user's active patient |
| `/ccow/active-patient` | PUT | ✅ JWT | Set user's active patient |
| `/ccow/active-patient` | DELETE | ✅ JWT | Clear user's active patient |
| `/ccow/history` | GET | ✅ JWT | Get context history (user/global) |
| `/ccow/active-patients` | GET | ✅ JWT | Get all active contexts (admin) |
| `/ccow/cleanup` | POST | ✅ JWT | Cleanup stale contexts (admin) |

### 7.2 OpenAPI Documentation

Auto-generated at:
- **Swagger UI**: http://localhost:8001/docs
- **ReDoc**: http://localhost:8001/redoc

### 7.3 Example API Calls

**Get Active Patient (with JWT bearer token):**
```bash
curl -X GET http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Set Active Patient:**
```bash
curl -X PUT http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "patient_id": "1012845331V153053",
    "set_by": "med-z1"
  }'
```

**Get History (user-scoped):**
```bash
curl -X GET "http://localhost:8001/ccow/history?scope=user" \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

## 8. Deployment

### 8.1 Docker Compose (Development)

```yaml
# docker-compose.yml

version: '3.8'

services:
  # Keycloak Identity Provider
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    container_name: med-keycloak
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: postgres
      KC_DB_PASSWORD: postgres
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_HOSTNAME: localhost
      KC_HOSTNAME_PORT: 8080
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT_HTTPS: "false"
    command: start-dev
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    networks:
      - med-network

  # PostgreSQL (shared: Keycloak + optional med-z8 storage)
  postgres:
    image: postgres:15
    container_name: med-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: keycloak
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - med-network

  # med-z8 CCOW Service
  med-z8-ccow:
    build: .
    container_name: med-z8-ccow
    environment:
      KEYCLOAK_SERVER_URL: http://keycloak:8080
      KEYCLOAK_REALM: va-development
      KEYCLOAK_CLIENT_ID: med-z8-ccow
      LOG_LEVEL: INFO
      STORAGE_BACKEND: memory  # or 'redis', 'postgres'
    ports:
      - "8001:8001"
    depends_on:
      - keycloak
    networks:
      - med-network
    command: uvicorn ccow.main:app --host 0.0.0.0 --port 8001 --reload

  # Redis (optional, for persistent context storage)
  redis:
    image: redis:7-alpine
    container_name: med-redis
    ports:
      - "6379:6379"
    networks:
      - med-network

volumes:
  postgres_data:

networks:
  med-network:
    driver: bridge
```

### 8.2 Dockerfile

```dockerfile
# Dockerfile

FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY ccow/ ./ccow/
COPY config.py .

# Expose port
EXPOSE 8001

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8001/health')"

# Run application
CMD ["uvicorn", "ccow.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

### 8.3 Kubernetes Deployment (Production)

```yaml
# k8s/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: med-z8-ccow
  labels:
    app: med-z8-ccow
spec:
  replicas: 3  # 3 instances for high availability
  selector:
    matchLabels:
      app: med-z8-ccow
  template:
    metadata:
      labels:
        app: med-z8-ccow
    spec:
      containers:
      - name: med-z8-ccow
        image: ghcr.io/va/med-z8-ccow:latest
        ports:
        - containerPort: 8001
        env:
        - name: KEYCLOAK_SERVER_URL
          value: "https://keycloak.va.gov"
        - name: KEYCLOAK_REALM
          value: "va-production"
        - name: KEYCLOAK_CLIENT_ID
          value: "med-z8-ccow"
        - name: STORAGE_BACKEND
          value: "redis"
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: med-z8-secrets
              key: redis-url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8001
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8001
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: med-z8-ccow
spec:
  selector:
    app: med-z8-ccow
  ports:
  - protocol: TCP
    port: 8001
    targetPort: 8001
  type: LoadBalancer
```

---

## 9. Security Considerations

### 9.1 Threat Model

| Threat | Mitigation |
|--------|------------|
| **JWT Token Theft** | - HTTPS only in production<br>- HttpOnly cookies<br>- Short token lifetime (15 min) |
| **Token Replay** | - JWT expiration validation<br>- Optional: JTI (JWT ID) tracking |
| **User Impersonation** | - JWT signature validation<br>- user_id from `sub` claim (trusted) |
| **MITM Attacks** | - TLS 1.2+ required<br>- Certificate pinning (optional) |
| **XSS Attacks** | - CORS policy enforcement<br>- HttpOnly cookies<br>- CSP headers (client apps) |
| **Denial of Service** | - Rate limiting (future)<br>- Resource limits in K8s<br>- JWKS caching |

### 9.2 Security Checklist

- [ ] TLS/HTTPS enabled in production
- [ ] JWKS fetched over HTTPS
- [ ] JWT signature validated (RS256 algorithm)
- [ ] JWT expiration enforced (`exp` claim)
- [ ] Issuer validation (`iss` claim)
- [ ] CORS policy configured (allowed origins)
- [ ] Audit logging enabled (all context changes)
- [ ] Secrets stored in environment variables (not code)
- [ ] Container runs as non-root user
- [ ] Health endpoint does not leak sensitive info

---

## 10. Testing Strategy

### 10.1 Unit Tests

```python
# ccow/tests/test_jwt_auth.py

import pytest
from ccow.auth.jwt_validator import validate_jwt, get_user_from_jwt


@pytest.mark.asyncio
async def test_validate_jwt_success(valid_jwt_token, mock_jwks):
    """Test JWT validation with valid token."""
    claims = await validate_jwt(valid_jwt_token)
    assert claims["sub"] == "user-123"
    assert claims["email"] == "user@va.gov"


@pytest.mark.asyncio
async def test_validate_jwt_expired(expired_jwt_token):
    """Test JWT validation rejects expired token."""
    with pytest.raises(JoseError):
        await validate_jwt(expired_jwt_token)


@pytest.mark.asyncio
async def test_validate_jwt_invalid_signature(tampered_jwt_token):
    """Test JWT validation rejects tampered token."""
    with pytest.raises(JoseError):
        await validate_jwt(tampered_jwt_token)
```

### 10.2 Integration Tests

```python
# ccow/tests/test_api.py

from fastapi.testclient import TestClient
from ccow.main import app

client = TestClient(app)


def test_get_active_patient_requires_auth():
    """Test that endpoint requires JWT."""
    response = client.get("/ccow/active-patient")
    assert response.status_code == 401


def test_set_and_get_active_patient(jwt_token_user1):
    """Test setting and retrieving patient context."""
    # Set context
    response = client.put(
        "/ccow/active-patient",
        json={"patient_id": "P1", "set_by": "med-z1"},
        headers={"Authorization": f"Bearer {jwt_token_user1}"}
    )
    assert response.status_code == 200

    # Get context
    response = client.get(
        "/ccow/active-patient",
        headers={"Authorization": f"Bearer {jwt_token_user1}"}
    )
    assert response.status_code == 200
    assert response.json()["patient_id"] == "P1"


def test_multi_user_isolation(jwt_token_user1, jwt_token_user2):
    """Test that users have isolated contexts."""
    # User 1 sets P1
    client.put(
        "/ccow/active-patient",
        json={"patient_id": "P1", "set_by": "med-z1"},
        headers={"Authorization": f"Bearer {jwt_token_user1}"}
    )

    # User 2 sets P2
    client.put(
        "/ccow/active-patient",
        json={"patient_id": "P2", "set_by": "cprs"},
        headers={"Authorization": f"Bearer {jwt_token_user2}"}
    )

    # User 1 still sees P1
    response = client.get(
        "/ccow/active-patient",
        headers={"Authorization": f"Bearer {jwt_token_user1}"}
    )
    assert response.json()["patient_id"] == "P1"

    # User 2 sees P2
    response = client.get(
        "/ccow/active-patient",
        headers={"Authorization": f"Bearer {jwt_token_user2}"}
    )
    assert response.json()["patient_id"] == "P2"
```

### 10.3 Test Coverage Target

- **Unit Tests**: 80%+ code coverage
- **Integration Tests**: All API endpoints covered
- **Load Tests**: 1000 concurrent users (future)

### 10.4 Development Mode (JWT Bypass)

**Problem:** Running Keycloak for every development iteration adds ~10 seconds startup time and resource overhead.

**Solution:** Implement optional "dev mode" that bypasses JWT validation for rapid local testing.

**Implementation:**

```python
# ccow/auth/jwt_validator.py
import os

DEV_MODE = os.getenv("DEV_MODE", "false").lower() == "true"
DEV_TOKEN = "dev-token"
DEV_USER = {
    "sub": "dev-user-00000000-0000-0000-0000-000000000001",
    "email": "developer@localhost",
    "name": "Dev User",
    "azp": "dev-client"
}

async def validate_jwt(token: str) -> dict:
    """
    Validate JWT access token issued by Keycloak.

    In DEV_MODE, accepts magic token "dev-token" for rapid testing.
    """
    # ⚠️ DEV MODE: Bypass JWT validation (NEVER enable in production)
    if DEV_MODE:
        if token == DEV_TOKEN:
            logger.warning("DEV_MODE enabled - using mock user")
            return DEV_USER
        # Still validate real tokens in dev mode

    # Normal Keycloak validation
    jwks = await get_jwks()
    claims = jwt.decode(token, jwks, ...)
    return claims


# Startup validation to prevent dev mode in production
# ccow/main.py
@app.on_event("startup")
async def validate_configuration():
    """Validate configuration on startup."""
    if os.getenv("DEV_MODE", "").lower() == "true":
        if os.getenv("ENVIRONMENT", "").lower() == "production":
            raise RuntimeError(
                "FATAL: DEV_MODE cannot be enabled in production environment"
            )
        logger.warning("=" * 60)
        logger.warning("⚠️  DEV MODE ENABLED - JWT validation bypassed")
        logger.warning("⚠️  Use 'Authorization: Bearer dev-token' for testing")
        logger.warning("⚠️  NEVER enable DEV_MODE in production")
        logger.warning("=" * 60)
```

**Configuration:**

```bash
# .env (development only)
DEV_MODE=true
ENVIRONMENT=development

# Test with dev token
curl -X GET http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer dev-token"
```

**Benefits:**
- ✅ Fast iteration: No Keycloak startup required
- ✅ Unit testing: Mock authentication in tests
- ✅ Offline development: No external dependencies

**Safety:**
- ❌ Application refuses to start if `DEV_MODE=true` AND `ENVIRONMENT=production`
- ✅ Prominent warning logs when dev mode enabled
- ✅ Real JWT validation still works (for testing with Keycloak)

---

## 11. Migration from med-z1 CCOW

### 11.1 Migration Strategy

**Phased Approach:**

1. **Phase 1: Deploy med-z8 alongside med-z1 CCOW** (no changes to med-z1)
   - Standalone med-z8 service running
   - med-z1 still uses embedded CCOW

2. **Phase 2: Update med-z1 to use Keycloak** (see med-z1-keycloak-migration.md)
   - med-z1 switches to OIDC authentication
   - med-z1 still uses embedded CCOW (JWT-based)

3. **Phase 3: Switch med-z1 to med-z8 CCOW**
   - Update med-z1 CCOW client to call med-z8
   - Deprecate med-z1 embedded CCOW service

4. **Phase 4: Remove med-z1 embedded CCOW**
   - Delete `ccow/` folder from med-z1 repository

### 11.2 User Identity Migration Strategy

**Critical Issue:** Existing med-z1 users have PostgreSQL-generated UUIDs. When users authenticate with Keycloak, they will receive NEW UUIDs in the JWT `sub` claim.

**Impact:**
- **Old med-z1 user_id:** `12345678-abcd-ef01-2345-678901234567` (PostgreSQL UUID)
- **New Keycloak sub claim:** `a1b2c3d4-e5f6-7890-abcd-ef1234567890` (Keycloak UUID)
- ❌ **These WILL NOT match**

**Recommended Strategy: Accept Clean Break**

```
Migration Event: med-z1 switches to Keycloak authentication

Before:
  med-z1 PostgreSQL: user.id = "12345678-abcd-ef01-2345-678901234567"
  med-z1 CCOW Context: keyed by PostgreSQL user.id

After:
  Keycloak JWT: sub = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  med-z8 CCOW Context: keyed by Keycloak sub

Result:
  ✅ Users get new user_id from Keycloak
  ✅ Context history reset (acceptable - ephemeral data)
  ✅ Users select patient again after first Keycloak login
```

**Why This Approach:**
- ✅ Simple: No mapping table or migration logic
- ✅ Clean: New identity system, fresh start
- ✅ Acceptable: CCOW context is ephemeral (not critical data)

**Alternative (If Historical Continuity Required):**

If maintaining context across migration is critical:

```python
# med-z1 database schema update
ALTER TABLE auth.users ADD COLUMN keycloak_sub UUID;

# During Keycloak migration, store both IDs
UPDATE auth.users
SET keycloak_sub = '<new-keycloak-uuid>'
WHERE id = '<old-postgres-uuid>';

# med-z8 would need migration endpoint
POST /ccow/migrate-user-id
{
  "old_user_id": "12345678-abcd-ef01-2345-678901234567",
  "new_user_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

**Decision:** Use clean break approach unless business requires historical continuity.

### 11.3 Data Migration

**No data migration needed** - CCOW context is ephemeral (in-memory).

Users will naturally re-establish context when they select patients in med-z8.

### 11.4 Rollback Plan

If issues arise, rollback by reverting med-z1 CCOW client to use embedded service.

**Rollback Procedure:**
1. Update med-z1 environment variable: `CCOW_SERVICE_URL=http://localhost:8001` (embedded)
2. Restart med-z1 application
3. med-z8 service can remain running (unused)
4. No data loss (context is ephemeral)

---

## 12. Future Enhancements

### 12.1 Phase 2: Persistent Storage

**Objective**: Context survives service restarts

**Implementation**:
- Redis backend for active contexts (fast)
- PostgreSQL backend for audit history (durable)

### 12.2 Phase 3: Real-Time Notifications

**Objective**: Push context changes to subscribed clients

**Implementation**:
- WebSocket endpoint `/ccow/subscribe`
- Server-Sent Events (SSE) alternative
- Publish/subscribe pattern (Redis Pub/Sub)

### 12.3 Phase 4: Role-Based Access Control

**Objective**: Restrict admin endpoints to admin users

**Implementation**:
- Extract `roles` claim from JWT
- Add `@require_role("admin")` decorator
- Return 403 Forbidden for unauthorized users

### 12.4 Phase 5: Metrics and Observability

**Objective**: Prometheus metrics, OpenTelemetry tracing

**Implementation**:
- Prometheus exporter at `/metrics`
- OpenTelemetry instrumentation
- Grafana dashboards

---

## 13. Implementation Checklist

**Recommended Implementation Order:** Start with mock authentication for rapid iteration, then integrate Keycloak once core logic is working.

### Phase 1: Project Scaffold and Mock Auth (Days 1-2)

- [ ] **Repository Setup**
  - [ ] Create med-z8 repository structure
  - [ ] Set up `ccow/`, `tests/`, `docs/` directories
  - [ ] Create `requirements.txt`, `.env.template`, `.gitignore`
  - [ ] Initialize pytest configuration

- [ ] **Mock Authentication (for rapid iteration)**
  - [ ] Implement `DEV_MODE` token bypass in `ccow/auth/jwt_validator.py`
  - [ ] Create `get_current_user()` FastAPI dependency
  - [ ] Test with `Authorization: Bearer dev-token`
  - [ ] Document dev mode in README.md

- [ ] **FastAPI Skeleton**
  - [ ] Create `ccow/main.py` with basic FastAPI app
  - [ ] Add health check endpoint
  - [ ] Add startup configuration validation
  - [ ] Test: `curl http://localhost:8001/health`

### Phase 2: Port Core CCOW Logic from med-z1 (Days 3-4)

- [ ] **Storage Implementation**
  - [ ] Port in-memory storage from med-z1 v2.0
  - [ ] Implement `ContextStorage` abstract base class
  - [ ] Implement `InMemoryStorage` with thread-safe dict
  - [ ] Write unit tests for storage layer

- [ ] **Data Models**
  - [ ] Define Pydantic models (`PatientContext`, `ContextHistoryEntry`)
  - [ ] Define request/response models
  - [ ] Write model validation tests

- [ ] **Vault Operations**
  - [ ] Implement get/set/clear context operations
  - [ ] Implement history tracking (100-entry circular buffer)
  - [ ] Test multi-user isolation
  - [ ] Write comprehensive unit tests

### Phase 3: API Endpoints with Mock Auth (Days 5-6)

- [ ] **Context Endpoints**
  - [ ] `GET /ccow/active-patient`
  - [ ] `PUT /ccow/active-patient` (use `azp` claim for `set_by`)
  - [ ] `DELETE /ccow/active-patient`
  - [ ] Test with dev-token and Swagger UI

- [ ] **Admin Endpoints**
  - [ ] `GET /ccow/history` (user-scoped and global)
  - [ ] `GET /ccow/active-patients`
  - [ ] `POST /ccow/cleanup`

- [ ] **Middleware**
  - [ ] Configure CORS (environment-based allowed origins)
  - [ ] Add request logging middleware
  - [ ] Write integration tests (TestClient)

### Phase 4: Keycloak Integration (Days 7-9)

- [ ] **Keycloak Setup**
  - [ ] Add Keycloak to `docker-compose.yml`
  - [ ] Create realm: `va-healthcare`
  - [ ] Create client: `med-z8` (bearer-only)
  - [ ] Create test users
  - [ ] Document setup in `docs/guide/keycloak-setup.md`

- [ ] **Real JWT Validation**
  - [ ] Implement JWKS fetching with caching
  - [ ] Implement JWT signature validation (RS256)
  - [ ] Validate `exp`, `iss`, `aud`, `sub` claims
  - [ ] Extract `azp` claim for client identity
  - [ ] Implement cache invalidation on validation failure
  - [ ] Write JWT validation unit tests

- [ ] **Integration Testing**
  - [ ] Test with real Keycloak-issued tokens
  - [ ] Test token expiration handling
  - [ ] Test invalid token rejection
  - [ ] Test multi-user isolation with real JWTs

### Phase 5: Client Integration (Days 10-11)

- [ ] **Python Client Library**
  - [ ] Create `CCOWClient` class with token refresh logic
  - [ ] Implement retry logic and error handling
  - [ ] Write client integration tests
  - [ ] Document client usage patterns

- [ ] **med-z1 Integration**
  - [ ] Update med-z1 to call med-z8 endpoints
  - [ ] Implement token refresh in med-z1
  - [ ] Test end-to-end flow
  - [ ] Document in `docs/spec/med-z1-keycloak-migration.md`

### Phase 6: Production Readiness (Days 12-15)

- [ ] **Documentation**
  - [ ] Complete README.md with quickstart
  - [ ] Document all API endpoints with examples
  - [ ] Create deployment guide
  - [ ] Write troubleshooting guide

- [ ] **Deployment Artifacts**
  - [ ] Create Dockerfile with health check
  - [ ] Create Kubernetes manifests (Deployment, Service, Ingress)
  - [ ] Add Prometheus metrics endpoint (optional)
  - [ ] Test container deployment

- [ ] **Testing and Validation**
  - [ ] Achieve 80%+ code coverage
  - [ ] Run load tests (1000 concurrent users)
  - [ ] Security audit (JWT validation, CORS, rate limiting)
  - [ ] Code review
  - [ ] Documentation review

### Implementation Tips

**Start Simple:**
1. Use DEV_MODE for first 3 phases (no Keycloak dependency)
2. Port battle-tested code from med-z1 v2.0
3. Test thoroughly with mock auth before Keycloak integration
4. Add Keycloak in Phase 4 when core logic is solid

**Key Decision Points:**
- [ ] Confirm user identity migration strategy (clean break vs. UUID mapping)
- [ ] Confirm `set_by` extraction from JWT `azp` claim
- [ ] Decide on rate limiting inclusion (Phase 1 or Phase 2)

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| v1.0 | 2026-01-08 | System | Initial design specification for med-z8 CCOW service |
| v1.1 | 2026-01-09 | System | Incorporated technical review feedback:<br>- Added token refresh handling (5.4)<br>- Added JWT `azp` claim extraction for client identity (5.6)<br>- Added user identity migration strategy (11.2)<br>- Added dev mode with JWT bypass (10.4)<br>- Revised implementation checklist with mock-auth-first approach |

---

**End of Document**
