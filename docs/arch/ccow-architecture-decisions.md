# med-z8 Architecture Decision Records (ADRs)

This document captures the key architectural decisions made during the design and implementation of med-z8, the standalone CCOW service for VA healthcare applications.

**Purpose:** Document WHY decisions were made, not just WHAT was decided. This provides critical context for future developers and prevents re-litigating settled decisions.

**Format:** Each ADR follows a simple structure:
- **Decision:** What was decided
- **Context:** Why this decision was needed
- **Options Considered:** Alternatives that were evaluated
- **Decision Rationale:** Why the chosen option was selected
- **Consequences:** Trade-offs and implications
- **Date:** When the decision was made

---

## ADR-001: Standalone Service vs. Embedded Library

**Decision:** Build med-z8 as a standalone microservice, not a library embedded in each application.

**Context:**
The original med-z1 CCOW implementation was embedded within the med-z1 codebase. As the VA healthcare ecosystem grows to include multiple applications (med-z1, CPRS, imaging, etc.), each application would need its own CCOW implementation or shared library. Patient context synchronization requires a single source of truth across all applications.

**Options Considered:**

1. **Standalone Microservice (CHOSEN)**
   - Independent FastAPI service
   - RESTful API for context management
   - Single deployment, shared by all applications

2. **Shared Python Library**
   - PyPI package imported by each application
   - Each app maintains own context storage
   - Context sync via message queue or database

3. **Embedded per Application**
   - Copy/paste CCOW code into each app
   - No dependency on external service
   - Each app owns its CCOW instance

**Decision Rationale:**

**Why Standalone Microservice:**
- ✅ **Single Source of Truth:** All applications query same service, guaranteed consistency
- ✅ **Technology Agnostic:** REST API works with Python, JavaScript, Java, C# clients
- ✅ **Independent Scaling:** CCOW can scale separately from applications
- ✅ **Centralized Monitoring:** One place to observe all context changes
- ✅ **Simplified Deployment:** Update CCOW without redeploying all apps

**Why NOT Shared Library:**
- ❌ Each app would need separate context storage (inconsistency risk)
- ❌ Requires message queue for real-time sync (added complexity)
- ❌ Language-specific (Python-only)
- ❌ Version conflicts (App A on v1.0, App B on v1.2)

**Why NOT Embedded:**
- ❌ Code duplication across applications
- ❌ No context synchronization between apps
- ❌ Update nightmare (change required in all apps)

**Consequences:**
- ✅ Applications depend on med-z8 availability (requires monitoring)
- ✅ Network latency for context operations (mitigated by fast in-memory storage)
- ✅ Requires container orchestration for production (Kubernetes/Docker Compose)
- ✅ Single point of failure (mitigated by HA deployment with multiple replicas)

**Date:** 2026-01-07

---

## ADR-002: JWT Authentication vs. Session-Based Authentication

**Decision:** Use JWT bearer token authentication with Keycloak, not session-based authentication with database.

**Context:**
The original med-z1 CCOW implementation used session-based authentication, validating session cookies against a PostgreSQL database. This approach tightly coupled CCOW to med-z1's authentication system. For med-z8 to be a shared service across multiple applications (each with potentially different authentication), a standard-based approach was needed.

**Options Considered:**

1. **JWT Bearer Token Authentication with Keycloak (CHOSEN)**
   - OAuth 2.0 / OpenID Connect standard
   - Stateless token validation
   - Keycloak as identity provider

2. **Session-Based Authentication per Application**
   - Each app provides session cookie
   - med-z8 queries each app's auth database
   - Requires database credentials for all apps

3. **API Key Authentication**
   - Static API keys per application
   - No user-level authentication
   - Shared key across all users of an app

4. **Custom Token Service**
   - Build proprietary token system
   - Custom validation logic
   - No third-party dependencies

**Decision Rationale:**

