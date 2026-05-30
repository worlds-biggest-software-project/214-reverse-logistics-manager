# Reverse Logistics Manager — Phased Development Plan

> Project: 214-reverse-logistics-manager · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12+ | ML/CV-heavy feature set (disposition ML, condition grading CV, fraud scoring) makes Python the natural choice. LLM SDKs, scikit-learn, PyTorch, and the data science ecosystem are Python-first. |
| API framework | FastAPI 0.115+ | Automatic OpenAPI 3.1 spec generation (required by standards.md), async support for long-running AI inference calls, Pydantic v2 for request/response validation, and native dependency injection. |
| Database | PostgreSQL 16 | Multi-tenant RLS, JSONB with GIN indexes for variable tenant data, ltree extension for category hierarchies, UUID primary keys, and production-grade reliability. SQLite is unsuitable for multi-tenant SaaS. |
| ORM / query builder | SQLAlchemy 2.0 + Alembic | Async support via asyncpg, declarative models aligned with Pydantic, Alembic for versioned migrations. |
| Task queue | Celery 5 + Redis | Async workloads: carrier label generation, AI inference (condition grading, fraud scoring, disposition ML), webhook delivery, carbon footprint calculation. Redis doubles as cache. |
| Cache | Redis 7 | Session cache, rate limiting, task queue broker, and hot-path caching for policy evaluation and fraud scores. |
| Frontend | Next.js 15 (React 19) | Consumer return portal (SSR for SEO/performance), merchant operations dashboard (SPA), and configurable branding per tenant. App Router with Server Components. |
| CSS / UI | Tailwind CSS 4 + shadcn/ui | Rapid UI development with accessible, composable components. Tenant-level theme customisation via CSS variables. |
| File storage | S3-compatible (MinIO for self-hosted) | Inspection photos, return labels (PDF), recycling certificates. Pre-signed URLs for direct browser uploads. |
| ML/CV framework | PyTorch + torchvision | Condition grading CV model, fraud image analysis. ONNX export for inference serving. |
| ML serving | FastAPI + ONNX Runtime | Co-located inference service for CV grading and disposition ML. Separable into dedicated service at scale. |
| LLM SDK | Anthropic SDK (Claude) | Chatbot return initiation, natural language reason code extraction, AI-assisted fraud narrative generation. |
| Carrier integration | EasyPost API | Single API for UPS, FedEx, USPS, DHL label generation and tracking. Avoids per-carrier integration complexity. |
| Containerisation | Docker + Docker Compose | Self-hosted deployment target; Compose for local dev with PostgreSQL, Redis, MinIO. |
| Testing | pytest + pytest-asyncio + httpx | Unit and integration tests. httpx.AsyncClient for API tests. Factory Boy for test fixtures. |
| E2E testing | Playwright | Consumer portal and merchant dashboard E2E tests. |
| Code quality | Ruff (lint + format) + mypy (strict) | Ruff replaces flake8/black/isort with a single fast tool. mypy strict mode enforces type safety. |
| Package manager | uv | Fast dependency resolution and virtual environment management. pyproject.toml for project metadata. |
| API documentation | Auto-generated OpenAPI 3.1 + Redoc | FastAPI generates the spec; Redoc serves interactive docs. Aligns with standards.md requirement. |
| Authentication | OAuth 2.0 (client credentials for M2M) + OIDC (merchant portal SSO) | RFC 6749 client credentials for API integrations; OIDC for merchant user login. JWT access tokens per RFC 7519. |

### Project Structure

```
reverse-logistics-manager/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   └── versions/
├── src/
│   └── rlm/
│       ├── __init__.py
│       ├── main.py                      # FastAPI app factory
│       ├── config.py                    # Settings via pydantic-settings
│       ├── database.py                  # SQLAlchemy engine, session factory
│       ├── models/                      # SQLAlchemy ORM models
│       │   ├── __init__.py
│       │   ├── tenant.py
│       │   ├── user.py
│       │   ├── product.py
│       │   ├── consumer.py
│       │   ├── order.py
│       │   ├── return_request.py
│       │   ├── return_item.py
│       │   ├── policy.py
│       │   ├── shipment.py
│       │   ├── inspection.py
│       │   ├── disposition.py
│       │   ├── warranty.py
│       │   ├── repair.py
│       │   ├── fraud.py
│       │   ├── carbon.py
│       │   ├── integration.py
│       │   └── audit.py
│       ├── schemas/                     # Pydantic request/response schemas
│       │   ├── __init__.py
│       │   ├── tenant.py
│       │   ├── consumer.py
│       │   ├── return_request.py
│       │   ├── inspection.py
│       │   ├── disposition.py
│       │   ├── shipment.py
│       │   ├── policy.py
│       │   ├── fraud.py
│       │   └── common.py
│       ├── api/                         # FastAPI route modules
│       │   ├── __init__.py
│       │   ├── deps.py                  # Dependency injection (auth, db session, tenant)
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── router.py            # v1 API router aggregation
│       │   │   ├── returns.py
│       │   │   ├── inspections.py
│       │   │   ├── dispositions.py
│       │   │   ├── shipments.py
│       │   │   ├── policies.py
│       │   │   ├── consumers.py
│       │   │   ├── products.py
│       │   │   ├── tenants.py
│       │   │   ├── fraud.py
│       │   │   ├── analytics.py
│       │   │   ├── webhooks.py
│       │   │   └── auth.py
│       │   └── portal/                  # Consumer-facing portal API
│       │       ├── __init__.py
│       │       └── returns.py
│       ├── services/                    # Business logic layer
│       │   ├── __init__.py
│       │   ├── return_service.py
│       │   ├── policy_engine.py
│       │   ├── disposition_engine.py
│       │   ├── fraud_service.py
│       │   ├── inspection_service.py
│       │   ├── shipment_service.py
│       │   ├── warranty_service.py
│       │   ├── carbon_service.py
│       │   ├── webhook_service.py
│       │   └── rma_number.py
│       ├── integrations/                # External system connectors
│       │   ├── __init__.py
│       │   ├── easypost.py              # Carrier label/tracking via EasyPost
│       │   ├── shopify.py
│       │   ├── ecommerce_base.py        # Abstract base for e-commerce connectors
│       │   └── s3.py                    # S3/MinIO file operations
│       ├── ml/                          # ML model inference
│       │   ├── __init__.py
│       │   ├── condition_grader.py      # CV condition grading
│       │   ├── disposition_model.py     # ML disposition recommendation
│       │   ├── fraud_scorer.py          # Fraud risk scoring
│       │   └── refurb_estimator.py      # Refurbishment cost estimation
│       ├── tasks/                       # Celery async tasks
│       │   ├── __init__.py
│       │   ├── celery_app.py
│       │   ├── label_tasks.py
│       │   ├── ai_tasks.py
│       │   ├── webhook_tasks.py
│       │   └── carbon_tasks.py
│       └── auth/                        # Authentication & authorisation
│           ├── __init__.py
│           ├── jwt.py
│           ├── oauth.py
│           └── permissions.py
├── frontend/                            # Next.js frontend
│   ├── package.json
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── portal/                  # Consumer return portal
│   │   │   │   ├── page.tsx
│   │   │   │   ├── [rma]/
│   │   │   │   └── submit/
│   │   │   └── dashboard/               # Merchant operations dashboard
│   │   │       ├── page.tsx
│   │   │       ├── returns/
│   │   │       ├── inspections/
│   │   │       ├── analytics/
│   │   │       ├── policies/
│   │   │       └── settings/
│   │   ├── components/
│   │   │   ├── ui/                      # shadcn/ui components
│   │   │   ├── portal/
│   │   │   └── dashboard/
│   │   └── lib/
│   │       ├── api-client.ts
│   │       └── types.ts
│   └── tests/
│       └── e2e/
├── tests/
│   ├── conftest.py
│   ├── factories/                       # Factory Boy test data factories
│   │   ├── __init__.py
│   │   ├── tenant.py
│   │   ├── consumer.py
│   │   ├── return_request.py
│   │   └── product.py
│   ├── unit/
│   │   ├── test_policy_engine.py
│   │   ├── test_fraud_service.py
│   │   ├── test_disposition_engine.py
│   │   ├── test_rma_number.py
│   │   └── test_carbon_service.py
│   ├── integration/
│   │   ├── test_returns_api.py
│   │   ├── test_inspections_api.py
│   │   ├── test_dispositions_api.py
│   │   ├── test_policies_api.py
│   │   └── test_webhooks.py
│   └── fixtures/
│       ├── sample_products.json
│       ├── sample_orders.json
│       └── sample_return_photos/
├── scripts/
│   ├── seed_dev_data.py
│   └── run_migrations.py
└── docs/
    └── api/                             # Auto-generated OpenAPI spec served here
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the project skeleton, development toolchain, database connectivity, configuration management, and CI-ready test infrastructure. After this phase, a developer can clone the repo, run `docker compose up`, and have a running FastAPI application connected to PostgreSQL and Redis with passing tests.

### Tasks

#### 1.1 — Project Initialisation & Dependency Management

**What**: Create the Python project with uv, configure pyproject.toml, and install core dependencies.

**Design**:

```toml
# pyproject.toml
[project]
name = "reverse-logistics-manager"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "pydantic>=2.10.0",
    "pydantic-settings>=2.6.0",
    "sqlalchemy[asyncio]>=2.0.36",
    "asyncpg>=0.30.0",
    "alembic>=1.14.0",
    "redis>=5.2.0",
    "celery[redis]>=5.4.0",
    "httpx>=0.28.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "boto3>=1.35.0",
    "python-multipart>=0.0.17",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "factory-boy>=3.3.0",
    "httpx>=0.28.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
    "types-redis>=4.6.0",
]
ml = [
    "torch>=2.5.0",
    "torchvision>=0.20.0",
    "onnxruntime>=1.20.0",
    "pillow>=11.0.0",
    "scikit-learn>=1.6.0",
]
```

**Testing**:
- `Unit: uv sync installs all dependencies without errors`
- `Unit: uv sync --extra dev installs dev dependencies`
- `Unit: python -c "import rlm" succeeds`

---

#### 1.2 — Configuration Management

**What**: Implement application configuration using pydantic-settings with environment variable support and sensible defaults.

**Design**:

```python
# src/rlm/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="RLM_",
        env_file=".env",
        env_file_encoding="utf-8",
    )

    # Application
    app_name: str = "Reverse Logistics Manager"
    debug: bool = False
    api_v1_prefix: str = "/api/v1"
    portal_api_prefix: str = "/api/portal"
    allowed_origins: list[str] = ["http://localhost:3000"]

    # Database
    database_url: str = "postgresql+asyncpg://rlm:rlm@localhost:5432/rlm"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"
    celery_broker_url: str = "redis://localhost:6379/1"

    # Auth
    jwt_secret_key: str = "CHANGE-ME-IN-PRODUCTION"
    jwt_algorithm: str = "HS256"
    jwt_access_token_expire_minutes: int = 60

    # S3 / MinIO
    s3_endpoint_url: str = "http://localhost:9000"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    s3_bucket_name: str = "rlm-uploads"

    # EasyPost (carrier integration)
    easypost_api_key: str = ""
    easypost_test_mode: bool = True

    # RMA number format
    rma_prefix: str = "RMA"
    rma_year_prefix: bool = True

