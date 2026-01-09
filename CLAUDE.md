# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in the **med-z8** repository.

## Project Overview

**med-z8** is a standalone Clinical Context Object Workgroup (CCOW) service that provides patient context synchronization across multiple clinical applications in the VA healthcare ecosystem. It serves as the single source of truth for "which patient is currently active" for any authenticated user across med-z1, CPRS, imaging systems, and other clinical applications.

**Key Design Principles:**
- **Stateless Architecture:** JWT-based authentication, no session database dependency
- **Multi-Application Support:** Single service shared by entire med-* family and beyond
- **Standards-Based:** OAuth 2.0 / OIDC authentication, RESTful API design
- **Per-User Context Isolation:** Each user has independent patient context
- **Production-Ready:** Thread-safe, horizontally scalable, container-native
- **Simple CCOW Subset:** REST API (not full HL7 CCOW standard)

## Project Structure

```
med-z8/
  app/
    auth/           # JWT validation, Keycloak integration
      __init__.py
      jwt_validator.py
      dependencies.py
    models/         # Pydantic models
      __init__.py
      context.py
      requests.py
      responses.py
    storage/        # Context storage backends
      __init__.py
      base.py
      in_memory.py
      redis.py (future)
      postgres.py (future)
    routes/         # API endpoints
      __init__.py
      context.py
      history.py
      health.py
    main.py         # FastAPI application
    config.py       # Configuration management
  scripts/          # Testing, setup, and utility scripts
  tests/
    unit/
    integration/
    load/
  docs/
    spec/           # Design specifications
    arch/           # ADRs, diagrams
    api/            # API documentation
    guide/          # Developer setup and other guides
  .env.example      # Environment variable template
  requirements.txt  # Python dependencies
  Dockerfile
  docker-compose.yml
  README.md
```

## Common Development Commands

### Running the Service Locally

From project root:
```bash
# Ensure dependencies are installed
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your Keycloak configuration

# Run the service (port 8001)
uvicorn app.main:app --reload --port 8001

# Access in browser
# Service: http://127.0.0.1:8001/
# API docs: http://127.0.0.1:8001/docs
# Health check: http://127.0.0.1:8001/health
```

### Running with Docker Compose

```bash
# Start Keycloak + med-z8
docker-compose up -d

# View logs
docker-compose logs -f med-z8

# Stop services
docker-compose down
```

### Testing

```bash
# Run all tests
pytest

# Run unit tests only
pytest tests/unit/

# Run integration tests (requires Keycloak)
pytest tests/integration/

# Run load tests
pytest tests/load/

# Run with coverage
pytest --cov=app --cov-report=html
```

## Environment Configuration

**Required Environment Variables:**
```bash
# Keycloak/OIDC Configuration
OIDC_ISSUER=http://localhost:8080/realms/va-healthcare
OIDC_AUDIENCE=med-z8
OIDC_JWKS_URI=http://localhost:8080/realms/va-healthcare/protocol/openid-connect/certs

# Service Configuration
SERVICE_PORT=8001
LOG_LEVEL=INFO
ENVIRONMENT=development

# Optional: Storage Backend
STORAGE_BACKEND=memory  # Options: memory, redis, postgres
# REDIS_URL=redis://localhost:6379/0
# DATABASE_URL=postgresql://user:pass@localhost:5432/med_z8
```

**Configuration File:**
- All configuration is centralized in `app/config.py`
- Uses python-dotenv for environment variable loading
- Validates required settings on startup
- Provides sensible defaults for development

## Architecture Highlights

### Authentication Flow

**JWT-Based Authentication:**
1. User authenticates with Keycloak (outside med-z8 scope)
2. Client application receives JWT access token
3. Client includes token in Authorization header: `Bearer <token>`
4. med-z8 validates JWT using Keycloak's public keys (JWKS)
5. Extracts user_id from JWT `sub` claim
6. Uses user_id for context isolation

**Key Code Pattern:**
```python
# app/auth/dependencies.py
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer
from app.auth.jwt_validator import validate_jwt

security = HTTPBearer()

async def get_current_user(token: str = Depends(security)) -> dict:
    """
    Validate JWT and extract user claims.

    Returns:
        dict: {
            "user_id": "uuid-from-sub-claim",
            "email": "user@va.gov",
            "name": "John Doe",
            ...
        }
    """
    try:
        claims = await validate_jwt(token.credentials)
        return {
            "user_id": claims["sub"],
            "email": claims.get("email"),
            "name": claims.get("name"),
        }
    except Exception as e:
        raise HTTPException(
            status_code=401,
            detail=f"Invalid authentication credentials: {str(e)}"
        )
```