**Why JWT + Keycloak:**
- ✅ **Industry Standard:** OAuth 2.0 / OIDC widely adopted
- ✅ **Stateless:** No database dependency for authentication
- ✅ **Single Sign-On:** Users authenticate once, use all apps
- ✅ **Horizontally Scalable:** No session affinity required
- ✅ **Rich Claims:** JWT contains user_id, email, roles
- ✅ **Production Pattern:** VA uses Azure AD (similar to Keycloak)
- ✅ **Third-Party Support:** Keycloak is battle-tested, enterprise-grade

**Why NOT Session-Based:**
- ❌ Database dependency (coupling)
- ❌ Requires session DB credentials for each app
- ❌ Session affinity prevents horizontal scaling
- ❌ Custom per-application integration

**Why NOT API Keys:**
- ❌ No user-level identity (cannot track which user set context)
- ❌ Key rotation challenges
- ❌ Shared secrets (security risk)

**Why NOT Custom Token Service:**
- ❌ Reinventing the wheel
- ❌ Security vulnerabilities from homegrown crypto
- ❌ No SSO support
- ❌ Maintenance burden

**Consequences:**
- ✅ Applications must integrate with Keycloak (one-time setup)
- ✅ Tokens expire (5-15 minutes typical, requires refresh flow)
- ✅ Clock synchronization required (JWT exp validation)
- ✅ Network dependency on Keycloak for JWKS endpoint
- ✅ JWKS caching required for performance (implemented with 1-hour TTL)

**Date:** 2026-01-07

---

## ADR-003: In-Memory Storage as Default Backend

**Decision:** Use thread-safe in-memory dictionary as default storage backend, with optional Redis/PostgreSQL for production.

**Context:**
Patient context data needs to be stored and retrieved quickly. The storage backend choice impacts latency, scalability, and deployment complexity. Context data is relatively small (user_id → patient_id mappings) and can afford to be ephemeral in some scenarios.

**Options Considered:**

1. **In-Memory Dictionary (CHOSEN for default)**
   - Python dict with threading.Lock
   - No external dependencies
   - Fastest possible retrieval

2. **Redis (Optional, recommended for production)**
   - Key-value store
   - Distributed across multiple instances
   - Persistence options available

3. **PostgreSQL (Optional, future)**
   - Relational database
   - ACID guarantees
   - Full audit trail

4. **SQLite**
   - File-based database
   - No external service
   - Limited concurrency

**Decision Rationale:**

**Why In-Memory as Default:**
- ✅ **Zero Configuration:** No setup required for development
- ✅ **Fastest Latency:** < 10ms for get/set operations
- ✅ **Simple Deployment:** Single container, no dependencies
- ✅ **Sufficient for Development:** Mock environment, low user count
- ✅ **Clear Upgrade Path:** Swap to Redis for production via config

**Why Redis for Production:**
- ✅ **Distributed:** Context shared across multiple med-z8 replicas
- ✅ **Persistence:** Context survives pod restarts (if enabled)
- ✅ **Battle-Tested:** Industry standard for session storage
- ✅ **Fast:** < 50ms latency for remote operations

**Why PostgreSQL (Future):**
- ✅ **Full Audit Trail:** Track all context changes with timestamps
- ✅ **Advanced Queries:** Analytics on context usage patterns
- ✅ **ACID Guarantees:** Strong consistency

**Why NOT SQLite:**
- ❌ File locking issues with concurrent writes
- ❌ Poor performance under high concurrency
- ❌ Not suitable for distributed deployments

**Consequences:**
- ✅ Development: No external dependencies, instant startup
- ✅ Single-instance production: In-memory is acceptable
- ✅ Multi-instance production: Must use Redis (configured via env var)
- ✅ Context lost on restart (acceptable for mock environment)
- ✅ Future: Can add PostgreSQL without API changes

**Configuration:**
```bash
# Development (default)
STORAGE_BACKEND=memory

# Production (multi-instance)
STORAGE_BACKEND=redis
REDIS_URL=redis://redis-cluster:6379/0

# Future (full audit trail)
STORAGE_BACKEND=postgres
DATABASE_URL=postgresql://user:pass@db:5432/med_z8
```

**Date:** 2026-01-07

---

## ADR-004: Per-User Context Isolation vs. Global Shared Context

**Decision:** Implement per-user context isolation. Each user has independent patient context.