def get_settings() -> Settings:
    return Settings()
```

Environment variables: `RLM_DATABASE_URL`, `RLM_REDIS_URL`, `RLM_JWT_SECRET_KEY`, etc.

**Testing**:
- `Unit: default settings load with correct defaults`
- `Unit: RLM_DATABASE_URL env var overrides default database_url`
- `Unit: missing required settings with no default raises ValidationError`
- `Unit: .env file is read when present`

---

#### 1.3 — Database Connection & Session Management

**What**: Set up async SQLAlchemy engine, session factory, and FastAPI dependency for database sessions.

**Design**:

```python
# src/rlm/database.py
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

engine = None
async_session_factory = None

def init_engine(database_url: str, pool_size: int = 20, max_overflow: int = 10):
    global engine, async_session_factory
    engine = create_async_engine(
        database_url,
        pool_size=pool_size,
        max_overflow=max_overflow,
        echo=False,
    )
    async_session_factory = async_sessionmaker(
        engine, class_=AsyncSession, expire_on_commit=False
    )

async def get_db_session() -> AsyncSession:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

**Testing**:
- `Integration: init_engine with valid PostgreSQL URL creates engine without error`
- `Integration: get_db_session yields a working session that can execute SELECT 1`
- `Integration: session rollback on exception prevents partial writes`

---

#### 1.4 — FastAPI Application Factory & Health Check

**What**: Create the FastAPI application with lifespan management, CORS, and a health check endpoint.

**Design**:

```python
# src/rlm/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from rlm.config import get_settings
from rlm.database import init_engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    init_engine(settings.database_url, settings.database_pool_size, settings.database_max_overflow)
    yield

def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(
        title=settings.app_name,
        version="0.1.0",
        lifespan=lifespan,
        docs_url="/docs",
        redoc_url="/redoc",
        openapi_url="/openapi.json",
    )
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.allowed_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    # Include routers (added in later phases)
    return app

app = create_app()

@app.get("/health", tags=["system"])
async def health_check():
    return {"status": "healthy", "version": "0.1.0"}
```

**Testing**:
- `Integration: GET /health returns 200 with {"status": "healthy"}`
- `Integration: GET /openapi.json returns valid OpenAPI 3.1 spec`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: CORS headers present for allowed origins`

---

#### 1.5 — Docker Compose Development Environment

**What**: Create Docker Compose configuration for local development with PostgreSQL, Redis, and MinIO.

**Design**:

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      RLM_DATABASE_URL: postgresql+asyncpg://rlm:rlm@db:5432/rlm
      RLM_REDIS_URL: redis://redis:6379/0
      RLM_CELERY_BROKER_URL: redis://redis:6379/1
      RLM_S3_ENDPOINT_URL: http://minio:9000
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src

  worker:
    build: .
    command: celery -A rlm.tasks.celery_app worker --loglevel=info
    environment:
      RLM_DATABASE_URL: postgresql+asyncpg://rlm:rlm@db:5432/rlm
      RLM_REDIS_URL: redis://redis:6379/0
      RLM_CELERY_BROKER_URL: redis://redis:6379/1
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: rlm
      POSTGRES_PASSWORD: rlm
      POSTGRES_DB: rlm
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rlm"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  miniodata:
```

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install uv

COPY pyproject.toml ./
RUN uv sync --frozen

COPY src/ ./src/
COPY alembic.ini ./
COPY alembic/ ./alembic/