**Why Stateless JWT?**
- No database dependency for authentication
- Horizontally scalable (no session affinity)
- Keycloak handles token lifecycle
- Clean separation of concerns (identity vs context)

### Context Storage

**In-Memory Storage (Default):**
- Thread-safe dictionary: `Dict[user_id, PatientContext]`
- Uses threading.Lock for concurrent access
- Suitable for single-instance deployments
- Fast, simple, no external dependencies

**Key Code Pattern:**
```python
# app/storage/in_memory.py
import threading
from typing import Optional
from app.models.context import PatientContext

class InMemoryStorage:
    def __init__(self):
        self._contexts: dict[str, PatientContext] = {}
        self._lock = threading.Lock()

    def get_context(self, user_id: str) -> Optional[PatientContext]:
        with self._lock:
            return self._contexts.get(user_id)

    def set_context(self, user_id: str, context: PatientContext) -> None:
        with self._lock:
            self._contexts[user_id] = context

    def clear_context(self, user_id: str) -> bool:
        with self._lock:
            if user_id in self._contexts:
                del self._contexts[user_id]
                return True
            return False
```

**Future Storage Backends:**
- **Redis:** Multi-instance deployments, shared context across pods
- **PostgreSQL:** Persistent storage, context survives restarts

### Per-User Context Isolation

**Critical Design Principle:**
Each user has their own independent patient context. User A selecting Patient 123 does NOT affect User B's context.

**Context Model:**
```python
# app/models/context.py
from pydantic import BaseModel, Field
from datetime import datetime

class PatientContext(BaseModel):
    user_id: str = Field(..., description="User ID from JWT sub claim")
    email: str = Field(..., description="User email from JWT")
    patient_id: str = Field(..., description="Active patient ICN or ID")
    set_by: str = Field(..., description="Application that set context (e.g., 'med-z1')")
    set_at: datetime = Field(default_factory=datetime.utcnow)
    last_accessed_at: datetime = Field(default_factory=datetime.utcnow)
```

**API Endpoint Pattern:**
```python
# app/routes/context.py
@router.get("/ccow/active-patient")
async def get_active_patient(
    user: dict = Depends(get_current_user),
    storage: Storage = Depends(get_storage)
) -> PatientContext:
    """Get the currently active patient for authenticated user."""
    context = storage.get_context(user["user_id"])
    if not context:
        raise HTTPException(status_code=404, detail="No active patient context")
    return context

@router.put("/ccow/active-patient")
async def set_active_patient(
    request: SetPatientRequest,
    user: dict = Depends(get_current_user),
    storage: Storage = Depends(get_storage)
) -> PatientContext:
    """Set the active patient for authenticated user."""
    context = PatientContext(
        user_id=user["user_id"],
        email=user["email"],
        patient_id=request.patient_id,
        set_by=request.set_by
    )
    storage.set_context(user["user_id"], context)
    return context
```

### API Design

**RESTful Endpoints:**
- `GET /ccow/active-patient` - Get user's current patient context
- `PUT /ccow/active-patient` - Set user's patient context
- `DELETE /ccow/active-patient` - Clear user's patient context
- `GET /ccow/history` - Get user's context change history
- `GET /ccow/active-patients` - Admin: Get all active contexts
- `POST /ccow/cleanup` - Admin: Clean up stale contexts
- `GET /health` - Health check endpoint
- `GET /docs` - OpenAPI/Swagger documentation

**Authentication Required:**
All `/ccow/*` endpoints require valid JWT in Authorization header.

**Error Responses:**
```json
// 401 Unauthorized - Invalid/missing JWT
{
  "detail": "Invalid authentication credentials: Token expired"
}

// 404 Not Found - No active patient
{
  "detail": "No active patient context"
}

// 500 Internal Server Error
{
  "detail": "Storage backend error"
}
```

## Important Implementation Notes

### JWT Validation

**JWKS Caching:**
- Keycloak public keys are cached to avoid excessive HTTP requests
- Cache TTL: 3600 seconds (1 hour)
- Cache invalidated on validation failure