**Context:**
The original med-z1 CCOW v1.0 used a single global context (one active patient for all users). This was identified as a critical flaw - when User A selects Patient 123, User B's screen would also switch to Patient 123. For a shared service supporting multiple concurrent users, proper isolation is essential.

**Options Considered:**

1. **Per-User Context Isolation (CHOSEN)**
   - Storage keyed by user_id
   - Each user's context is independent
   - User A selecting Patient 123 does NOT affect User B

2. **Global Shared Context**
   - Single active patient for entire system
   - All users see same patient
   - Mimics physical context board in clinic

3. **Per-Session Context Isolation**
   - Storage keyed by session_id
   - Context lost on logout
   - New session = new context

4. **Per-Application Context Isolation**
   - Storage keyed by application name
   - All users of med-z1 see same patient
   - med-z2 users see different patient

**Decision Rationale:**

**Why Per-User Isolation:**
- ✅ **Privacy:** User A cannot see User B's active patient
- ✅ **Safety:** Prevents unintended patient switches
- ✅ **Workflow Support:** Users can work on different patients simultaneously
- ✅ **Production Requirement:** VA security standards require user-level isolation
- ✅ **Context Persistence:** User can logout/login and resume same patient

**Why NOT Global Shared:**
- ❌ **Critical Safety Issue:** User A's selection affects User B
- ❌ **Privacy Violation:** All users see same patient
- ❌ **Unusable in Production:** Violates HIPAA/VA security requirements

**Why NOT Per-Session:**
- ❌ Context lost on logout (poor UX)
- ❌ Requires re-selecting patient after every login
- ❌ Doesn't leverage SSO benefits

**Why NOT Per-Application:**
- ❌ All users of same app see same patient (same issues as global)
- ❌ Doesn't solve the isolation problem

**Consequences:**
- ✅ Storage keys: `Dict[user_id, PatientContext]`
- ✅ User identified by JWT `sub` claim (immutable)
- ✅ Context persists across logout/login (same user_id)
- ✅ Admin endpoint `/ccow/active-patients` returns all user contexts
- ✅ History endpoint `/ccow/history` can filter by user_id

**Security Note:**
User identity MUST come from validated JWT `sub` claim, NEVER from client request body. This prevents User A from impersonating User B.

**Date:** 2026-01-07

---

## ADR-005: Simplified CCOW REST API vs. Full HL7 CCOW Standard

**Decision:** Implement simplified REST API subset of CCOW, not full HL7 CCOW standard.

**Context:**
HL7 CCOW (Clinical Context Object Workgroup) is a comprehensive standard for context synchronization, including transaction model, secure binding, visual notification, and complex protocol. Full CCOW compliance requires significant implementation effort and provides features beyond VA healthcare's current needs.

**Options Considered:**

1. **Simplified REST API (CHOSEN)**
   - Core functionality: get/set/clear patient context
   - RESTful endpoints (GET, PUT, DELETE)
   - Simple JSON request/response
   - No full CCOW protocol

2. **Full HL7 CCOW Compliance**
   - Transaction model (context change proposals)
   - Secure binding protocol
   - Visual notification protocol
   - Survey protocol (app participation in context changes)

3. **Custom Protocol**
   - Proprietary context management
   - WebSocket for real-time push
   - GraphQL API

**Decision Rationale:**

**Why Simplified REST API:**
- ✅ **Development Speed:** 6 weeks vs. 6+ months for full CCOW
- ✅ **Sufficient for Requirements:** Med-z1 only needs get/set/clear
- ✅ **Easy Integration:** REST is universally understood
- ✅ **Testable:** Simple HTTP requests, no complex protocol
- ✅ **Maintainable:** Straightforward code, minimal abstractions
- ✅ **Future Extensible:** Can add full CCOW later if needed

**Why NOT Full CCOW:**
- ❌ **Over-Engineering:** VA healthcare doesn't need transaction model yet
- ❌ **Complexity:** Secure binding, visual notification, survey protocol
- ❌ **Development Time:** 6+ months for full compliance
- ❌ **Library Availability:** Limited Python CCOW libraries
- ❌ **Testing Burden:** Complex protocol requires extensive test infrastructure

