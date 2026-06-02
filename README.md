# PulzeCure

> A multi-tenant clinic booking & management SaaS platform — Spring Boot backend.

![Java](https://img.shields.io/badge/Java-21-007396?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.5-6DB33F?logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-4169E1?logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-Proprietary-lightgrey)

PulzeCure is the backend service powering a SaaS platform where individual clinics sign up, get their own isolated workspace (reached via a dedicated subdomain), and manage doctors, patients, appointment sessions, and subscriptions. Each clinic operates as a fully isolated **tenant** while sharing a single, centrally managed deployment.

---

## Highlights

- **Subdomain-based multi-tenancy** — every request is resolved to its owning clinic before it reaches application logic, with tenant data scoped per request.
- **Stateless JWT security** — access/refresh token flow, multi-role users, and a dedicated role-selection step for users belonging to more than one clinic.
- **Self-service onboarding** — OTP-verified clinic registration backed by SMS.
- **Recurring appointment engine** — doctor sessions are generated from recurrence rules by a scheduled background job.
- **Subscription lifecycle** — tiered plans with feature gating and automated expiry handling.
- **Production-minded foundations** — Flyway-managed schema, Redis + Caffeine caching, Bucket4j API rate limiting, and AWS S3 file storage.

---

## Architecture

### Multi-tenancy

PulzeCure uses a **shared-database, tenant-scoped** model with **subdomain-based resolution**:

1. A `TenantFilter` (`OncePerRequestFilter`, runs first in the chain) inspects the incoming request host.
2. Requests to the bare application domain are treated as **platform-level** traffic (e.g. clinic registration, admin) and run without a tenant.
3. Requests to a clinic subdomain are resolved to a `Clinic` (the tenant). If no clinic matches, the request is rejected with `404`.
4. The resolved tenant is held in a request-scoped `TenantContext` (`ThreadLocal`) and **always cleared** at the end of the request to prevent leakage between threads.

A dedicated security filter clears any lingering authentication/tenant context as an extra safety net.

### Modular design

The codebase is organised **by feature**, each module owning its own `controller`, `service`, `dto`, `entity`, and `repository` layers:

```
auth · otp · tenant · clinic · user · doctor · patient
appointment · subscription · sms · storage · admin
security · config · common · exception · util
```

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Java 21 |
| Framework | Spring Boot 3.5 (Web, Data JPA, Security, Validation) |
| Persistence | Hibernate / JPA, PostgreSQL 18 |
| Migrations | Flyway |
| Caching | Redis (Spring Data Redis), Caffeine |
| Auth | Spring Security + JWT (jjwt) |
| Rate limiting | Bucket4j |
| File storage | AWS S3 (AWS SDK v2) |
| Build | Maven (wrapper included) |
| Local infra | Docker Compose (PostgreSQL + Redis) |

---

## Domain Model

| Entity | Purpose |
|--------|---------|
| `Clinic` | The tenant; root of every clinic's isolated data |
| `User` / `UserClinicRole` | Accounts and their roles within one or more clinics |
| `Doctor` / `Specialization` | Practitioners and their fields |
| `Patient` | Patient records, scoped to a clinic |
| `DoctorSession` / `SessionRecurrence` / `Booking` | Bookable sessions, their recurrence rules, and patient bookings |
| `Subscription` / `PlanConfig` / `PlanFeature` | Tiered plans, feature gating, and billing state |
| `SmsUsage` | Per-clinic SMS consumption tracking |

---

## API Surface

All routes are served under the `/api` context path. Tenant-scoped routes are reached through the relevant clinic subdomain.

| Group | Base path | Responsibility |
|-------|-----------|----------------|
| Auth | `/auth` | Login, token refresh, role selection |
| OTP | `/otp` | Request / verify one-time passwords |
| Clinics | `/clinics` | Clinic registration & management |
| Clinic doctors | `/clinics/{clinicId}/doctors` | Doctors within a clinic |
| Clinic users | `/clinics/{clinicId}/users` | Staff / user management within a clinic |
| Doctors | `/doctors` | Doctor profiles |
| Appointments | `/appointments` | Sessions & bookings |
| Storage | `/upload` | File uploads to S3 |
| Admin | `/admin/users` | Platform-level administration |

---

## Getting Started

### Prerequisites

- JDK 21
- Docker & Docker Compose
- (Maven not required — use the bundled `./mvnw` wrapper)

### 1. Clone

```bash
git clone <your-repo-url>
cd pulzecure
```

### 2. Start local infrastructure

PostgreSQL and Redis are provided via Docker Compose. Provide database credentials (e.g. in a local `.env` file, which is git-ignored):

```bash
# .env
DB_USER=pulzecure
DB_PASSWORD=change-me
```

```bash
docker compose up -d
```

This starts PostgreSQL on host port **5434** and Redis on **6379**.

### 3. Configure the application

Provide the configuration below via environment variables or a local (untracked) `application-dev.yml`. **Never commit real secrets** (see [Security](#security)).

| Key | Example / Notes |
|-----|-----------------|
| `SPRING_PROFILES_ACTIVE` | `dev` |
| `spring.datasource.url` | `jdbc:postgresql://localhost:5434/pulzecure` |
| `spring.datasource.username` / `password` | match your `.env` |
| `spring.data.redis.host` / `port` | `localhost` / `6379` |
| `app.domain` | `localhost` (the bare platform domain) |
| `app.jwt.secret` | long random secret |
| `app.jwt.expiration` | token TTL in ms (e.g. `10800000` = 3h) |
| `app.otp.secret` | OTP signing secret |
| `aws.s3.region` | e.g. `ap-south-1` |
| `aws.s3.bucket-name` | your S3 bucket |
| `aws.s3.access-key` / `secret-key` | IAM credentials |
| `server.port` | `9090` |

### 4. Run

```bash
./mvnw spring-boot:run
```

The API will be available at **`http://localhost:9090/api`**.

### 5. Build

```bash
./mvnw clean package
```

---

## Database & Migrations

Schema is owned and versioned by **Flyway** (`src/main/resources/db/migration`) and applied automatically on startup. JPA runs with `ddl-auto: validate`, so the schema is never modified by Hibernate — migrations are the single source of truth.

```
V1__init.sql                  core schema
V2__seed_plans.sql            subscription plan seed data
V3__seed_specializations.sql  doctor specialization seed data
V4__appointments.sql          appointment / session schema
```

---

## Background Jobs

| Job | Responsibility |
|-----|----------------|
| `RecurrenceEngineJob` | Generates upcoming doctor sessions from recurrence rules |
| `SubscriptionExpiryJob` | Detects and processes expired subscriptions |

---

## Project Structure

```
src/main/java/com/bootloaderlabs/pulzecure/
├── PulzecureApplication.java
├── auth/          # login, JWT issuance, role selection
├── otp/           # OTP request & verification
├── tenant/        # TenantFilter, TenantContext, Clinic resolution
├── clinic/        # clinic registration & management
├── user/          # users and per-clinic roles
├── doctor/        # doctors & specializations
├── patient/       # patient records
├── appointment/   # sessions, recurrence, bookings, scheduled jobs
├── subscription/  # plans, features, expiry job
├── sms/           # SMS sending & usage tracking
├── storage/       # AWS S3 uploads
├── admin/         # platform administration
├── security/      # SecurityConfig, JWT filters
├── config/        # Jackson, S3, web configuration
├── common/        # shared API response types
├── exception/     # global exception handling
└── util/
```

---

## Security

- Authentication is stateless via JWT; passwords and secrets are never logged.
- Tenant context is request-scoped and cleared after every request.
- API endpoints are protected by Bucket4j rate limiting.

> **Important:** Configuration files containing real credentials (database, JWT/OTP secrets, AWS keys) must **not** be committed to version control. Keep them in a local, git-ignored file or inject them as environment variables, and rotate any secret that has ever been committed.

---

## License

Proprietary — All Rights Reserved. © M. J. Rohan.

This source code is not licensed for redistribution or reuse without explicit permission.

---

## Author

**M. J. Rohan** — Full-Stack Software Engineer
React & TypeScript · Spring Boot & Java