**Key Code Pattern:**
```python
# app/auth/jwt_validator.py
import httpx
from jose import jwt, JWTError
from functools import lru_cache

@lru_cache(maxsize=1)
async def get_jwks():
    """Fetch and cache Keycloak JWKS."""
    async with httpx.AsyncClient() as client:
        response = await client.get(OIDC_JWKS_URI)
        response.raise_for_status()
        return response.json()

async def validate_jwt(token: str) -> dict:
    """
    Validate JWT using Keycloak public keys.

    Raises:
        HTTPException: If token is invalid, expired, or malformed
    """
    try:
        jwks = await get_jwks()
        # Find the key used to sign this token
        unverified_header = jwt.get_unverified_header(token)
        key = next(k for k in jwks["keys"] if k["kid"] == unverified_header["kid"])

        # Validate signature, expiration, audience, issuer
        claims = jwt.decode(
            token,
            key,
            algorithms=["RS256"],
            audience=OIDC_AUDIENCE,
            issuer=OIDC_ISSUER
        )
        return claims
    except JWTError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {str(e)}")
```

### Thread Safety

**Critical:** The in-memory storage MUST be thread-safe because FastAPI uses a thread pool for async operations.

**Always use locks when accessing shared state:**
```python
# CORRECT
with self._lock:
    self._contexts[user_id] = context

# INCORRECT - Race condition!
self._contexts[user_id] = context
```

### Context Cleanup

**Stale Context Detection:**
- Contexts not accessed for 24 hours are considered stale
- Admin endpoint `/ccow/cleanup` removes stale contexts
- Optional: Scheduled cleanup task (future enhancement)

### CORS Configuration

**Required for Browser-Based Clients:**
```python
# app/main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:8000",  # med-z1
        "http://localhost:8002",  # future med-z2
        # Add other client origins
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

## Client Integration Patterns

### Python/FastAPI Client (med-z1)

```python
# med-z1: app/utils/ccow_client.py
import httpx
from fastapi import Request

class CCOWClient:
    def __init__(self, base_url: str = "http://localhost:8001"):
        self.base_url = base_url

    async def set_active_patient(
        self,
        access_token: str,
        patient_id: str,
        set_by: str = "med-z1"
    ):
        """Set active patient context in med-z8."""
        async with httpx.AsyncClient() as client:
            response = await client.put(
                f"{self.base_url}/ccow/active-patient",
                json={"patient_id": patient_id, "set_by": set_by},
                headers={"Authorization": f"Bearer {access_token}"}
            )
            response.raise_for_status()
            return response.json()

    async def get_active_patient(self, access_token: str):
        """Get current active patient context."""
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/ccow/active-patient",
                headers={"Authorization": f"Bearer {access_token}"}
            )
            if response.status_code == 404:
                return None  # No active patient
            response.raise_for_status()
            return response.json()
```

### JavaScript/Browser Client

```javascript
// JavaScript client for browser-based applications
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
            body: JSON.stringify({ patient_id: patientId, set_by: setBy })
        });

        if (!response.ok) {
            throw new Error(`CCOW error: ${response.statusText}`);
        }

        return await response.json();
    }

    async getActivePatient(accessToken) {
        const response = await fetch(`${this.baseUrl}/ccow/active-patient`, {
            headers: { 'Authorization': `Bearer ${accessToken}` }
        });

        if (response.status === 404) {
            return null;  // No active patient
        }

        if (!response.ok) {
            throw new Error(`CCOW error: ${response.statusText}`);
        }

        return await response.json();
    }
}
```

## Testing Strategy

### Unit Tests

**Focus:** Individual components in isolation
**Location:** `tests/unit/`
**No External Dependencies:** Mock all HTTP calls, no real Keycloak

**Example:**
```python
# tests/unit/test_storage.py
from app.storage.in_memory import InMemoryStorage
from app.models.context import PatientContext

def test_set_and_get_context():
    storage = InMemoryStorage()

    context = PatientContext(
        user_id="user-123",
        email="test@va.gov",
        patient_id="ICN-456",
        set_by="test-app"
    )

    storage.set_context("user-123", context)
    retrieved = storage.get_context("user-123")

    assert retrieved.patient_id == "ICN-456"
    assert retrieved.user_id == "user-123"
```

### Integration Tests

**Focus:** End-to-end API flows with real Keycloak
**Location:** `tests/integration/`
**Requires:** Running Keycloak instance

**Example:**
```python
# tests/integration/test_context_api.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture
def valid_jwt(keycloak_client):
    """Fixture to get real JWT from test Keycloak."""
    return keycloak_client.get_access_token(
        username="test-user",
        password="test-password"
    )