**Why NOT Custom Protocol:**
- ❌ **Standards Exist:** CCOW already solves this problem
- ❌ **Maintenance Burden:** Proprietary protocols harder to document/support
- ❌ **Learning Curve:** New developers must learn custom system

**Current API:**
```
GET    /ccow/active-patient        # Get user's active patient
PUT    /ccow/active-patient        # Set user's active patient
DELETE /ccow/active-patient        # Clear user's active patient
GET    /ccow/history               # Get context change history
GET    /ccow/active-patients       # Admin: All active contexts
POST   /ccow/cleanup               # Admin: Clean up stale contexts
```

**Future Enhancements (if needed):**
- Transaction model: Propose context change, apps can veto
- Visual notification: Apps display "context changed by User A"
- Survey: Apps participate in context change decisions
- Real-time push: WebSocket for instant updates (vs. polling)

**Consequences:**
- ✅ Not "CCOW compliant" in formal sense (simplified subset)
- ✅ Documentation calls this "CCOW-like" or "Simplified CCOW"
- ✅ Can upgrade to full CCOW in future without breaking changes (add endpoints)
- ✅ Clients poll for context changes (future: add WebSocket)

**Date:** 2026-01-07

---

## ADR-006: FastAPI vs. Flask vs. Django

**Decision:** Use FastAPI as web framework for med-z8.

**Context:**
med-z8 needs a Python web framework for building the REST API. The choice impacts performance, developer experience, and ecosystem compatibility. med-z1 already uses FastAPI, providing team familiarity.

**Options Considered:**

1. **FastAPI (CHOSEN)**
   - Modern async framework
   - Automatic OpenAPI/Swagger docs
   - Pydantic for request/response validation

2. **Flask**
   - Lightweight, minimal framework
   - Large ecosystem
   - Synchronous by default

3. **Django + Django REST Framework**
   - Full-stack framework
   - ORM, admin panel, auth built-in
   - Heavyweight for microservice

**Decision Rationale:**

**Why FastAPI:**
- ✅ **Async Support:** Native async/await for concurrent requests
- ✅ **Auto Documentation:** OpenAPI/Swagger generated automatically
- ✅ **Type Safety:** Pydantic validation catches errors at runtime
- ✅ **Performance:** ~3x faster than Flask in benchmarks
- ✅ **Team Familiarity:** med-z1 already uses FastAPI
- ✅ **Dependency Injection:** Built-in DI system for testability
- ✅ **Modern Python:** Leverages Python 3.11 type hints

**Why NOT Flask:**
- ❌ **Synchronous:** Would need Flask-Async extension
- ❌ **Manual Validation:** Requires marshmallow or manual checks
- ❌ **No Auto Docs:** Would need Flask-RESTPlus or Flasgger
- ❌ **Older Design:** Pre-dates modern Python async patterns

**Why NOT Django:**
- ❌ **Overkill:** ORM, admin panel, templates not needed
- ❌ **Heavyweight:** Slower startup, larger footprint
- ❌ **Opinionated:** Django way vs. simple microservice
- ❌ **Learning Curve:** Larger framework to learn

**Consequences:**
- ✅ Excellent developer experience (auto docs, type safety)
- ✅ Fast performance (async, Pydantic)
- ✅ Consistent with med-z1 (same framework)
- ✅ Easy testing (TestClient, dependency overrides)
- ✅ Requires Python 3.11+ (uses modern type hints)

**Date:** 2026-01-07

---

## ADR-007: Docker Compose for Development, Kubernetes for Production

**Decision:** Use Docker Compose for local development, Kubernetes for production deployment.

**Context:**
med-z8 requires orchestration of multiple services (Keycloak, med-z8, optionally Redis). Development environment should be easy to start/stop, while production requires high availability, scaling, and monitoring.

**Options Considered:**

1. **Docker Compose (Development) + Kubernetes (Production) (CHOSEN)**
   - Simple local setup: `docker-compose up`
   - Production: HA, scaling, monitoring

2. **Kubernetes for Everything**
   - Minikube for local development
   - Same config dev and prod