CMD ["uv", "run", "uvicorn", "rlm.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

**Testing**:
- `E2E: docker compose up --build starts all services without errors`
- `E2E: curl http://localhost:8000/health returns 200`
- `E2E: PostgreSQL accepts connections on port 5432`
- `E2E: Redis responds to PING on port 6379`

---

#### 1.6 — Test Infrastructure

**What**: Configure pytest with async support, database fixtures, and Factory Boy base classes.

**Design**:

```python
# tests/conftest.py
import asyncio
from collections.abc import AsyncGenerator

import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

from rlm.database import Base, get_db_session
from rlm.main import create_app
from rlm.config import Settings

TEST_DATABASE_URL = "postgresql+asyncpg://rlm:rlm@localhost:5432/rlm_test"

@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

@pytest_asyncio.fixture(scope="session")
async def test_engine():
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest_asyncio.fixture
async def db_session(test_engine) -> AsyncGenerator[AsyncSession, None]:
    session_factory = async_sessionmaker(test_engine, class_=AsyncSession, expire_on_commit=False)
    async with session_factory() as session:
        yield session
        await session.rollback()

@pytest_asyncio.fixture
async def client(db_session: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    app = create_app()
    app.dependency_overrides[get_db_session] = lambda: db_session
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client
```

**Testing**:
- `Unit: pytest discovers and runs a sample test`
- `Integration: db_session fixture provides a working database session`
- `Integration: client fixture sends requests to the test app`
- `Unit: ruff check passes with zero violations`
- `Unit: mypy --strict src/ passes with zero errors`

---

## Phase 2: Core Data Model & Multi-Tenancy

### Purpose
Implement the database schema covering tenants, users, products, consumers, orders, and the return lifecycle. Establish row-level tenant isolation and Alembic migration infrastructure. After this phase, the foundational data model supports creating tenants, products, consumers, and return requests via direct database operations.

### Tasks

#### 2.1 — Alembic Migration Infrastructure

**What**: Configure Alembic for async SQLAlchemy with an initial migration that creates core tables.

**Design**:

Alembic `env.py` configured for async with `asyncpg`. Target metadata from `rlm.database.Base.metadata`. Migration naming convention: `NNNN_description.py` (auto-generated timestamp).

```python
# alembic/env.py (key section)
from rlm.database import Base
from rlm.models import *  # noqa: ensure all models are imported
target_metadata = Base.metadata
```

**Testing**:
- `Integration: alembic upgrade head applies all migrations without error on empty database`
- `Integration: alembic downgrade base reverses all migrations cleanly`
- `Integration: alembic check reports no pending migrations when models match schema`

---

#### 2.2 — Tenant & User Models

**What**: Implement the tenant and app_user SQLAlchemy models based on data-model-suggestion-1 schema.

**Design**:

```python
# src/rlm/models/tenant.py
import uuid
from datetime import datetime
from sqlalchemy import Boolean, DateTime, String, text
from sqlalchemy.dialects.postgresql import JSONB, UUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from rlm.database import Base

class Tenant(Base):
    __tablename__ = "tenant"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, server_default=text("gen_random_uuid()"))
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    slug: Mapped[str] = mapped_column(String(100), nullable=False, unique=True)
    type: Mapped[str] = mapped_column(String(50), nullable=False)  # retailer, manufacturer, 3pl, brand
    subscription_tier: Mapped[str] = mapped_column(String(50), nullable=False, server_default="standard")
    default_currency: Mapped[str] = mapped_column(String(3), nullable=False, server_default="USD")
    timezone: Mapped[str] = mapped_column(String(50), nullable=False, server_default="UTC")
    settings: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default=text("'{}'::jsonb"))
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default=text("true"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=text("now()"))
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=text("now()"))

    users: Mapped[list["AppUser"]] = relationship(back_populates="tenant")
```

```python
# src/rlm/models/user.py
class AppUser(Base):
    __tablename__ = "app_user"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, server_default=text("gen_random_uuid()"))
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    email: Mapped[str] = mapped_column(String(255), nullable=False)
    full_name: Mapped[str] = mapped_column(String(255), nullable=False)
    role: Mapped[str] = mapped_column(String(50), nullable=False)  # admin, manager, operator, viewer, api_client
    password_hash: Mapped[str | None] = mapped_column(String(255), nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default=text("true"))
    last_login_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=text("now()"))
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=text("now()"))

    tenant: Mapped["Tenant"] = relationship(back_populates="users")

    __table_args__ = (
        UniqueConstraint("tenant_id", "email", name="uq_app_user_tenant_email"),
        Index("idx_app_user_tenant", "tenant_id"),
    )
```

**Testing**:
- `Integration: create Tenant with all fields persists and is retrievable`
- `Integration: create AppUser linked to Tenant via tenant_id foreign key`
- `Integration: duplicate tenant slug raises IntegrityError`
- `Integration: duplicate email within same tenant raises IntegrityError`
- `Integration: same email across different tenants succeeds`

---

#### 2.3 — Product, Consumer & Order Models

**What**: Implement product, consumer, address, and original_order models.

**Design**:

Follow data-model-suggestion-1 schema for product (with `gtin`, `weee_category`, `right_to_repair_eligible`), consumer (with `fraud_risk_score`, `data_retention_expiry`), address (ISO 3166 codes), and original_order (with `platform`, `channel`). All models include `tenant_id` foreign key and appropriate indexes.

Key fields on Product:
- `gtin: VARCHAR(14)` — GS1 GTIN-14, indexed for lookups
- `weee_category: VARCHAR(50)` — nullable, for WEEE-regulated items
- `right_to_repair_eligible: BOOLEAN` — defaults to FALSE
- `digital_link_url: VARCHAR(500)` — GS1 Digital Link URI

Key fields on Consumer:
- `fraud_risk_score: NUMERIC(5,2)` — 0.00 to 100.00
- `data_retention_expiry: TIMESTAMPTZ` — GDPR/CCPA compliance
- `customer_tier: VARCHAR(50)` — standard, vip, high_risk

**Testing**:
- `Integration: create Product with GTIN, retrieve by GTIN index`
- `Integration: create Consumer with fraud_risk_score, enforce UNIQUE(tenant_id, email)`
- `Integration: create Address with ISO 3166-1 country_code`
- `Integration: create OriginalOrder linked to Consumer and Address`
- `Integration: UNIQUE(tenant_id, sku) prevents duplicate SKUs within a tenant`

---

#### 2.4 — Return Request & Return Item Models

**What**: Implement the core return_request and return_item models with lifecycle status enforcement.

**Design**:

```python
# src/rlm/models/return_request.py
class ReturnRequest(Base):
    __tablename__ = "return_request"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, server_default=text("gen_random_uuid()"))
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("tenant.id"), nullable=False)
    rma_number: Mapped[str] = mapped_column(String(50), nullable=False)
    consumer_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("consumer.id"), nullable=False)
    original_order_id: Mapped[uuid.UUID] = mapped_column(UUID, ForeignKey("original_order.id"), nullable=False)
    status: Mapped[str] = mapped_column(
        String(50), nullable=False, server_default="requested"
    )
    # Valid statuses: requested, approved, rejected, label_generated,
    #   in_transit, received, inspecting, disposition_decided, resolved, cancelled
    return_type: Mapped[str] = mapped_column(String(50), nullable=False, server_default="return")
    # return, exchange, warranty_claim, repair
    resolution_type: Mapped[str | None] = mapped_column(String(50), nullable=True)
    # refund, exchange, store_credit, repair, keep_item
    initiated_via: Mapped[str] = mapped_column(String(50), server_default="portal")
    # portal, api, chatbot, phone, in_store
    fraud_risk_score: Mapped[float | None] = mapped_column(Numeric(5, 2), nullable=True)
    fraud_flags: Mapped[list | None] = mapped_column(ARRAY(String), nullable=True)
    approved_by: Mapped[uuid.UUID | None] = mapped_column(UUID, ForeignKey("app_user.id"), nullable=True)
    approved_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    received_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    resolved_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    refund_amount: Mapped[float | None] = mapped_column(Numeric(12, 2), nullable=True)
    refund_currency: Mapped[str | None] = mapped_column(String(3), nullable=True)
    store_credit_amount: Mapped[float | None] = mapped_column(Numeric(12, 2), nullable=True)
    notes: Mapped[str | None] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=text("now()"))
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, server_default=text("now()"))

    items: Mapped[list["ReturnItem"]] = relationship(back_populates="return_request")

    __table_args__ = (
        UniqueConstraint("tenant_id", "rma_number", name="uq_return_request_tenant_rma"),
        Index("idx_return_request_tenant_status", "tenant_id", "status"),
        Index("idx_return_request_consumer", "consumer_id"),
        Index("idx_return_request_created", "tenant_id", postgresql_using="btree"),
    )
```

Return item model with `reason_code`, `condition_at_request`, `unit_price`, `serial_number`, and FK to `product`.

State machine for `return_request.status`:
```
requested -> approved -> label_generated -> in_transit -> received -> inspecting -> disposition_decided -> resolved
requested -> rejected
requested -> cancelled
approved -> cancelled
```

**Testing**:
- `Integration: create ReturnRequest with status "requested", verify default`
- `Integration: add ReturnItems to a ReturnRequest, verify relationship loads`
- `Integration: UNIQUE(tenant_id, rma_number) prevents duplicate RMA numbers`
- `Unit: status transitions validated (requested->approved OK, resolved->requested rejected)`
- `Integration: cascade delete — deleting ReturnRequest deletes related ReturnItems`

---

#### 2.5 — Reason Code & Audit Log Models

**What**: Implement the reason_code reference table and the audit_log table for state change tracking.

**Design**:

Reason code model per data-model-suggestion-1: `code`, `label`, `category` (quality, preference, logistics, fraud), `requires_photo`, `display_order`. Tenant-scoped with UNIQUE(tenant_id, code).

Audit log model: `entity_type`, `entity_id`, `action` (created, updated, status_changed, approved, rejected), `old_values` (JSONB), `new_values` (JSONB), `performed_by`, `ip_address` (INET). Indexed on (entity_type, entity_id) and (tenant_id, created_at DESC).

**Testing**:
- `Integration: create reason codes for a tenant, retrieve ordered by display_order`
- `Integration: audit log entry created with old/new JSONB values`
- `Integration: query audit log by entity_type and entity_id returns chronological history`

---

#### 2.6 — RMA Number Generation Service

**What**: Implement deterministic, collision-free RMA number generation.

**Design**:

```python
# src/rlm/services/rma_number.py
from datetime import datetime

class RMANumberGenerator:
    """Generates RMA numbers in format: {prefix}-{year}-{sequence}
    Example: RMA-2026-00142
    """

    def __init__(self, prefix: str = "RMA", include_year: bool = True):
        self.prefix = prefix
        self.include_year = include_year

    async def next_number(self, session: AsyncSession, tenant_id: uuid.UUID) -> str:
        """Generate next RMA number using a database sequence per tenant-year."""
        year = datetime.now().year
        # Use advisory lock to prevent race conditions
        await session.execute(text(f"SELECT pg_advisory_xact_lock(hashtext(:key))"),
                              {"key": f"rma_{tenant_id}_{year}"})

        result = await session.execute(
            text("""
                SELECT COUNT(*) + 1
                FROM return_request
                WHERE tenant_id = :tenant_id
                AND EXTRACT(YEAR FROM created_at) = :year
            """),
            {"tenant_id": tenant_id, "year": year}
        )
        seq = result.scalar_one()

        if self.include_year:
            return f"{self.prefix}-{year}-{seq:05d}"
        return f"{self.prefix}-{seq:07d}"
```

**Testing**:
- `Unit: RMA-2026-00001 generated for first return of 2026`
- `Unit: RMA-2026-00142 generated when sequence is at 141`
- `Unit: custom prefix "RTN" produces RTN-2026-00001`
- `Integration: concurrent RMA generation produces unique numbers (no collisions)`

---

## Phase 3: Authentication, API Layer & Return Lifecycle

### Purpose
Implement JWT authentication, tenant-scoped API authorization, and the core return CRUD endpoints. After this phase, API clients can authenticate, create return requests, update statuses, and query returns — all tenant-isolated.

### Tasks

#### 3.1 — JWT Authentication & API Key Support

**What**: Implement JWT token issuance, validation, and API key authentication for M2M integration.

**Design**:

```python
# src/rlm/auth/jwt.py
from datetime import datetime, timedelta, timezone
from jose import JWTError, jwt
from pydantic import BaseModel

class TokenPayload(BaseModel):
    sub: str              # user ID or api_client ID
    tenant_id: str
    role: str             # admin, manager, operator, viewer, api_client
    exp: datetime
    iat: datetime

def create_access_token(
    user_id: str,
    tenant_id: str,
    role: str,
    secret_key: str,
    algorithm: str = "HS256",
    expires_delta: timedelta = timedelta(minutes=60),
) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": user_id,
        "tenant_id": tenant_id,
        "role": role,
        "iat": now,
        "exp": now + expires_delta,
    }
    return jwt.encode(payload, secret_key, algorithm=algorithm)

def verify_token(token: str, secret_key: str, algorithm: str = "HS256") -> TokenPayload:
    payload = jwt.decode(token, secret_key, algorithms=[algorithm])
    return TokenPayload(**payload)
```

```python
# src/rlm/api/deps.py
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(security),
    db: AsyncSession = Depends(get_db_session),
) -> AppUser:
    token_data = verify_token(credentials.credentials, settings.jwt_secret_key)
    user = await db.get(AppUser, uuid.UUID(token_data.sub))
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="Invalid or inactive user")
    return user

async def get_current_tenant(user: AppUser = Depends(get_current_user)) -> Tenant:
    return user.tenant

def require_role(*roles: str):
    def checker(user: AppUser = Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return checker
```

Endpoints:
- `POST /api/v1/auth/login` — email/password -> JWT token
- `POST /api/v1/auth/token` — OAuth 2.0 client credentials grant (for M2M)

**Testing**:
- `Unit: create_access_token produces valid JWT with correct claims`
- `Unit: verify_token rejects expired tokens with appropriate error`
- `Unit: verify_token rejects tokens signed with wrong key`
- `Integration: POST /api/v1/auth/login with valid credentials returns JWT`
- `Integration: POST /api/v1/auth/login with invalid password returns 401`
- `Integration: protected endpoint without token returns 401`
- `Integration: protected endpoint with valid token returns 200`
- `Integration: require_role("admin") rejects "viewer" role with 403`

---

#### 3.2 — Tenant Management API

**What**: CRUD endpoints for tenant management (admin-only).

**Design**:

```python
# Pydantic schemas
class TenantCreate(BaseModel):
    name: str
    slug: str = Field(pattern=r"^[a-z0-9-]+$", max_length=100)
    type: Literal["retailer", "manufacturer", "3pl", "brand"]
    default_currency: str = Field(default="USD", max_length=3)
    timezone: str = "UTC"

class TenantResponse(BaseModel):
    id: uuid.UUID
    name: str
    slug: str
    type: str
    subscription_tier: str
    default_currency: str
    timezone: str
    is_active: bool
    created_at: datetime
```

Endpoints:
- `POST /api/v1/tenants` — create tenant (system admin only)
- `GET /api/v1/tenants/{tenant_id}` — get tenant details
- `PATCH /api/v1/tenants/{tenant_id}` — update tenant settings
- `POST /api/v1/tenants/{tenant_id}/users` — create user within tenant

**Testing**:
- `Integration: POST /api/v1/tenants creates tenant and returns 201`
- `Integration: GET /api/v1/tenants/{id} returns tenant details`
- `Integration: PATCH with invalid slug pattern returns 422`
- `Integration: non-admin user receives 403 on POST /api/v1/tenants`

---

#### 3.3 — Return Request CRUD API

**What**: Full CRUD endpoints for return requests with tenant isolation.

**Design**:

```python
# Pydantic schemas
class ReturnItemCreate(BaseModel):
    product_id: uuid.UUID
    quantity: int = Field(ge=1, default=1)
    reason_code: str
    reason_detail: str | None = None
    condition_at_request: str | None = None
    unit_price: Decimal
    serial_number: str | None = None

class ReturnRequestCreate(BaseModel):
    consumer_id: uuid.UUID
    original_order_id: uuid.UUID
    return_type: Literal["return", "exchange", "warranty_claim", "repair"] = "return"
    initiated_via: Literal["portal", "api", "chatbot", "phone", "in_store"] = "api"
    items: list[ReturnItemCreate] = Field(min_length=1)
    notes: str | None = None

class ReturnRequestResponse(BaseModel):
    id: uuid.UUID
    rma_number: str
    consumer_id: uuid.UUID
    original_order_id: uuid.UUID
    status: str
    return_type: str
    resolution_type: str | None
    initiated_via: str
    fraud_risk_score: float | None
    items: list[ReturnItemResponse]
    refund_amount: Decimal | None
    store_credit_amount: Decimal | None
    created_at: datetime
    updated_at: datetime

class ReturnListParams(BaseModel):
    status: str | None = None
    consumer_id: uuid.UUID | None = None
    created_after: datetime | None = None
    created_before: datetime | None = None
    page: int = Field(ge=1, default=1)
    page_size: int = Field(ge=1, le=100, default=20)

class ReturnStatusUpdate(BaseModel):
    status: Literal["approved", "rejected", "cancelled"]
    resolution_type: Literal["refund", "exchange", "store_credit", "repair", "keep_item"] | None = None
    notes: str | None = None
```

Endpoints:
- `POST /api/v1/returns` — create return request, auto-generates RMA number
- `GET /api/v1/returns` — list returns with filtering and pagination (Link header pagination per RFC 8288)
- `GET /api/v1/returns/{rma_number}` — get single return by RMA number
- `PATCH /api/v1/returns/{rma_number}/status` — update return status (approve, reject, cancel)
- `GET /api/v1/returns/{rma_number}/timeline` — audit log for this return

All queries scoped to `tenant_id` from the authenticated user's token.

**Testing**:
- `Integration: POST /api/v1/returns creates return with auto-generated RMA number`
- `Integration: POST with empty items array returns 422`
- `Integration: GET /api/v1/returns returns only returns for authenticated tenant`
- `Integration: GET /api/v1/returns?status=requested filters correctly`
- `Integration: GET /api/v1/returns with pagination returns Link header`
- `Integration: PATCH status from "requested" to "approved" succeeds`
- `Integration: PATCH status from "resolved" to "requested" returns 409 (invalid transition)`
- `Integration: tenant A cannot access tenant B's returns (returns 404)`
- `Integration: audit log entry created on status change with old/new values`

---

#### 3.4 — Product & Consumer APIs

**What**: CRUD endpoints for products (synced from e-commerce) and consumers.

**Design**:

Product endpoints:
- `POST /api/v1/products` — create/upsert product
- `GET /api/v1/products` — list with filtering (category, GTIN, active status)
- `GET /api/v1/products/{id}` — get product details
- `PATCH /api/v1/products/{id}` — update product
- `POST /api/v1/products/bulk` — bulk upsert (for e-commerce sync)

Consumer endpoints:
- `POST /api/v1/consumers` — create consumer
- `GET /api/v1/consumers` — list with filtering (tier, risk score range)
- `GET /api/v1/consumers/{id}` — get consumer with return history summary
- `GET /api/v1/consumers/{id}/returns` — list consumer's returns

Product bulk upsert uses `ON CONFLICT (tenant_id, sku) DO UPDATE` for idempotent syncing.

**Testing**:
- `Integration: POST /api/v1/products creates product, returns 201`
- `Integration: bulk upsert 100 products completes without error`
- `Integration: bulk upsert with existing SKU updates (not duplicates)`
- `Integration: GET /api/v1/consumers/{id}/returns returns consumer's returns with correct count`
- `Integration: consumer.fraud_risk_score updates when return history changes`

---

## Phase 4: Policy Engine & Fraud Scoring

### Purpose
Implement the configurable return policy engine with rule evaluation, and the initial fraud scoring system using heuristic signals. After this phase, return requests are automatically evaluated against tenant policies and assigned fraud risk scores.

### Tasks

#### 4.1 — Return Policy Model & CRUD

**What**: Implement the return_policy and policy_rule models with tenant-scoped CRUD.

**Design**:

Policy model with JSONB rule definitions (per data-model-suggestion-3 approach for flexibility):

```python
class PolicyRuleCondition(BaseModel):
    field: str         # customer_tier, product_category, order_value, item_value, reason_code, jurisdiction, return_count_30d
    operator: str      # equals, not_equals, greater_than, less_than, in, not_in, contains
    value: Any         # comparison value

class PolicyRuleAction(BaseModel):
    type: str          # approve, reject, require_review, extend_window, apply_fee, require_photo, require_serial, keep_item
    params: dict = {}  # e.g., {"fee_pct": 15}, {"extended_days": 60}

class PolicyRule(BaseModel):
    name: str
    conditions: list[PolicyRuleCondition]
    actions: list[PolicyRuleAction]
    priority: int = 0

class PolicyDefinition(BaseModel):
    return_window_days: int = 30
    exchange_window_days: int | None = None
    requires_receipt: bool = True
    free_return_shipping: bool = False
    restocking_fee_pct: float = 0.0
    keep_item: dict = {"enabled": False, "max_value": 0}
    rules: list[PolicyRule] = []
```

Endpoints:
- `POST /api/v1/policies` — create policy
- `GET /api/v1/policies` — list policies for tenant
- `GET /api/v1/policies/{id}` — get policy with rules
- `PUT /api/v1/policies/{id}` — replace policy definition
- `DELETE /api/v1/policies/{id}` — deactivate policy
- `POST /api/v1/policies/{id}/evaluate` — dry-run evaluate a return against this policy

**Testing**:
- `Integration: create policy with rules, retrieve and verify JSONB structure`
- `Integration: only one policy can be is_default=true per tenant`
- `Integration: deactivated policy excluded from listing`

---

#### 4.2 — Policy Evaluation Engine

**What**: Implement the rule engine that evaluates return requests against the tenant's active policy.

**Design**:

```python
# src/rlm/services/policy_engine.py
from dataclasses import dataclass

@dataclass
class PolicyEvaluation:
    eligible: bool
    actions: list[PolicyRuleAction]       # Accumulated actions from matched rules
    matched_rules: list[str]              # Names of rules that fired
    return_window_remaining_days: int
    requires_review: bool
    restocking_fee_pct: float
    keep_item_eligible: bool
    rejection_reason: str | None = None

class PolicyEngine:
    def evaluate(
        self,
        policy: PolicyDefinition,
        context: dict,    # Contains: customer_tier, product_category, order_value,
                          #           item_value, reason_code, jurisdiction, return_count_30d,
                          #           order_date, days_since_order
    ) -> PolicyEvaluation:
        """
        Evaluate all rules in priority order (highest priority first).
        Accumulate actions. If any rule action is "reject", the return is ineligible.
        If any rule action is "require_review", flag for manual review.
        """
        # 1. Check return window
        # 2. Evaluate rules in priority order
        # 3. Accumulate actions (extend_window, apply_fee, require_photo, etc.)
        # 4. Return PolicyEvaluation
```

The engine is invoked automatically when a return request is created. The result is stored in the return's audit log and influences auto-approval or manual-review routing.

**Testing**:
- `Unit: return within 30-day window -> eligible`
- `Unit: return outside 30-day window -> ineligible, rejection_reason="outside_return_window"`
- `Unit: VIP customer rule extends window to 60 days -> eligible for day-45 return`
- `Unit: item value > $500 triggers require_review action`
- `Unit: multiple rules fire in priority order, highest priority first`
- `Unit: "reject" action from any rule makes entire return ineligible`
- `Unit: keep_item eligible when item value < threshold and keep_item.enabled`
- `Unit: restocking fee applied correctly based on rule match`
- `Integration: POST /api/v1/returns auto-evaluates policy and stores result`

---

#### 4.3 — Heuristic Fraud Scoring

**What**: Implement initial fraud risk scoring using heuristic signals (no ML yet).

**Design**:

```python
# src/rlm/services/fraud_service.py
@dataclass
class FraudSignal:
    signal_type: str       # high_frequency, value_anomaly, serial_mismatch, reason_pattern, new_account
    severity: str          # low, medium, high, critical
    score_contribution: float
    details: str

@dataclass
class FraudAssessment:
    risk_score: float                # 0.0 to 100.0
    signals: list[FraudSignal]
    recommendation: str              # approve, review, reject

class FraudService:
    # Signal weights
    WEIGHTS = {
        "high_frequency": 25.0,      # >3 returns in 30 days
        "value_anomaly": 20.0,       # Return value > 2x average order value
        "serial_mismatch": 30.0,     # Serial number doesn't match purchase record
        "reason_pattern": 15.0,      # Same reason code >5 times in 90 days
        "new_account": 10.0,         # Account age < 30 days
        "wardrobing": 35.0,          # Pattern: buy, use briefly, return (apparel)
    }

    THRESHOLDS = {
        "approve": 30.0,             # Below this: auto-approve
        "review": 70.0,              # Below this: flag for review
        # Above review threshold: recommend reject
    }

    async def assess(
        self,
        consumer_id: uuid.UUID,
        return_request: ReturnRequest,
        session: AsyncSession,
    ) -> FraudAssessment:
        signals = []
        # 1. Query consumer's return history (last 30/90 days)
        # 2. Check each heuristic signal
        # 3. Compute weighted risk score
        # 4. Store signals in fraud_signal table
        # 5. Return assessment
```

Fraud signals stored in dedicated `fraud_signal` table per data-model-suggestion-1. Each signal has `signal_type`, `severity`, `confidence`, `model_version` (set to "heuristic-v1.0"), and `details`.

**Testing**:
- `Unit: consumer with 0 prior returns -> risk score near 0`
- `Unit: consumer with 5 returns in 30 days -> high_frequency signal fires, score > 25`
- `Unit: return value 3x average order value -> value_anomaly signal fires`
- `Unit: account age < 30 days -> new_account signal fires`
- `Unit: score > 70 -> recommendation is "reject"`
- `Unit: score 30-70 -> recommendation is "review"`
- `Unit: score < 30 -> recommendation is "approve"`
- `Integration: fraud signals persisted to fraud_signal table with correct return_request_id`
- `Integration: consumer.fraud_risk_score updated after assessment`

---

#### 4.4 — Auto-Approval Workflow

**What**: Wire policy evaluation and fraud scoring into the return creation flow for automatic approve/review/reject decisions.

**Design**:

When a return request is created via `POST /api/v1/returns`:
1. Generate RMA number
2. Evaluate policy rules -> `PolicyEvaluation`
3. Run fraud assessment -> `FraudAssessment`
4. Decision logic:
   - If policy says ineligible -> status = `rejected`, store `rejection_reason`
   - If fraud says reject (score > review_threshold) -> status = `requested`, flagged for manual review
   - If policy says `require_review` -> status = `requested`, flagged for manual review
   - If both policy and fraud approve -> status = `approved` (auto-approved)
5. Create audit log entries for each decision step
6. Return the complete return request with status and fraud_risk_score

**Testing**:
- `Integration: eligible return with low fraud score -> auto-approved, status="approved"`
- `Integration: return outside policy window -> auto-rejected, status="rejected"`
- `Integration: eligible return with high fraud score -> status="requested", requires manual review`
- `Integration: policy with require_review rule -> status="requested" regardless of fraud score`
- `Integration: audit log captures policy evaluation and fraud assessment details`

---

## Phase 5: Shipping Integration & Label Generation

### Purpose
Integrate with EasyPost for carrier label generation, implement return shipment tracking, and build the shipping lifecycle. After this phase, approved returns get shipping labels and tracking updates flow through the system.

### Tasks

#### 5.1 — EasyPost Integration Client

**What**: Implement the EasyPost API client for label generation and tracking across UPS, FedEx, USPS, and DHL.

**Design**:

```python
# src/rlm/integrations/easypost.py
from dataclasses import dataclass

@dataclass
class ShippingLabel:
    tracking_number: str
    carrier: str
    service_level: str
    label_url: str          # PDF download URL
    qr_code_url: str | None # QR code for box-free drop-off
    estimated_delivery: datetime | None
    shipping_cost: Decimal
    currency: str

@dataclass
class TrackingEvent:
    status: str             # pre_transit, in_transit, out_for_delivery, delivered, failure
    description: str
    location: str | None
    timestamp: datetime

class EasyPostClient:
    def __init__(self, api_key: str, test_mode: bool = True):
        self.api_key = api_key
        self.test_mode = test_mode

    async def create_return_label(
        self,
        from_address: dict,     # consumer's address
        to_address: dict,       # warehouse address
        parcel: dict,           # weight, dimensions
        carrier: str = "USPS",  # preferred carrier
        service_level: str = "Priority",
    ) -> ShippingLabel:
        """Create a prepaid return shipping label via EasyPost."""

    async def get_tracking(self, tracking_number: str, carrier: str) -> list[TrackingEvent]:
        """Fetch current tracking events for a shipment."""

    async def register_tracking_webhook(self, webhook_url: str) -> str:
        """Register a webhook URL for tracking event callbacks."""
```

**Testing**:
- `Integration (mocked): create_return_label returns ShippingLabel with tracking number and label URL`
- `Integration (mocked): create_return_label with invalid address returns validation error`
- `Integration (mocked): get_tracking returns list of TrackingEvent objects`
- `Unit: ShippingLabel dataclass validates all required fields`

---

#### 5.2 — Return Shipment Model & API

**What**: Implement the return_shipment model and shipping endpoints.

**Design**:

Model per data-model-suggestion-1 `return_shipment` table with status lifecycle:
`pending -> label_created -> picked_up -> in_transit -> delivered -> exception | cancelled`

Endpoints:
- `POST /api/v1/returns/{rma_number}/shipment` — generate shipping label for an approved return
- `GET /api/v1/returns/{rma_number}/shipment` — get shipment details and tracking events
- `POST /api/v1/webhooks/easypost` — webhook receiver for tracking event updates from EasyPost

Shipment creation flow:
1. Verify return status is `approved`
2. Look up tenant's warehouse address
3. Call EasyPost to generate label
4. Store shipment record with tracking number, label URL, QR code URL
5. Update return status to `label_generated`
6. Fire webhook event `return.label_generated`

**Testing**:
- `Integration: POST shipment for approved return -> label generated, status becomes "label_generated"`
- `Integration: POST shipment for non-approved return -> 409 Conflict`
- `Integration: GET shipment returns tracking events`
- `Integration (mocked): EasyPost webhook with valid tracking update -> shipment status updated`
- `Integration: EasyPost webhook with invalid signature -> 401`
- `Integration: return status transitions to "in_transit" when first tracking event received`
- `Integration: return status transitions to "received" when delivered tracking event received`

---

#### 5.3 — Webhook Delivery System

**What**: Implement outbound webhook delivery for tenant integrations with retry logic.

**Design**:

```python
# src/rlm/services/webhook_service.py
class WebhookEvent(BaseModel):
    event_type: str          # return.created, return.approved, return.label_generated,
                             # return.received, inspection.completed, disposition.decided, return.resolved
    timestamp: datetime
    tenant_id: uuid.UUID
    data: dict               # Event-specific payload

class WebhookService:
    MAX_RETRIES = 5
    RETRY_DELAYS = [60, 300, 900, 3600, 14400]  # 1m, 5m, 15m, 1h, 4h

    async def dispatch(self, event: WebhookEvent) -> None:
        """Queue webhook delivery to all registered endpoints for this tenant."""

    async def deliver(self, delivery_id: uuid.UUID) -> bool:
        """Attempt delivery with HMAC-SHA256 signature in X-RLM-Signature header."""
```

Uses Celery task for async delivery. HMAC-SHA256 signature using tenant's webhook secret. Stores delivery attempts in `webhook_delivery` table with response status, body, and retry scheduling.

**Testing**:
- `Integration: dispatch creates webhook_delivery records for each registered endpoint`
- `Integration (mocked): successful delivery updates delivered_at timestamp`
- `Integration (mocked): failed delivery (500 response) schedules retry with correct delay`
- `Integration (mocked): delivery after MAX_RETRIES marks as permanently failed`
- `Unit: HMAC-SHA256 signature computed correctly for known payload and secret`

---

## Phase 6: Inspection, Grading & Disposition

### Purpose
Implement the inspection workflow with photo capture, condition grading, rule-based disposition routing, and the disposition partner management. This is the core operational workflow that processes received returns. After this phase, warehouse operators can inspect returned items, assign condition grades, and the system routes items to the correct disposition channel.

### Tasks

#### 6.1 — Inspection Model & Photo Upload

**What**: Implement inspection and inspection_photo models with S3 photo upload.

**Design**:

Models per data-model-suggestion-1: `inspection` (condition_grade, is_authentic, ai_condition_grade, ai_confidence) and `inspection_photo` (photo_url, photo_type, ai_analysis JSONB).

Condition grades: `new`, `like_new`, `good`, `fair`, `poor`, `damaged`, `counterfeit`.

Photo upload flow:
1. Client requests pre-signed S3 upload URL
2. Client uploads photo directly to S3
3. Client registers the photo with the inspection endpoint
4. Backend stores the photo URL and triggers AI analysis (Phase 8)

Endpoints:
- `POST /api/v1/returns/{rma_number}/items/{item_id}/inspection` — create inspection for a return item
- `PATCH /api/v1/returns/{rma_number}/items/{item_id}/inspection` — update inspection (add grade, notes)
- `GET /api/v1/returns/{rma_number}/items/{item_id}/inspection` — get inspection with photos
- `POST /api/v1/uploads/presign` — get pre-signed URL for photo upload
- `POST /api/v1/returns/{rma_number}/items/{item_id}/inspection/photos` — register uploaded photo

**Testing**:
- `Integration: create inspection with condition_grade "good" -> persisted`
- `Integration: pre-signed URL generation returns valid S3 URL`
- `Integration: register photo links photo to inspection`
- `Integration: inspection with condition_grade "counterfeit" sets is_authentic=false`
- `Integration: return status transitions to "inspecting" when first inspection created`
- `Unit: condition_grade not in allowed list -> 422`

---

#### 6.2 — Disposition Partner Management

**What**: Implement disposition_partner model and CRUD with certification tracking.

**Design**:

Model per data-model-suggestion-1: partner_type (recycler, liquidator, refurbisher, recommerce), certification_type (r2v3, e_stewards, weee_approved, none), certification_expiry.

Endpoints:
- `POST /api/v1/disposition-partners` — create partner
- `GET /api/v1/disposition-partners` — list partners with filtering by type, certification
- `GET /api/v1/disposition-partners/{id}` — get partner details
- `PATCH /api/v1/disposition-partners/{id}` — update partner
- Automatic alerting when certification_expiry is within 30 days

**Testing**:
- `Integration: create partner with R2v3 certification, verify certification_type stored`
- `Integration: list partners filtered by partner_type="recycler" returns correct set`
- `Integration: expired certification flagged in response`

---

#### 6.3 — Rule-Based Disposition Engine

**What**: Implement the initial disposition routing engine using configurable rules (ML-based routing is Phase 8).

**Design**:

```python
# src/rlm/services/disposition_engine.py
@dataclass
class DispositionRecommendation:
    disposition_type: str      # restock, refurbish, liquidate, recycle, donate, destroy, return_to_vendor
    decided_by: str            # "rule_engine"
    confidence: float          # 1.0 for rule-based (deterministic)
    partner_id: uuid.UUID | None
    estimated_recovery: Decimal | None
    reasoning: str             # Human-readable explanation

class DispositionEngine:
    """
    Rule-based disposition routing. Rules evaluated in order:
    1. Counterfeit -> destroy
    2. Damaged + WEEE category -> recycle (certified recycler required)
    3. New/Like New -> restock
    4. Good + refurbishment cost < 40% of unit price -> refurbish
    5. Fair + resale value > liquidation value -> refurbish
    6. Fair/Poor -> liquidate
    7. Default -> liquidate
    """

    async def recommend(
        self,
        return_item: ReturnItem,
        inspection: Inspection,
        product: Product,
        tenant_settings: dict,
    ) -> DispositionRecommendation:
        """Evaluate disposition rules and return recommendation."""
```

Disposition model per data-model-suggestion-1 with `decided_by` (rule_engine, ml_model, manual), `ml_confidence`, `estimated_recovery`, `actual_recovery`.

Endpoints:
- `POST /api/v1/returns/{rma_number}/items/{item_id}/disposition` — create disposition (accepts recommendation or manual override)
- `GET /api/v1/returns/{rma_number}/items/{item_id}/disposition` — get disposition details
- `POST /api/v1/returns/{rma_number}/items/{item_id}/disposition/recommend` — get AI/rule recommendation without applying

When all items in a return have dispositions, return status transitions to `disposition_decided`.

**Testing**:
- `Unit: counterfeit item -> disposition "destroy"`
- `Unit: new condition grade -> disposition "restock"`
- `Unit: good condition + low refurb cost -> disposition "refurbish"`
- `Unit: damaged + WEEE category -> disposition "recycle", partner must have WEEE certification`
- `Unit: poor condition -> disposition "liquidate"`
- `Integration: manual override stores decided_by="manual"`
- `Integration: disposition created -> audit log entry with recommendation details`
- `Integration: all items dispositioned -> return status becomes "disposition_decided"`
- `Unit: estimated_recovery calculation based on condition grade and product price`

---

#### 6.4 — Return Resolution & Financial Settlement

**What**: Implement the return resolution flow (refund, exchange, store credit) and financial tracking.

**Design**:

Resolution flow:
1. All items inspected and dispositioned
2. Calculate refund/credit amount based on:
   - Original item price
   - Restocking fee (from policy)
   - Condition-based deductions
   - Exchange price difference
3. Record resolution type and amounts on return_request
4. Transition status to `resolved`
5. Fire `return.resolved` webhook event

Endpoints:
- `POST /api/v1/returns/{rma_number}/resolve` — resolve the return with financial settlement
- `GET /api/v1/returns/{rma_number}/financials` — get financial summary (refund, credit, recovery)

```python
class ResolutionRequest(BaseModel):
    resolution_type: Literal["refund", "exchange", "store_credit", "keep_item"]
    refund_amount: Decimal | None = None        # If None, calculated from items
    store_credit_amount: Decimal | None = None
    exchange_product_id: uuid.UUID | None = None # For exchange resolution
    notes: str | None = None
```

**Testing**:
- `Integration: resolve return as refund -> refund_amount set, status="resolved"`
- `Integration: resolve as store_credit -> store_credit_amount set`
- `Integration: resolve as exchange -> exchange linked to new order/shipment`
- `Integration: restocking fee deducted from refund per policy`
- `Integration: cannot resolve return that has uninspected items -> 409`
- `Integration: return.resolved webhook fired on resolution`

---

## Phase 7: Consumer Portal & Dashboard Frontend

### Purpose
Build the Next.js frontend with two views: a consumer-facing self-service return portal and a merchant operations dashboard. After this phase, consumers can submit returns through a branded portal, and merchants can manage returns, inspections, and dispositions through a dashboard.

### Tasks

#### 7.1 — Next.js Project Setup & API Client

**What**: Initialise the Next.js frontend with TypeScript, Tailwind CSS, shadcn/ui, and typed API client.

**Design**:

```
frontend/
├── package.json
├── next.config.ts         # API proxy to backend
├── tailwind.config.ts
├── src/
│   ├── app/layout.tsx
│   ├── lib/
│   │   ├── api-client.ts  # Typed fetch wrapper for /api/v1/*
│   │   └── types.ts       # TypeScript types generated from OpenAPI spec
│   └── components/
│       └── ui/            # shadcn/ui components
```

API client uses `fetch` with typed request/response interfaces auto-generated from the backend's OpenAPI spec. Authentication token stored in httpOnly cookie for the dashboard, no auth for the consumer portal (uses tenant slug + order lookup).

**Testing**:
- `E2E: npm run build completes without errors`
- `E2E: npm run dev starts development server on port 3000`
- `Unit: API client types match backend OpenAPI spec`

---

#### 7.2 — Consumer Return Portal

**What**: Build the self-service return submission flow with exchange-first UX.

**Design**:

Portal URL pattern: `/{tenant-slug}/returns`

Flow:
1. **Order Lookup** — consumer enters order number + email to identify their order
2. **Item Selection** — consumer selects items to return, enters reason code and optional photo
3. **Resolution Selection** — exchange-first presentation:
   - Exchange option presented first with product browser
   - Store credit option with bonus incentive (configurable per tenant)
   - Refund option shown last
4. **Confirmation** — review selected items, resolution, and shipping method
5. **Label/QR** — display shipping label download or QR code for drop-off
6. **Tracking** — status page with timeline showing return progress

Pages:
- `/portal/page.tsx` — order lookup form
- `/portal/submit/page.tsx` — multi-step return submission wizard
- `/portal/[rma]/page.tsx` — return status and tracking page

Design:
- Tenant branding loaded from `tenant.settings.branding` (logo, primary color)
- Mobile-first responsive design
- Accessibility: WCAG 2.1 AA compliance
- Reason code dropdown populated from tenant's configured reason codes

**Testing**:
- `E2E (Playwright): complete return submission flow from order lookup to label display`
- `E2E: exchange option presented before refund option`
- `E2E: invalid order number shows error message`
- `E2E: status page shows timeline with correct states`
- `E2E: portal loads with tenant branding (logo, colors)`
- `E2E: mobile viewport renders correctly`

---

#### 7.3 — Merchant Operations Dashboard

**What**: Build the merchant dashboard for managing returns, inspections, and dispositions.

**Design**:

Dashboard URL pattern: `/dashboard/*`

Pages:
- `/dashboard/page.tsx` — overview with KPI cards and return volume chart
- `/dashboard/returns/page.tsx` — return list with filtering, sorting, search
- `/dashboard/returns/[rma]/page.tsx` — return detail with timeline, items, inspection results, disposition
- `/dashboard/inspections/page.tsx` — inspection queue (returns with status "received")
- `/dashboard/analytics/page.tsx` — charts and metrics (Phase 10 detail)
- `/dashboard/policies/page.tsx` — policy editor (visual rule builder)
- `/dashboard/settings/page.tsx` — tenant settings, branding, integrations

KPI cards on overview:
- Returns created (today / 7d / 30d)
- Awaiting inspection (count)
- Average resolution time
- Fraud flags (count requiring review)
- Recovery rate (actual recovery / total return value)

Return detail page:
- Header with RMA number, status badge, consumer info, order link
- Timeline showing all status transitions with timestamps
- Item table with condition grade, disposition, recovery amount
- Inspection photos gallery
- Fraud signals panel (if any)
- Action buttons: Approve, Reject, Inspect, Disposition, Resolve

**Testing**:
- `E2E: login flow -> dashboard loads with KPI cards`
- `E2E: return list displays with correct filtering`
- `E2E: return detail shows timeline, items, and action buttons`
- `E2E: approve button transitions return to "approved" status`
- `E2E: inspection queue shows only "received" returns`

---

#### 7.4 — Policy Editor UI

**What**: Build a visual rule builder for configuring return policies without code.

**Design**:

Visual rule builder component:
- Drag-and-drop rule ordering (priority)
- Condition builder: field selector + operator selector + value input
- Action selector with parameter inputs
- Live preview: "If customer_tier equals VIP, then extend_window by 60 days"
- Save/publish workflow with diff preview

Backed by `PUT /api/v1/policies/{id}` which replaces the full policy definition.

**Testing**:
- `E2E: create new rule with condition and action`
- `E2E: reorder rules by dragging`
- `E2E: save policy and verify via API that JSONB structure is correct`
- `E2E: delete rule and save`

---

## Phase 8: AI/ML Layer — Condition Grading, Disposition ML & Fraud Enhancement

### Purpose
Add the AI-native capabilities that differentiate this platform: computer vision condition grading from inspection photos, ML-driven disposition routing, and enhanced fraud detection. After this phase, inspections are augmented by AI predictions, disposition routing uses ML instead of static rules, and fraud detection goes beyond heuristics.

### Tasks

#### 8.1 — CV Condition Grading Model

**What**: Implement a computer vision model that predicts condition grade from inspection photos.

**Design**:

```python
# src/rlm/ml/condition_grader.py
from dataclasses import dataclass

@dataclass
class GradingResult:
    predicted_grade: str          # new, like_new, good, fair, poor, damaged
    confidence: float             # 0.0 to 1.0
    damage_regions: list[dict]    # [{"x": int, "y": int, "w": int, "h": int, "label": str, "confidence": float}]
    model_version: str

class ConditionGrader:
    """
    Uses a fine-tuned ResNet-50 or EfficientNet-B4 model trained on product condition images.
    Exported to ONNX for inference.

    Training data: labeled inspection photos with condition grades (collected in production).
    Initial model: pre-trained on ImageNet, fine-tuned on synthetic/open condition grading datasets.
    """

    def __init__(self, model_path: str):
        self.session = onnxruntime.InferenceSession(model_path)
        self.model_version = "cv-grading-v1.0"

    def predict(self, image_bytes: bytes) -> GradingResult:
        """
        1. Preprocess: resize to 224x224, normalize
        2. Run inference
        3. Map output logits to condition grades
        4. Run damage detection head for region localization
        5. Return GradingResult
        """

    def predict_batch(self, images: list[bytes]) -> list[GradingResult]:
        """Batch inference for throughput."""
```

Integration point: when inspection photos are registered, a Celery task runs the grader and stores `ai_condition_grade` and `ai_confidence` on the inspection record. The human inspector sees the AI prediction as a suggestion but makes the final decision.

**Testing**:
- `Unit: predict returns GradingResult with valid grade and confidence 0-1`
- `Unit: predict with empty image raises ValueError`
- `Unit (fixture): predict on known "damaged" test image returns "damaged" or "poor" grade`
- `Integration: photo registration triggers async grading task`
- `Integration: grading result stored on inspection record as ai_condition_grade`
- `Unit: damage_regions contain bounding boxes with valid coordinates`

---

#### 8.2 — ML Disposition Model

**What**: Replace the static rule-based disposition engine with an ML model that predicts the most profitable disposition for each item.

**Design**:

```python
# src/rlm/ml/disposition_model.py
@dataclass
class DispositionPrediction:
    disposition_type: str              # restock, refurbish, liquidate, recycle, donate
    confidence: float
    estimated_recovery: Decimal
    alternative_dispositions: list[dict]  # [{type, confidence, estimated_recovery}, ...]
    model_version: str

class DispositionModel:
    """
    Gradient-boosted tree model (XGBoost or LightGBM) trained on:
    - Product features: category, brand, unit_price, WEEE eligibility
    - Condition features: condition_grade, damage regions, authenticity
    - Market features: current resale value, demand score
    - Historical: past disposition outcomes and actual recovery for similar items

    Target: disposition_type that maximises actual_recovery
    """

    FEATURE_COLUMNS = [
        "unit_price", "condition_grade_encoded", "category_encoded",
        "brand_encoded", "weight_grams", "days_since_purchase",
        "weee_eligible", "right_to_repair", "consumer_tier_encoded",
        "avg_recovery_rate_for_category", "refurb_cost_estimate",
    ]

    def predict(self, features: dict) -> DispositionPrediction:
        """Predict optimal disposition and estimated recovery."""

    def retrain(self, training_data: pd.DataFrame) -> dict:
        """Retrain on accumulated disposition outcome data. Returns metrics."""
```

The disposition engine (`disposition_engine.py`) is updated to:
1. First try the ML model
2. If ML confidence < 0.6, fall back to rule-based
3. Always provide the ML prediction alongside rule-based as `decided_by` = "ml_model" or "rule_engine"

**Testing**:
- `Unit: predict returns DispositionPrediction with valid type and recovery estimate`
- `Unit: feature encoding handles missing values (unknown brand, missing weight)`
- `Unit: alternative_dispositions sorted by confidence descending`
- `Integration: ML prediction stored with decided_by="ml_model" and ml_confidence`
- `Integration: low-confidence ML prediction triggers rule-based fallback`
- `Unit: retrain on fixture data produces model with accuracy > baseline`

---

#### 8.3 — Enhanced Fraud Detection

**What**: Add image-based and behavioral fraud signals beyond simple heuristics.

**Design**:

New fraud signals:
- **Image-based**: CV analysis of return photos for signs of tampering, tag removal, use marks on "unused" claims
- **Behavioral clustering**: group consumers with similar return patterns (same products, same reason codes, coordinated timing)
- **Address network**: flag consumers sharing shipping addresses with known fraudulent accounts
- **Value-pattern anomaly**: detect systematic returns just below review thresholds

```python
# Enhanced fraud_service.py additions
class EnhancedFraudService(FraudService):
    ADDITIONAL_WEIGHTS = {
        "image_tampering": 25.0,
        "behavioral_cluster": 20.0,
        "address_network": 30.0,
        "threshold_gaming": 15.0,
    }

    async def analyze_images(self, photos: list[InspectionPhoto]) -> list[FraudSignal]:
        """CV analysis for fraud indicators in return photos."""

    async def detect_behavioral_cluster(
        self, consumer_id: uuid.UUID, session: AsyncSession
    ) -> list[FraudSignal]:
        """Identify coordinated return patterns across consumers."""
```

**Testing**:
- `Unit (fixture): image with visible use marks on "new" item -> image_tampering signal`
- `Unit: consumer with 3 returns of same product in 60 days -> behavioral_cluster signal`
- `Integration: fraud score recalculated after image analysis adds new signals`
- `Unit: return value $99 when review threshold is $100 -> threshold_gaming signal`

---

#### 8.4 — Refurbishment Cost Estimation

**What**: Predict refurbishment cost from inspection photos and product data to drive accept/reject decisions on warranty returns.

**Design**:

```python
# src/rlm/ml/refurb_estimator.py
@dataclass
class RefurbEstimate:
    estimated_cost: Decimal
    confidence: float
    cost_breakdown: dict       # {"parts": Decimal, "labor": Decimal, "testing": Decimal}
    profitable: bool           # estimated_cost < (unit_price * recovery_rate_threshold)
    model_version: str

class RefurbCostEstimator:
    """
    Regression model predicting refurbishment cost from:
    - Damage region count and severity from CV model
    - Product category and historical repair costs
    - Parts availability and cost
    """

    def estimate(self, inspection: Inspection, product: Product) -> RefurbEstimate:
        """Estimate refurbishment cost and profitability."""
```

Integrated into disposition recommendation: if `refurb_estimate.profitable` is False, the engine routes to liquidate/recycle instead of refurbish.

**Testing**:
- `Unit: minor damage on low-cost item -> profitable=False (refurb cost > item value)`
- `Unit: minor damage on high-value item -> profitable=True`
- `Unit: cost_breakdown components sum to estimated_cost`
- `Integration: disposition recommendation uses refurb estimate when disposition_type would be "refurbish"`

---

## Phase 9: E-Commerce Integration & Webhooks

### Purpose
Build the integration layer for syncing with e-commerce platforms (Shopify first, then Magento/WooCommerce), including order import, product sync, and return status callbacks. After this phase, merchants can connect their Shopify store and returns are automatically linked to orders.

### Tasks

#### 9.1 — Integration Configuration & Credential Management

**What**: Implement the integration model with encrypted credential storage and tenant-scoped configuration.

**Design**:

Integration model per data-model-suggestion-1 with `platform`, `integration_type`, `credentials` (JSONB, encrypted at application layer), `webhook_url`, `last_sync_at`.

```python
# src/rlm/integrations/ecommerce_base.py
from abc import ABC, abstractmethod

class EcommerceConnector(ABC):
    @abstractmethod
    async def sync_orders(self, since: datetime | None = None) -> list[dict]:
        """Fetch new/updated orders from the e-commerce platform."""

    @abstractmethod
    async def sync_products(self, since: datetime | None = None) -> list[dict]:
        """Fetch new/updated products from the platform."""

    @abstractmethod
    async def notify_return_status(self, rma_number: str, status: str, details: dict) -> bool:
        """Push return status update back to the e-commerce platform."""

    @abstractmethod
    async def process_refund(self, order_id: str, amount: Decimal, currency: str) -> dict:
        """Issue refund via the e-commerce platform's payment system."""
```

Endpoints:
- `POST /api/v1/integrations` — create integration with credentials
- `GET /api/v1/integrations` — list integrations for tenant
- `PATCH /api/v1/integrations/{id}` — update configuration
- `POST /api/v1/integrations/{id}/test` — test connectivity
- `POST /api/v1/integrations/{id}/sync` — trigger manual sync

**Testing**:
- `Integration: create integration stores encrypted credentials`
- `Integration: test endpoint verifies connectivity (mocked)`
- `Integration: credentials not returned in GET response (redacted)`

---

#### 9.2 — Shopify Connector

**What**: Implement the Shopify integration for order sync, product sync, and return notifications.

**Design**:

```python
# src/rlm/integrations/shopify.py
class ShopifyConnector(EcommerceConnector):
    def __init__(self, shop_domain: str, api_key: str, api_secret: str):
        self.shop_domain = shop_domain
        self.api_key = api_key
        self.api_secret = api_secret
        self.base_url = f"https://{shop_domain}/admin/api/2025-01"

    async def sync_orders(self, since: datetime | None = None) -> list[dict]:
        """
        GET /orders.json with updated_at_min filter
        Maps Shopify order -> OriginalOrder + Consumer + ReturnItems
        """

    async def sync_products(self, since: datetime | None = None) -> list[dict]:
        """
        GET /products.json with updated_at_min filter
        Maps Shopify product -> Product (with SKU, price, category)
        """

    async def register_webhooks(self) -> None:
        """
        Register Shopify webhooks for:
        - orders/create, orders/updated, orders/fulfilled
        - products/create, products/update
        """

    async def process_refund(self, order_id: str, amount: Decimal, currency: str) -> dict:
        """POST /orders/{id}/refunds.json"""
```

Incoming webhook handler at `POST /api/v1/webhooks/shopify` with HMAC-SHA256 verification using Shopify's shared secret.

Periodic sync via Celery beat task (configurable interval, default 15 minutes).

**Testing**:
- `Integration (mocked): sync_orders maps Shopify order to OriginalOrder correctly`
- `Integration (mocked): sync_products maps Shopify product to Product with SKU and price`
- `Integration (mocked): Shopify webhook with valid HMAC -> order created/updated`
- `Integration (mocked): Shopify webhook with invalid HMAC -> 401`
- `Integration (mocked): process_refund calls Shopify refund API with correct amount`
- `Unit: Shopify order mapping handles missing fields gracefully (nullable address, etc.)`

---

#### 9.3 — Consumer Portal API (Shopify-Connected)

**What**: Extend the consumer portal API to support order lookup via Shopify for connected tenants.

**Design**:

Portal API endpoint for order lookup:
- `POST /api/portal/{tenant-slug}/lookup` — look up order by email + order number
  - If tenant has Shopify integration: fetch from Shopify if not in local DB
  - If no integration: look up from local `original_order` table

Return submission via portal:
- `POST /api/portal/{tenant-slug}/returns` — create return from consumer portal
  - No authentication required (identified by order + email)
  - Rate limited per IP and email
  - Policy evaluation and fraud scoring still run

**Testing**:
- `Integration: portal lookup with valid order returns order with items`
- `Integration: portal lookup with invalid order returns 404`
- `Integration: portal return creation generates RMA and triggers policy evaluation`
- `Integration: rate limiting blocks excessive requests from same IP`

---

## Phase 10: Analytics, Carbon Footprint & Sustainability

### Purpose
Build the operational analytics dashboard, carbon footprint tracking per return, and sustainability reporting. After this phase, merchants have visibility into return metrics, environmental impact, and disposition outcomes.

### Tasks

#### 10.1 — Operational Analytics API

**What**: Implement aggregation queries for return volume, resolution time, and disposition outcomes.

**Design**:

```python
# Pydantic response schemas
class DailyMetric(BaseModel):
    date: date
    returns_created: int
    returns_resolved: int
    avg_resolution_hours: float | None
    refund_total: Decimal
    store_credit_total: Decimal
    recovery_total: Decimal

class DispositionBreakdown(BaseModel):
    disposition_type: str
    count: int
    total_recovery: Decimal
    avg_recovery_rate: float    # actual_recovery / unit_price

class TopReturnedProduct(BaseModel):
    product_id: uuid.UUID
    sku: str
    name: str
    return_count: int
    return_rate: float          # returns / orders

class AnalyticsSummary(BaseModel):
    period_start: datetime
    period_end: datetime
    total_returns: int
    total_resolved: int
    avg_resolution_hours: float
    total_refunded: Decimal
    total_store_credit: Decimal
    total_recovery: Decimal
    recovery_rate: float
    fraud_flags_count: int
    top_reason_codes: list[dict]
    disposition_breakdown: list[DispositionBreakdown]
    top_returned_products: list[TopReturnedProduct]
    daily_metrics: list[DailyMetric]
```

Endpoints:
- `GET /api/v1/analytics/summary?period=7d|30d|90d|custom&start=&end=` — aggregated metrics
- `GET /api/v1/analytics/daily?start=&end=` — daily time series
- `GET /api/v1/analytics/dispositions?start=&end=` — disposition breakdown
- `GET /api/v1/analytics/products?sort=return_count&limit=20` — top returned products

**Testing**:
- `Integration: summary for 30d period returns correct counts`
- `Integration: daily metrics match sum of individual return records`
- `Integration: disposition breakdown percentages sum to 100%`
- `Integration: top returned products sorted by return_count descending`
- `Integration: tenant isolation — analytics only include tenant's own data`

---

#### 10.2 — Carbon Footprint Calculation

**What**: Implement carbon footprint estimation per return based on shipping distance, processing, and packaging.

**Design**:

```python
# src/rlm/services/carbon_service.py
@dataclass
class CarbonFootprint:
    transport_kg_co2: Decimal    # Based on distance and carrier mode
    processing_kg_co2: Decimal   # Fixed per-unit processing cost
    packaging_kg_co2: Decimal    # Based on packaging weight
    total_kg_co2: Decimal
    calculation_method: str      # "estimated" or "carrier_reported"
    avoided_kg_co2: Decimal      # CO2 avoided by refurbish/restock vs. new manufacture

class CarbonService:
    # Emission factors (kg CO2 per km per kg)
    EMISSION_FACTORS = {
        "ground": 0.0001,        # Truck transport
        "air": 0.0006,           # Air freight
        "last_mile": 0.0002,     # Local delivery
    }

    PROCESSING_FACTORS = {
        "inspection": 0.05,       # kg CO2 per item inspected
        "refurbishment": 0.15,    # kg CO2 per item refurbished
        "recycling": 0.10,        # kg CO2 per item recycled
    }

    async def calculate(
        self,
        return_request: ReturnRequest,
        shipment: ReturnShipment,
        dispositions: list[Disposition],
    ) -> CarbonFootprint:
        """
        1. Estimate transport distance from ship_from to ship_to addresses
        2. Calculate transport emissions based on distance, weight, and mode
        3. Add processing emissions based on disposition type
        4. Calculate avoided emissions for restock/refurbish dispositions
        """
```

Carbon footprint stored in `carbon_footprint` table per data-model-suggestion-1. Calculated asynchronously via Celery task after return is resolved.

Endpoints:
- `GET /api/v1/returns/{rma_number}/carbon` — carbon footprint for a specific return
- `GET /api/v1/analytics/carbon?period=30d` — aggregated carbon metrics for tenant

**Testing**:
- `Unit: 500km ground transport with 1kg parcel -> ~0.05 kg CO2`
- `Unit: restock disposition calculates avoided emissions correctly`
- `Unit: total_kg_co2 = transport + processing + packaging`
- `Integration: carbon footprint record created after return resolved`
- `Integration: analytics carbon endpoint aggregates across returns`

---

#### 10.3 — Analytics Dashboard UI

**What**: Build the analytics page in the merchant dashboard with charts and KPI cards.

**Design**:

Charts (using Recharts or Chart.js via React wrapper):
- Return volume time series (line chart, daily)
- Disposition breakdown (pie/donut chart)
- Resolution time distribution (histogram)
- Recovery rate trend (line chart)
- Top return reasons (horizontal bar chart)
- Carbon footprint trend (area chart, cumulative)

KPI cards:
- Total returns (period)
- Average resolution time
- Recovery rate (actual recovery / item value)
- Carbon footprint (total kg CO2)
- Fraud detection rate

**Testing**:
- `E2E: analytics page loads with charts populated from API data`
- `E2E: period selector (7d, 30d, 90d) updates all charts`
- `E2E: disposition pie chart shows correct percentages`
- `E2E: carbon footprint card shows total kg CO2 for period`

---

## Phase 11: Warranty, Repair & Cross-Border

### Purpose
Add warranty lifecycle management, depot repair workflow, and cross-border return support with customs documentation. These are v1.1 features that extend the platform for B2B and international use cases.

### Tasks

#### 11.1 — Warranty Model & Claims Processing

**What**: Implement warranty tracking and warranty-linked return/repair workflows.

**Design**:

Warranty model per data-model-suggestion-1: `warranty_type` (manufacturer, extended, third_party), `start_date`, `end_date`, `serial_number`, `status` (active, expired, claimed, voided).

When a return_type is `warranty_claim`:
1. Look up active warranty by product + serial number
2. Validate claim is within warranty period
3. If valid, bypass policy window check
4. Route to repair workflow instead of standard disposition
5. Update warranty status to `claimed`

Endpoints:
- `POST /api/v1/warranties` — create warranty record
- `GET /api/v1/warranties` — list warranties with filtering
- `GET /api/v1/warranties/{id}` — get warranty details
- `POST /api/v1/returns` with `return_type: "warranty_claim"` — warranty-aware return flow

**Testing**:
- `Integration: warranty claim within valid period -> auto-approved`
- `Integration: warranty claim after expiry -> rejected with reason "warranty_expired"`
- `Integration: warranty status transitions to "claimed" after claim created`
- `Unit: warranty lookup by serial_number + product_id`

---

#### 11.2 — Depot Repair Workflow

**What**: Implement multi-stage repair job tracking with parts inventory.

**Design**:

Repair job model per data-model-suggestion-1: status lifecycle `pending -> diagnosed -> parts_ordered -> in_repair -> testing -> completed | failed | cancelled`.

Repair parts model: `part_sku`, `part_name`, `quantity`, `unit_cost`, `source` (new, harvested, refurbished).

Right to Repair compliance: `right_to_repair_eligible` flag from product, documentation generation for EU compliance.

Endpoints:
- `POST /api/v1/returns/{rma_number}/items/{item_id}/repair` — create repair job
- `PATCH /api/v1/returns/{rma_number}/items/{item_id}/repair` — update repair status
- `POST /api/v1/returns/{rma_number}/items/{item_id}/repair/parts` — add parts to repair
- `GET /api/v1/repairs` — list all repairs with status filtering (for repair technician queue)

**Testing**:
- `Integration: create repair job -> status "pending"`
- `Integration: update with diagnosis -> status "diagnosed", estimated_cost set`
- `Integration: add parts -> parts linked to repair, costs accumulated`
- `Integration: complete repair -> actual_cost calculated from parts + labor`
- `Integration: right_to_repair flag generates compliance documentation`
- `Integration: repair status lifecycle enforced (cannot skip from pending to completed)`

---

#### 11.3 — Cross-Border Returns & Customs

**What**: Implement cross-border return support with automated customs documentation.

**Design**:

Cross-border detection: when consumer's country differs from warehouse country, flag as cross-border.

Customs documentation stored in `return_shipment.customs_declaration` (JSONB):
```json
{
    "declaration_number": "CUS-2026-001",
    "origin_country": "DE",
    "destination_country": "US",
    "hs_codes": [{"sku": "WIDGET-BLU-M", "hs_code": "8471.30", "value": 299.99}],
    "total_declared_value": 299.99,
    "currency": "USD",
    "duty_drawback_eligible": true
}
```

HS code mapping stored per product or product category.

Endpoints:
- `POST /api/v1/returns/{rma_number}/shipment` — enhanced to generate customs docs for cross-border
- `GET /api/v1/returns/{rma_number}/customs` — get customs documentation

**Testing**:
- `Integration: cross-border return generates customs declaration`
- `Integration: HS codes looked up from product catalog`
- `Unit: duty_drawback_eligible set for EU->EU returns`
- `Integration: domestic return does not generate customs docs`

---

## Phase 12: Chatbot, Advanced Features & Production Hardening

### Purpose
Add conversational return initiation, production security hardening, and operational tooling. After this phase, the platform is production-ready with all MVP and v1.1 features.

### Tasks

#### 12.1 — Conversational Return Initiation

**What**: Implement a chatbot that allows consumers to initiate returns via natural language.

**Design**:

```python
# Uses Claude API for natural language understanding
class ReturnChatbot:
    SYSTEM_PROMPT = """You are a returns assistant for {merchant_name}.
    Help the customer initiate a return. Collect:
    1. Order number
    2. Which item(s) to return
    3. Reason for return
    4. Preferred resolution (exchange, store credit, or refund)

    When you have all information, call the create_return tool.
    Be empathetic and professional. Suggest exchanges before refunds.
    """

    TOOLS = [
        {
            "name": "lookup_order",
            "description": "Look up an order by order number and email",
            "input_schema": {
                "type": "object",
                "properties": {
                    "order_number": {"type": "string"},
                    "email": {"type": "string", "format": "email"}
                },
                "required": ["order_number", "email"]
            }
        },
        {
            "name": "create_return",
            "description": "Create a return request",
            "input_schema": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string"},
                    "items": {"type": "array", "items": {
                        "type": "object",
                        "properties": {
                            "product_id": {"type": "string"},
                            "reason_code": {"type": "string"},
                            "reason_detail": {"type": "string"}
                        }
                    }},
                    "resolution_type": {"type": "string", "enum": ["exchange", "store_credit", "refund"]}
                }
            }
        }
    ]
```

Endpoints:
- `POST /api/portal/{tenant-slug}/chat` — send message, receive assistant response
- Stateful conversation via session ID
- Tool use for order lookup and return creation

**Testing**:
- `Integration (mocked LLM): "I want to return my order ORD-12345" -> asks for email`
- `Integration (mocked LLM): complete conversation flow -> return created with correct details`
- `Integration: chatbot-initiated return has initiated_via="chatbot"`
- `Unit: system prompt includes tenant name`

---

#### 12.2 — Production Security Hardening

**What**: Implement OWASP API Security Top 10 protections, rate limiting, and security headers.

**Design**:

Protections per OWASP API Security Top 10 (2023):
- **BOLA (A01)**: All queries scoped to tenant_id from JWT; no direct object reference without tenant check
- **Broken Authentication (A02)**: Password hashing with bcrypt, JWT rotation, API key scoping
- **Excessive Data Exposure (A03)**: Response schemas explicitly define returned fields; no model dump
- **Rate Limiting (A04)**: Per-IP and per-tenant rate limits via Redis sliding window
- **Function-Level Auth (A05)**: role-based access via require_role dependency
- **Mass Assignment (A06)**: Pydantic schemas define allowed input fields explicitly
- **Security Misconfiguration (A07)**: CORS restricted, debug off in production, no stack traces
- **Injection (A08)**: Parameterised queries via SQLAlchemy, no raw SQL with user input
- **Improper Inventory (A09)**: OpenAPI spec only exposes public endpoints
- **Unsafe Consumption (A10)**: Validate all webhook payloads, verify HMAC signatures

Rate limiting middleware:
```python
# 100 requests per minute per IP for portal
# 1000 requests per minute per API key for integrations
# 10 requests per minute per email for return creation
```

Security headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, `Content-Security-Policy`.

**Testing**:
- `Integration: BOLA — tenant A cannot access tenant B data via direct ID`
- `Integration: rate limiting returns 429 after threshold exceeded`
- `Integration: security headers present on all responses`
- `Integration: expired JWT returns 401, not 500`
- `Integration: SQL injection attempt in query parameter returns 422`

---

#### 12.3 — Data Privacy & GDPR/CCPA Compliance

**What**: Implement data subject access and deletion requests for GDPR and CCPA compliance.

**Design**:

Data deletion request model per data-model-suggestion-1. When a consumer requests deletion:
1. Record the request in `data_deletion_request` table
2. Anonymise consumer PII: replace email, name, phone with hashed values
3. Retain return records (anonymised) for business reporting
4. Remove inspection photos containing identifiable information
5. Mark consumer record with `data_retention_expiry`

Endpoints:
- `POST /api/portal/{tenant-slug}/data-request` — consumer submits data access/deletion request
- `GET /api/v1/data-requests` — admin view of pending requests
- `POST /api/v1/data-requests/{id}/process` — process the request

**Testing**:
- `Integration: deletion request anonymises consumer email and name`
- `Integration: anonymised consumer's returns still appear in aggregate analytics`
- `Integration: inspection photos for anonymised consumer are deleted from S3`
- `Integration: data access request returns all consumer data as JSON download`

---

#### 12.4 — Deployment Configuration & Monitoring

**What**: Production Docker configuration, health checks, structured logging, and observability.

**Design**:

Production Dockerfile with multi-stage build (builder + runtime). Structured JSON logging via `structlog`. Health check endpoints:
- `GET /health` — basic liveness (always returns 200)
- `GET /health/ready` — readiness (checks DB, Redis, S3 connectivity)

```python
# Structured logging configuration
import structlog
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer(),
    ],
    logger_factory=structlog.PrintLoggerFactory(),
)
```

Metrics endpoint for Prometheus scraping (optional). Request ID middleware for correlation. Database connection pool monitoring.

**Testing**:
- `Integration: /health/ready returns 200 when all dependencies are up`
- `Integration: /health/ready returns 503 when database is down`
- `Integration: structured logs contain request_id, tenant_id, and duration`
- `E2E: production Docker build succeeds and container starts`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (6 tasks)
    │
Phase 2: Core Data Model (6 tasks) ─── requires Phase 1
    │
Phase 3: Auth & API Layer (4 tasks) ─── requires Phase 2
    │
    ├── Phase 4: Policy & Fraud (4 tasks) ─── requires Phase 3
    │       │
    │       └── Phase 6: Inspection & Disposition (4 tasks) ─── requires Phase 4
    │               │
    │               ├── Phase 8: AI/ML Layer (4 tasks) ─── requires Phase 6
    │               │
    │               └── Phase 10: Analytics & Carbon (3 tasks) ─── requires Phase 6
    │                       │
    │                       └── Phase 11: Warranty, Repair & Cross-Border (3 tasks) ─── requires Phase 10
    │
    ├── Phase 5: Shipping Integration (3 tasks) ─── requires Phase 3
    │       │
    │       └── Phase 9: E-Commerce Integration (3 tasks) ─── requires Phase 5
    │
    ├── Phase 7: Frontend (4 tasks) ─── requires Phase 3
    │       │
    │       └── Phase 12: Chatbot & Production (4 tasks) ─── requires Phases 7, 8, 9, 10
    │
    └── Parallelism opportunities:
        - Phases 4, 5, 7 can begin concurrently after Phase 3
        - Phases 8, 10 can proceed concurrently after Phase 6
        - Phase 9 can proceed concurrently with Phases 6, 7, 8
```

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with no TODOs in production code paths.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. Ruff linting passes with zero violations (`ruff check src/ tests/`).
5. Ruff formatting passes (`ruff format --check src/ tests/`).
6. mypy strict mode passes (`mypy --strict src/`).
7. Docker build succeeds (`docker compose build`).
8. All services start and health checks pass (`docker compose up` + `curl /health`).
9. New API endpoints appear in auto-generated OpenAPI spec at `/openapi.json`.
10. Database migrations run cleanly on empty database (`alembic upgrade head`).
11. New configuration options have sensible defaults and are documented in `.env.example`.
12. Audit log entries are generated for all state-changing operations.
13. Tenant isolation verified — no cross-tenant data leakage in new endpoints.
14. E2E tests pass for any new frontend pages (Phases 7+).