def test_set_active_patient(valid_jwt):
    client = TestClient(app)

    response = client.put(
        "/ccow/active-patient",
        json={"patient_id": "ICN-123", "set_by": "test"},
        headers={"Authorization": f"Bearer {valid_jwt}"}
    )

    assert response.status_code == 200
    assert response.json()["patient_id"] == "ICN-123"
```

### Load Tests

**Focus:** Performance under concurrent load
**Location:** `tests/load/`
**Tools:** pytest-benchmark, locust

**Example:**
```python
# tests/load/test_concurrent_access.py
import concurrent.futures
from fastapi.testclient import TestClient

def test_concurrent_context_updates(valid_jwt):
    """Test 100 concurrent context updates."""
    client = TestClient(app)

    def update_context(i):
        return client.put(
            "/ccow/active-patient",
            json={"patient_id": f"ICN-{i}", "set_by": "load-test"},
            headers={"Authorization": f"Bearer {valid_jwt}"}
        )

    with concurrent.futures.ThreadPoolExecutor(max_workers=100) as executor:
        futures = [executor.submit(update_context, i) for i in range(100)]
        results = [f.result() for f in futures]

    assert all(r.status_code == 200 for r in results)
```

## Deployment

### Docker Deployment

**Dockerfile Best Practices:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ ./app/

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8001/health || exit 1

# Run application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

### Kubernetes Deployment

**Key Resources:**
- Deployment: 3 replicas for high availability
- Service: ClusterIP for internal access, LoadBalancer for external
- ConfigMap: Non-sensitive configuration
- Secret: Keycloak client credentials
- Ingress: HTTPS termination

**Example Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: med-z8
spec:
  replicas: 3
  selector:
    matchLabels:
      app: med-z8
  template:
    metadata:
      labels:
        app: med-z8
    spec:
      containers:
      - name: med-z8
        image: va-healthcare/med-z8:latest
        ports:
        - containerPort: 8001
        env:
        - name: OIDC_ISSUER
          valueFrom:
            configMapKeyRef:
              name: med-z8-config
              key: oidc_issuer
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
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
```

## Security Considerations

### JWT Validation

**Always Validate:**
- Signature (using Keycloak public key)
- Expiration (exp claim)
- Issuer (iss claim must match Keycloak)
- Audience (aud claim must be "med-z8")

**Never:**
- Trust tokens without validation
- Accept tokens from unknown issuers
- Use tokens past expiration
- Store tokens in logs

### Transport Security

**HTTPS Required in Production:**
- All production traffic must use TLS 1.2+
- Development can use HTTP for localhost only

### Rate Limiting (Future)

**Recommended Limits:**
- 100 requests per minute per user
- 1000 requests per minute per IP
- Prevents abuse and DoS attacks

## Performance Goals

**Target Metrics:**
- **JWT Validation:** < 50ms (p95)
- **Context Retrieval:** < 10ms (p95) for in-memory storage
- **Context Update:** < 20ms (p95) for in-memory storage
- **Concurrent Users:** 1000+ users with minimal latency increase

**Bottlenecks to Monitor:**
- JWKS HTTP fetching (mitigated by caching)
- Storage backend latency (Redis/PostgreSQL)
- Lock contention in high-concurrency scenarios

## Migration from med-z1 CCOW

**If migrating from embedded CCOW in med-z1:**

1. **Keep Old CCOW Running:** Feature flag for gradual cutover
2. **Add JWT Support to med-z1:** Implement Keycloak OIDC first
3. **Update CCOW Client:** Switch from session cookie to JWT bearer token
4. **Test Dual Mode:** Run both old and new CCOW simultaneously
5. **Cutover:** Flip feature flag, decommission old CCOW

**See:** `docs/spec/med-z1-keycloak-migration.md` for detailed migration plan

## Related Projects

**Keycloak:**
- Identity Provider (simulates Azure AD)
- Manages users, authentication, JWT issuance
- Port 8080 (default)

**med-z1:**
- Longitudinal health record viewer
- Primary client application for med-z8
- Port 8000

**Future med-* Applications:**
- med-z2: Orders management
- med-z3: Imaging viewer
- All will use med-z8 for patient context

## Reference Documentation