3. **Docker Compose for Everything**
   - Simple everywhere
   - No Kubernetes

4. **Bare Metal / VMs**
   - No containers
   - Manual process management

**Decision Rationale:**

**Why Docker Compose for Development:**
- ✅ **Simple:** One command starts everything
- ✅ **Fast:** Instant start/stop/restart
- ✅ **Portable:** Works on Mac, Linux, Windows
- ✅ **Low Resource:** No Kubernetes overhead locally

**Why Kubernetes for Production:**
- ✅ **High Availability:** Multiple replicas, auto-recovery
- ✅ **Horizontal Scaling:** Add replicas under load
- ✅ **Service Discovery:** Built-in DNS for services
- ✅ **Rolling Updates:** Zero-downtime deployments
- ✅ **Monitoring:** Prometheus, Grafana integration
- ✅ **VA Standard:** VA infrastructure uses Kubernetes

**Why NOT Kubernetes for Development:**
- ❌ **Complexity:** Minikube setup, kubectl commands
- ❌ **Resource Usage:** K8s control plane overhead
- ❌ **Slower Iteration:** Build → tag → apply vs. docker-compose up

**Why NOT Docker Compose for Production:**
- ❌ **No HA:** Single host, no auto-recovery
- ❌ **No Auto-Scaling:** Manual replica management
- ❌ **Limited Monitoring:** No built-in observability

**Consequences:**
- ✅ Development: `docker-compose up -d` (Keycloak + med-z8)
- ✅ Production: Kubernetes manifests (Deployment, Service, Ingress)
- ✅ CI/CD: Build Docker image → push to registry → deploy to K8s
- ✅ Configuration parity: Same env vars dev and prod

**Configuration:**
```yaml
# docker-compose.yml (development)
version: '3.8'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    ports: ["8080:8080"]
  med-z8:
    build: .
    ports: ["8001:8001"]
    depends_on: [keycloak]

# k8s/deployment.yml (production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: med-z8
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: med-z8
        image: va-healthcare/med-z8:v1.0.0
```

**Date:** 2026-01-07

---

## ADR-008: user_id from JWT `sub` Claim as Identity Key

**Decision:** Use JWT `sub` (subject) claim as immutable user_id for context storage keys.

**Context:**
Patient context must be keyed by a stable, immutable user identifier. The key must be trustworthy (cannot be spoofed by client) and consistent across sessions. JWT tokens contain multiple claims that could identify users (sub, email, preferred_username).

**Options Considered:**

1. **JWT `sub` Claim (CHOSEN)**
   - Immutable user identifier (UUID)
   - Guaranteed present in OIDC tokens
   - Never changes for a user

2. **JWT `email` Claim**
   - Human-readable
   - Can change (user updates email)
   - May not be unique in all systems

3. **JWT `preferred_username` Claim**
   - Human-readable
   - Can change (user renames account)
   - May not be present

4. **Custom User ID from Request Body**
   - Client sends user_id in JSON
   - Vulnerable to spoofing
   - No validation

**Decision Rationale:**

**Why JWT `sub` Claim:**
- ✅ **Immutable:** Never changes for a user (UUID or unique ID)
- ✅ **Trustworthy:** Extracted from cryptographically validated JWT
- ✅ **Standard:** OIDC spec requires `sub` claim in all tokens
- ✅ **Non-Spoofable:** Client cannot forge JWT signature
- ✅ **Consistent:** Same `sub` across all applications using Keycloak

**Why NOT `email`:**
- ❌ **Mutable:** User can change email address
- ❌ **Context Loss:** Changing email = new user_id = lost context
- ❌ **Optional:** Email claim may not be present in all tokens

**Why NOT `preferred_username`:**
- ❌ **Mutable:** User can change username
- ❌ **Optional:** May not be present in all OIDC tokens
- ❌ **Not Guaranteed Unique:** Some systems allow duplicate usernames

**Why NOT Request Body:**
- ❌ **Security Violation:** Client can impersonate any user
- ❌ **No Validation:** Cannot verify user_id authenticity

