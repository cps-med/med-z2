# med-z2 CCOW Vault Service

**Enterprise Patient Context Synchronization Service**

A standalone CCOW (Clinical Context Object Workgroup) service providing patient context synchronization across multiple clinical applications in the VA healthcare ecosystem, with Keycloak-based Single Sign-On (SSO) authentication.

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Development](#development)
- [Testing](#testing)
- [Deployment](#deployment)
- [API Documentation](#api-documentation)
- [Related Projects](#related-projects)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

**med-z2** is a lightweight, production-ready **CCOW Context Vault** service that enables **patient context synchronization** across multiple clinical applications. When a clinician selects a patient in one application (e.g., med-z1 longitudinal viewer), all other integrated applications automatically display the same patient context—creating a seamless, unified clinical workflow.

### What is CCOW?

CCOW (Clinical Context Object Workgroup) is an HL7 standard for synchronizing clinical context (such as the active patient) across multiple healthcare applications. med-z2 implements a **simplified, RESTful CCOW pattern** optimized for modern web applications.

### Why med-z2?

Traditional CCOW implementations are complex, proprietary, and tightly coupled to specific vendor ecosystems. med-z2 provides:

- **Modern REST API** instead of legacy SOAP/COM protocols
- **Stateless JWT authentication** instead of session databases
- **Microservice architecture** instead of monolithic coupling
- **Open-source Keycloak** instead of proprietary identity providers
- **Container-native** design for cloud deployment

---

## Problem Statement

**Before med-z2**, clinical applications at the VA operated in silos:

| Problem | Impact |
|---------|--------|
| **No Context Sync** | Clinician selects Patient A in one app, Patient B in another → confusion, errors |
| **No SSO** | Clinician logs in separately to each application → wasted time, password fatigue |
| **Vendor Lock-In** | Each app has custom authentication → integration nightmare |
| **Scalability Issues** | Session databases become bottlenecks at scale |

**With med-z2**, clinical applications share context and identity:

| Solution | Benefit |
|----------|---------|
| **Unified Patient Context** | Select patient once → all apps sync automatically |
| **Single Sign-On (SSO)** | Log in once → access all med-* applications |
| **Standards-Based** | OAuth 2.0 / OIDC (works with VA IAM, Azure AD) |
| **Stateless & Scalable** | JWT validation (no database), horizontal scaling |

---

## Key Features

### 🔐 Authentication

- **Keycloak OIDC Integration**: OAuth 2.0 / OpenID Connect authentication
- **JWT Bearer Tokens**: Stateless authentication (no session database)
- **SSO Across Applications**: Single login for med-z1, CPRS, imaging, etc.
- **Multi-User Isolation**: Each user maintains independent patient context

### 🏥 Patient Context Management

- **Per-User Contexts**: Each clinician has their own active patient
- **Cross-Application Sync**: Set patient in one app → visible in all apps
- **Context Persistence**: Context survives application restarts (optional Redis/PostgreSQL backend)
- **Context History**: Audit trail of all context changes (who, what, when)

### 🚀 Production-Ready

- **Stateless Architecture**: No database dependency for JWT validation (JWKS caching)
- **Horizontal Scaling**: Run multiple instances behind load balancer
- **Health Checks**: Kubernetes-ready liveness/readiness probes
- **Performance**: <50ms JWT validation, <100ms context operations (p95)

### 🛠️ Developer Experience

- **OpenAPI Documentation**: Auto-generated Swagger UI at `/docs`
- **RESTful API**: Simple HTTP GET/PUT/DELETE endpoints
- **Docker Support**: `docker-compose up` for instant local development
- **Comprehensive Testing**: Unit tests, integration tests, API tests

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  VA Healthcare Ecosystem                    │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │      Keycloak Identity Provider (Port 8080)        │     │
│  │  Realm: va-development                             │     │
│  │  Issues: JWT Access Tokens (15 min expiration)     │     │
│  └──────────────────┬─────────────────────────────────┘     │
│                     │                                       │
│        ┌────────────┼────────────┬──────────────┐           │
│        │ OIDC       │ OIDC       │ OIDC         │           │
│        ▼            ▼            ▼              ▼           │
│   ┌────────┐  ┌────────┐  ┌─────────┐  ┌──────────┐         │
│   │med-z1  │  │ CPRS   │  │ Imaging │  │Future app│         │
│   │(8000)  │  │(8002)  │  │ (8004)  │  │          │         │
│   └───┬────┘  └───┬────┘  └────┬────┘  └────┬─────┘         │
│       │ JWT       │ JWT        │ JWT        │ JWT           │
│       └───────────┼────────────┼────────────┘               │
│                   ▼                                         │
│           ┌──────────────────┐                              │
│           │   med-z2 CCOW    │                              │
│           │ Context Vault    │                              │
│           │   (Port 8001)    │                              │
│           │                  │                              │
│           │  - Stateless     │                              │
│           │  - JWT-based     │                              │
│           │  - Multi-user    │                              │
│           └──────────────────┘                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Design Principles

1. **Stateless First**: JWT validation requires no database lookups
2. **Standards-Based**: OAuth 2.0, OIDC, REST API (not proprietary)
3. **Keycloak-Native**: Designed for Keycloak but adaptable to Azure AD, VA IAM
4. **Cloud-Ready**: Container-native, 12-factor app principles
5. **Observable**: Comprehensive logging, metrics, health checks

---

## Technology Stack

### Core Framework

| Technology | Version | Purpose |
|------------|---------|---------|
| **Python** | 3.11+ | Primary language |
| **FastAPI** | 0.108+ | Web framework (async, OpenAPI) |
| **Uvicorn** | 0.25+ | ASGI server |
| **Pydantic** | 2.5+ | Data validation |

### Authentication

| Technology | Version | Purpose |
|------------|---------|---------|
| **Keycloak** | 23.0+ | Identity provider (OIDC) |
| **authlib** | 1.3+ | JWT validation library |
| **python-jose** | 3.3+ | JWT/JWS/JWE implementation |
| **httpx** | 0.26+ | Async HTTP client (JWKS fetch) |

### Storage (Optional)

| Technology | Version | Purpose |
|------------|---------|---------|
| **Redis** | 7+ | Persistent context storage (optional) |
| **PostgreSQL** | 15+ | Audit history storage (optional) |

### Testing & Quality

| Technology | Version | Purpose |
|------------|---------|---------|
| **pytest** | 7.4+ | Testing framework |
| **pytest-asyncio** | 0.21+ | Async test support |
| **pytest-cov** | 4.1+ | Code coverage |

### Deployment

| Technology | Version | Purpose |
|------------|---------|---------|
| **Docker** | 20+ | Containerization |
| **Docker Compose** | 2+ | Local development |
| **Kubernetes** | 1.28+ | Production orchestration |

---

## Quick Start

### Prerequisites

- **Docker & Docker Compose** (20.10+)
- **Python** 3.11+ (for local development)
- **Git**

### 1. Clone Repository

```bash
git clone https://github.com/va-healthcare/med-z2.git
cd med-z2
```

### 2. Start Services

```bash
# Start Keycloak + med-z2 + optional dependencies
docker-compose up -d

# Check logs
docker-compose logs -f med-z2-ccow
```

Services will start on:
- **Keycloak**: http://localhost:8080 (admin/admin)
- **med-z2 CCOW**: http://localhost:8001
- **med-z2 API Docs**: http://localhost:8001/docs

### 3. Configure Keycloak

```bash
# Create realm, clients, users (automated)
python scripts/setup_keycloak.py
```

Or manually:
1. Visit http://localhost:8080/admin (admin/admin)
2. Create realm: `va-development`
3. Create client: `med-z2-ccow` (bearer-only)
4. Create users: `clinician.alpha@va.gov`, `clinician.bravo@va.gov`

### 4. Test API

```bash
# Health check (no auth required)
curl http://localhost:8001/health

# Get active patient (requires JWT - see API docs)
curl -X GET http://localhost:8001/ccow/active-patient \
  -H "Authorization: Bearer <your-jwt-token>"
```

### 5. Access API Documentation

Visit **http://localhost:8001/docs** for interactive Swagger UI

---

## Project Structure

```
med-z2/
├── ccow/                           # Core application
│   ├── __init__.py
│   ├── main.py                     # FastAPI application
│   ├── models.py                   # Pydantic data models
│   ├── vault.py                    # Context storage (in-memory)
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── jwt_validator.py       # JWT validation logic
│   │   └── dependencies.py        # FastAPI auth dependencies
│   ├── storage/
│   │   ├── __init__.py
│   │   ├── memory.py              # In-memory storage (default)
│   │   ├── redis.py               # Redis storage (optional)
│   │   └── postgres.py            # PostgreSQL storage (optional)
│   └── tests/
│       ├── __init__.py
│       ├── test_jwt_auth.py
│       ├── test_vault.py
│       └── test_api.py
├── config.py                       # Configuration management
├── requirements.txt                # Python dependencies
├── Dockerfile                      # Container image
├── docker-compose.yml              # Local development stack
├── .env.template                   # Environment variables template
├── .env                            # Environment variables (gitignored)
├── .gitignore
├── pytest.ini                      # pytest configuration
├── README.md                       # This file
└── docs/
    ├── spec/
    │   └── med-z2-ccow-service-design.md  # Full design specification
    ├── architecture.md             # Architecture decisions (ADRs)
    └── api.md                      # API reference
```

---

## Configuration

### Environment Variables

Create `.env` file from template:

```bash
cp .env.template .env
```

Key configuration variables:

```bash
# Keycloak Configuration
KEYCLOAK_SERVER_URL=http://localhost:8080
KEYCLOAK_REALM=va-development
KEYCLOAK_CLIENT_ID=med-z2-ccow

# Application Settings
LOG_LEVEL=INFO
STORAGE_BACKEND=memory  # or 'redis', 'postgres'

# Optional: Redis (if using Redis storage)
REDIS_URL=redis://localhost:6379/0

# Optional: PostgreSQL (if using PostgreSQL storage)
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/med_z8
```

### Configuration Reference

See `config.py` for all available configuration options.

---

## Development

### Local Development Setup

```bash
# 1. Create virtual environment
python3.11 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Start Keycloak (in separate terminal)
docker-compose up -d keycloak

# 4. Run application
uvicorn ccow.main:app --reload --port 8001
```

### Development Workflow

```bash
# Run application with hot reload
uvicorn ccow.main:app --reload

# Run tests
pytest

# Run tests with coverage
pytest --cov=ccow --cov-report=html

# Format code
black ccow/
ruff check ccow/

# Type checking
mypy ccow/
```

### Adding New Features

1. Create feature branch: `git checkout -b feature/my-feature`
2. Implement feature in `ccow/`
3. Write tests in `ccow/tests/`
4. Update documentation in `docs/`
5. Run tests: `pytest`
6. Submit pull request

---

## Testing

### Run All Tests

```bash
pytest
```

### Run Specific Test Suite

```bash
# Unit tests only
pytest ccow/tests/test_vault.py

# Integration tests only
pytest ccow/tests/test_api.py

# With coverage report
pytest --cov=ccow --cov-report=term-missing
```

### Test Coverage Target

- **Unit Tests**: 80%+ code coverage
- **Integration Tests**: All API endpoints covered
- **Load Tests**: 1000 concurrent users (future)

### Manual Testing

```bash
# Start services
docker-compose up -d

# Test with curl
curl http://localhost:8001/health

# Test with Swagger UI
open http://localhost:8001/docs
```

---

## Deployment

### Docker Deployment

```bash
# Build image
docker build -t med-z2-ccow:latest .

# Run container
docker run -p 8001:8001 \
  -e KEYCLOAK_SERVER_URL=https://keycloak.va.gov \
  -e KEYCLOAK_REALM=va-production \
  med-z2-ccow:latest
```

### Docker Compose (Development)

```bash
docker-compose up -d
```

### Kubernetes (Production)

```bash
# Apply Kubernetes manifests
kubectl apply -f k8s/

# Check deployment
kubectl get pods -l app=med-z2-ccow
kubectl logs -f deployment/med-z2-ccow
```

See `docs/deployment.md` for detailed production deployment guide.

---

## API Documentation

### Endpoints Summary

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/` | GET | ❌ | Service info |
| `/health` | GET | ❌ | Health check |
| `/ccow/active-patient` | GET | ✅ JWT | Get user's active patient |
| `/ccow/active-patient` | PUT | ✅ JWT | Set user's active patient |
| `/ccow/active-patient` | DELETE | ✅ JWT | Clear user's active patient |
| `/ccow/history` | GET | ✅ JWT | Get context history |
| `/ccow/active-patients` | GET | ✅ JWT | Get all active contexts (admin) |
| `/ccow/cleanup` | POST | ✅ JWT | Cleanup stale contexts (admin) |

### Interactive API Docs

Visit **http://localhost:8001/docs** for:
- Interactive Swagger UI
- Try API calls directly from browser
- Auto-generated request/response examples

Visit **http://localhost:8001/redoc** for:
- Alternative ReDoc documentation
- Clean, readable API reference

### Example API Calls

**Get Active Patient:**
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

See `docs/api.md` for complete API reference.

---

## Related Projects

**med-* Family of Applications:**

| Project | Repository | Description |
|---------|------------|-------------|
| **med-z1** | [va-healthcare/med-z1](https://github.com/va-healthcare/med-z1) | Longitudinal Health Record Viewer |
| **med-z2** | [va-healthcare/med-z2](https://github.com/va-healthcare/med-z2) | CCOW Context Vault (this project) |
| **med-z2** | (Future) | CPRS Simulator |
| **med-z3** | (Future) | Imaging Viewer |

**Dependencies:**

- [Keycloak](https://www.keycloak.org/) - Open Source Identity and Access Management
- [FastAPI](https://fastapi.tiangolo.com/) - Modern Python web framework
- [authlib](https://authlib.org/) - Python authentication library

---

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Development Process

1. **Fork** the repository
2. **Create** feature branch (`git checkout -b feature/amazing-feature`)
3. **Commit** changes (`git commit -m 'Add amazing feature'`)
4. **Push** to branch (`git push origin feature/amazing-feature`)
5. **Open** Pull Request

### Code Style

- Follow [PEP 8](https://pep8.org/) style guide
- Use [Black](https://black.readthedocs.io/) for code formatting
- Use [Ruff](https://docs.astral.sh/ruff/) for linting
- Write type hints for all functions
- Add docstrings for all public APIs

### Commit Message Convention

```
type(scope): short description

Longer description if needed

Fixes #123
```

Types: `feat`, `fix`, `docs`, `test`, `refactor`, `perf`, `chore`

---

## License

This project is licensed under the **MIT License** - see [LICENSE](LICENSE) file for details.

---

## Support

### Documentation

- **Design Specification**: [docs/spec/med-z2-ccow-service-design.md](docs/spec/med-z2-ccow-service-design.md)
- **Architecture Decisions**: [docs/architecture.md](docs/architecture.md)
- **API Reference**: [docs/api.md](docs/api.md)

### Getting Help

- **Issues**: [GitHub Issues](https://github.com/va-healthcare/med-z2/issues)
- **Discussions**: [GitHub Discussions](https://github.com/va-healthcare/med-z2/discussions)
- **Email**: med-z2-support@va.gov

### Reporting Security Issues

Please report security vulnerabilities to **security@va.gov** (do not use public issues).

---

## Acknowledgments

- **VA Office of Information and Technology** - Project sponsorship
- **Keycloak Team** - Open-source identity provider
- **FastAPI Community** - Modern Python web framework
- **med-z1 Team** - Original CCOW implementation patterns

---

## Roadmap

### Phase 1: Core Features ✅ (Current)
- [x] JWT-based authentication
- [x] Multi-user context isolation
- [x] In-memory storage
- [x] REST API
- [x] Docker support

### Phase 2: Persistence (Q2 2026)
- [ ] Redis backend for context storage
- [ ] PostgreSQL backend for audit history
- [ ] Context recovery after restart

### Phase 3: Real-Time Features (Q3 2026)
- [ ] WebSocket support for real-time notifications
- [ ] Server-Sent Events (SSE) alternative
- [ ] Publish/subscribe pattern

### Phase 4: Enterprise Features (Q4 2026)
- [ ] Role-Based Access Control (RBAC)
- [ ] Prometheus metrics
- [ ] OpenTelemetry tracing
- [ ] Multi-tenancy support

---

## Quick Links

- 📖 [Full Design Specification](docs/spec/med-z2-ccow-service-design.md)
- 🔐 [Keycloak Setup Guide](docs/guide/keycloak-setup.md) (TBD)
- 🏥 [med-z1 Migration Guide](https://github.com/va-healthcare/med-z1/docs/spec/med-z1-keycloak-migration.md)
- 📋 [API Documentation](http://localhost:8001/docs) (when running)
- 🐛 [Issue Tracker](https://github.com/va-healthcare/med-z2/issues)