**Core Design Documents:**
- `docs/spec/med-z8-ccow-service-design.md` - Complete technical specification
- `docs/spec/med-z8-architecture-decisions.md` - ADRs explaining key decisions
- `docs/spec/med-z8-api-examples.md` - Request/response examples with real JWTs

**Related Specifications:**
- `docs/spec/med-z1-keycloak-migration.md` - How med-z1 integrates with med-z8
- `docs/spec/med-application-template.md` - Template for new med-* apps

**API Documentation:**
- OpenAPI/Swagger: http://localhost:8001/docs (when running)
- ReDoc: http://localhost:8001/redoc (when running)

## Development Workflow

### Adding a New Endpoint

1. **Define Pydantic Models** (`app/models/`)
2. **Create Route Handler** (`app/routes/`)
3. **Add to Router** (`app/main.py`)
4. **Write Unit Tests** (`tests/unit/`)
5. **Write Integration Tests** (`tests/integration/`)
6. **Update API Documentation** (`docs/api/`)

### Adding a New Storage Backend

1. **Implement BaseStorage Interface** (`app/storage/base.py`)
2. **Create Backend Class** (`app/storage/<backend>.py`)
3. **Add Configuration** (`app/config.py`)
4. **Write Tests** (`tests/unit/test_storage_<backend>.py`)
5. **Update Documentation**

### Common Pitfalls to Avoid

**1. Thread Safety:**
❌ Don't modify shared state without locks
✅ Always use `with self._lock:` for in-memory storage

**2. JWT Expiration:**
❌ Don't assume tokens are always valid
✅ Validate expiration on every request (FastAPI dependency handles this)

**3. User ID from JWT:**
❌ Don't trust user_id from request body
✅ Always extract user_id from validated JWT `sub` claim

**4. Error Handling:**
❌ Don't expose internal errors to clients
✅ Log detailed errors, return generic HTTP status codes

**5. CORS:**
❌ Don't use `allow_origins=["*"]` in production
✅ Whitelist specific client origins

## Troubleshooting

### "Invalid authentication credentials"

**Cause:** JWT validation failure

**Solutions:**
- Check JWT expiration (tokens expire after 5-15 minutes typically)
- Verify OIDC_ISSUER matches Keycloak realm
- Verify OIDC_AUDIENCE is "med-z8"
- Check Keycloak is running and accessible
- Verify JWKS endpoint is reachable

### "No active patient context" (404)

**Cause:** User has not selected a patient yet

**Expected Behavior:** This is normal if user hasn't called PUT /ccow/active-patient

**Client Should:**
- Check for 404 response
- Prompt user to select a patient
- Do NOT retry repeatedly

### High Latency

**Possible Causes:**
1. JWKS cache miss (first request after cache expiration)
2. Network latency to Keycloak
3. Lock contention (too many concurrent requests)

**Solutions:**
- Monitor JWKS cache hit rate
- Use Redis/PostgreSQL for distributed deployments
- Scale horizontally (add more replicas)

### Context Not Syncing Across Apps

**Possible Causes:**
1. Different user_id values (JWT sub claims don't match)
2. Different med-z8 instances (if using in-memory storage)
3. Token from different Keycloak realm

**Solutions:**
- Verify all apps use same Keycloak realm
- Check user_id in JWT (use /docs to inspect token)
- Use Redis backend for multi-instance deployments

## Best Practices

1. **Always use dependency injection** for storage and authentication
2. **Log user_id with every operation** for audit trail
3. **Use type hints** for all function parameters and return values
4. **Write tests first** for critical functionality
5. **Keep routes thin** - business logic belongs in separate modules
6. **Use Pydantic for validation** - never trust client input
7. **Monitor JWT validation failures** - may indicate attack
8. **Cache JWKS** - but invalidate on validation failure
9. **Use structured logging** - JSON format for production
10. **Document breaking changes** - this is a shared service

## Contributing

See `docs/CONTRIBUTING.md` for:
- Code style guidelines (ruff, black, mypy)
- PR review process
- Testing requirements
- Documentation standards

## Getting Help

**Project Issues:** https://github.com/va-healthcare/med-z8/issues
**Design Questions:** See `docs/spec/med-z8-architecture-decisions.md`
**API Reference:** http://localhost:8001/docs

**Related Projects:**
- med-z1: https://github.com/va-healthcare/med-z1
- Keycloak: https://www.keycloak.org/documentation

---

**Document Version:** v1.0
**Last Updated:** 2026-01-08
**Author:** VA Healthcare Engineering Team