**Implementation:**
```python
# app/auth/dependencies.py
async def get_current_user(token: str = Depends(security)) -> dict:
    claims = await validate_jwt(token.credentials)
    return {
        "user_id": claims["sub"],  # ✅ IMMUTABLE, TRUSTWORTHY
        "email": claims.get("email"),
        "name": claims.get("name"),
    }

# app/routes/context.py
@router.put("/ccow/active-patient")
async def set_active_patient(
    request: SetPatientRequest,
    user: dict = Depends(get_current_user)  # ✅ From validated JWT
):
    context = PatientContext(
        user_id=user["user_id"],  # ✅ From JWT sub claim
        # ...
    )
    storage.set_context(user["user_id"], context)
```

**Security Note:**
NEVER accept user_id from client request body or query parameters. Always extract from validated JWT.

**Consequences:**
- ✅ User context persists across email changes
- ✅ User context persists across username changes
- ✅ User context persists across logout/login (same sub)
- ✅ Impossible for User A to access User B's context
- ✅ Admin endpoints can map sub → email for human-readable display

**Date:** 2026-01-07

---

## ADR-009: JWKS Caching Strategy

**Decision:** Cache Keycloak JWKS (public keys) for 1 hour, invalidate on validation failure.

**Context:**
JWT validation requires fetching Keycloak's public keys (JWKS endpoint) to verify token signatures. Fetching JWKS on every request would add ~50-100ms latency and create excessive load on Keycloak. However, Keycloak may rotate keys periodically, so cache must be refreshed.

**Options Considered:**

1. **Cache with 1-hour TTL + Invalidate on Failure (CHOSEN)**
   - Cache JWKS for 3600 seconds
   - On validation failure, invalidate cache and retry once
   - Balances performance and freshness

2. **No Caching**
   - Fetch JWKS on every request
   - Always fresh keys
   - High latency

3. **Cache Forever**
   - Fetch once, never refresh
   - Breaks when Keycloak rotates keys
   - Minimal latency

4. **Background Refresh**
   - Fetch JWKS every N minutes in background
   - Always warm cache
   - Requires background task

**Decision Rationale:**

**Why 1-hour TTL + Invalidate on Failure:**
- ✅ **Performance:** ~0ms latency after initial fetch (cache hit)
- ✅ **Handles Key Rotation:** Invalidate on failure catches rotated keys
- ✅ **Simple:** Python functools.lru_cache handles caching
- ✅ **Resilient:** Retry logic handles transient failures
- ✅ **Low Keycloak Load:** Max 1 request per hour per med-z8 instance

**Why NOT No Caching:**
- ❌ **Latency:** +50-100ms per request
- ❌ **Keycloak Load:** 1000 req/min = 1000 JWKS fetches/min

**Why NOT Cache Forever:**
- ❌ **Key Rotation:** Breaks when Keycloak rotates signing keys
- ❌ **Security Risk:** Compromised key cannot be revoked

**Why NOT Background Refresh:**
- ❌ **Complexity:** Requires background task management
- ❌ **Unnecessary:** Invalidate-on-failure is simpler and sufficient

**Implementation:**
```python
# app/auth/jwt_validator.py
import httpx
from functools import lru_cache
from datetime import datetime, timedelta

_jwks_cache_expiry = None

@lru_cache(maxsize=1)
async def get_jwks():
    """Fetch and cache Keycloak JWKS for 1 hour."""
    global _jwks_cache_expiry

    # Check if cache expired
    if _jwks_cache_expiry and datetime.utcnow() > _jwks_cache_expiry:
        get_jwks.cache_clear()

    async with httpx.AsyncClient() as client:
        response = await client.get(OIDC_JWKS_URI, timeout=5.0)
        response.raise_for_status()

    # Set cache expiry to 1 hour from now
    _jwks_cache_expiry = datetime.utcnow() + timedelta(hours=1)
    return response.json()

async def validate_jwt(token: str) -> dict:
    try:
        jwks = await get_jwks()
        claims = jwt.decode(token, jwks, ...)
        return claims
    except JWTError as e:
        # Validation failed, possibly due to key rotation
        get_jwks.cache_clear()  # ✅ Invalidate cache

        # Retry once with fresh keys
        jwks = await get_jwks()
        claims = jwt.decode(token, jwks, ...)
        return claims
```

**Cache Hit Rate:**
- Expected: >99% (tokens validated within 1-hour window)
- Misses: Initial request, after cache expiry, after invalidation

**Consequences:**
- ✅ First request: ~50-100ms (JWKS fetch + validation)
- ✅ Subsequent requests: <5ms (cache hit)
- ✅ After key rotation: One failure → cache invalidate → success
- ✅ Keycloak load: 1 request per hour per instance (minimal)
- ⚠️ Brief downtime if Keycloak unreachable (5s timeout)

**Date:** 2026-01-07

---

## ADR-010: History Tracking with 100-Entry Limit

**Decision:** Track last 100 context change events in memory, user-scoped filtering available.

**Context:**
Debugging and auditing require visibility into context changes (who set what patient when). Unlimited history would consume unbounded memory. Production audit trail requires database (future), but basic history helps with development/troubleshooting.

**Options Considered:**

1. **In-Memory Circular Buffer (100 entries) (CHOSEN)**
   - Store last 100 events
   - Oldest events discarded
   - Per-user filtering available

2. **No History Tracking**
   - Only current context stored
   - No audit trail
   - Cannot debug context issues

3. **Unlimited In-Memory History**
   - Store all events forever
   - Memory leak risk
   - OOM after months of operation

4. **Database-Backed History**
   - PostgreSQL for all events
   - Permanent audit trail
   - Requires database setup

**Decision Rationale:**

**Why 100-Entry Circular Buffer:**
- ✅ **Sufficient for Debugging:** Last 100 events covers typical session
- ✅ **Bounded Memory:** Fixed size, no leak risk
- ✅ **No Dependencies:** Works with in-memory storage
- ✅ **Fast:** No I/O for history queries
- ✅ **Simple:** Python collections.deque handles rotation

**Why NOT No History:**
- ❌ **No Debugging:** Cannot see who changed context when
- ❌ **No Audit Trail:** Security/compliance requirement (future)

**Why NOT Unlimited History:**
- ❌ **Memory Leak:** Unbounded growth
- ❌ **OOM Risk:** Crashes after months

**Why NOT Database (Yet):**
- ✅ **Future Enhancement:** Will add when persistence backend implemented
- ❌ **Not Required:** Development doesn't need permanent audit trail
- ❌ **Added Complexity:** Database setup for every dev environment

**Implementation:**
```python
# app/storage/in_memory.py
from collections import deque
from app.models.context import ContextHistoryEntry

class InMemoryStorage:
    def __init__(self, max_history: int = 100):
        self._history: deque[ContextHistoryEntry] = deque(maxlen=max_history)
        # maxlen=100 means deque automatically discards oldest when full

    def add_history(self, entry: ContextHistoryEntry):
        self._history.append(entry)  # ✅ Auto-rotation

    def get_history(self, user_id: Optional[str] = None) -> List[ContextHistoryEntry]:
        if user_id:
            return [e for e in self._history if e.user_id == user_id]
        return list(self._history)
```

**History Entry Model:**
```python
class ContextHistoryEntry(BaseModel):
    action: str  # "set" | "clear" | "access"
    user_id: str
    email: str
    patient_id: Optional[str]
    actor: str  # Application that triggered action
    timestamp: datetime
```

**API Endpoints:**
```
GET /ccow/history              # All users (last 100 events)
GET /ccow/history?user_id=123  # Specific user only
```

**Consequences:**
- ✅ Development: Last 100 events visible for debugging
- ✅ Memory: Fixed size (~10 KB for 100 entries)
- ⚠️ History lost on restart (acceptable for development)
- 🔮 Future: Migrate to PostgreSQL for permanent audit trail

**Date:** 2026-01-07

---

## ADR-011: Health Check Endpoint with Detailed Status

**Decision:** Implement `/health` endpoint with detailed status checks for dependencies.

**Context:**
Container orchestration (Kubernetes, Docker Compose) requires health checks for liveness and readiness probes. Simple "200 OK" doesn't distinguish between "service starting" vs. "service ready" vs. "service degraded."

**Options Considered:**

1. **Detailed Health Check (CHOSEN)**
   - Returns service status + dependency status
   - 200 OK if healthy, 503 if degraded
   - JSON response with details

2. **Simple 200 OK**
   - Always returns 200 if process running
   - No dependency checks
   - Minimal information

3. **No Health Endpoint**
   - Rely on TCP port check
   - No application-level status

**Decision Rationale:**

**Why Detailed Health Check:**
- ✅ **Kubernetes Ready:** Distinguishes startup vs. ready vs. degraded
- ✅ **Dependency Awareness:** Checks Keycloak JWKS reachability
- ✅ **Debugging:** JSON response shows what's failing
- ✅ **Monitoring:** Prometheus can scrape health metrics

**Why NOT Simple 200:**
- ❌ **No Dependency Checks:** Service may be up but Keycloak down
- ❌ **No Context:** Cannot distinguish between startup and ready

**Why NOT No Endpoint:**
- ❌ **TCP Check Insufficient:** Port open ≠ service ready
- ❌ **No Visibility:** Cannot debug health issues

**Implementation:**
```python
# app/routes/health.py
@router.get("/health")
async def health_check():
    """
    Health check endpoint for container orchestration.

    Returns:
        200 OK: Service healthy and ready
        503 Service Unavailable: Service degraded
    """
    status = {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "version": "1.0.0",
        "checks": {}
    }

    # Check Keycloak JWKS reachability
    try:
        jwks = await get_jwks()
        status["checks"]["keycloak_jwks"] = "healthy"
    except Exception as e:
        status["status"] = "degraded"
        status["checks"]["keycloak_jwks"] = f"unhealthy: {str(e)}"

    # Check storage backend
    try:
        storage.get_context("health-check-test")
        status["checks"]["storage"] = "healthy"
    except Exception as e:
        status["status"] = "degraded"
        status["checks"]["storage"] = f"unhealthy: {str(e)}"

    http_status = 200 if status["status"] == "healthy" else 503
    return JSONResponse(content=status, status_code=http_status)
```

**Response Examples:**
```json
// Healthy
{
  "status": "healthy",
  "timestamp": "2026-01-08T12:34:56.789Z",
  "version": "1.0.0",
  "checks": {
    "keycloak_jwks": "healthy",
    "storage": "healthy"
  }
}

// Degraded (Keycloak unreachable)
{
  "status": "degraded",
  "timestamp": "2026-01-08T12:34:56.789Z",
  "version": "1.0.0",
  "checks": {
    "keycloak_jwks": "unhealthy: Connection timeout",
    "storage": "healthy"
  }
}
```

**Kubernetes Integration:**
```yaml
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

**Consequences:**
- ✅ Kubernetes knows when service is ready to receive traffic
- ✅ Kubernetes restarts pod if health check fails (liveness)
- ✅ Operators can debug issues via health endpoint JSON
- ✅ Monitoring can alert on degraded status

**Date:** 2026-01-07

---

## Summary of Key Decisions

| ADR | Decision | Rationale |
|-----|----------|-----------|
| 001 | Standalone microservice | Single source of truth, technology agnostic |
| 002 | JWT + Keycloak authentication | Stateless, standards-based, SSO support |
| 003 | In-memory default storage | Zero config dev, Redis for production |
| 004 | Per-user context isolation | Privacy, safety, production requirement |
| 005 | Simplified REST API | Sufficient features, 6 weeks vs. 6 months |
| 006 | FastAPI framework | Async, auto docs, type safety, team familiarity |
| 007 | Docker Compose dev, K8s prod | Simple dev, HA production |
| 008 | JWT `sub` as user_id | Immutable, trustworthy, consistent |
| 009 | JWKS caching (1 hour) | Performance, handles key rotation |
| 010 | 100-entry history buffer | Debugging support, bounded memory |
| 011 | Detailed health checks | Kubernetes ready, dependency awareness |

---

**Document Version:** v1.0
**Last Updated:** 2026-01-08
**Author:** VA Healthcare Engineering Team
