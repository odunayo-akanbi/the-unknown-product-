# MediLink — System Architecture (Technical Blueprint)

> Technical blueprint for the MediLink MVP:
> - Core goal: universal Health ID + consented access + emergency QR access
> - Intended audience: backend/frontend engineers, infrastructure engineers, security reviewers

---

## Overview — high level
MediLink is a modular, API-first platform consisting of:

1. **Mobile/Web Clients** (React Native / React)  
2. **API Gateway & Backend** (Node.js + Express / NestJS)  
3. **Data Stores** (Postgres for relational data; Redis for caching, queue coordination)  
4. **Interoperability Layer** (FHIR-compatible service + HealthLink API)  
5. **Auth & Consent Service** (OAuth2 + JWT + consent tokening)  
6. **Optional**: FHIR Server (HAPI FHIR or cloud FHIR service) for complex clinical data exchange

---

## Component Architecture

[Mobile / Web Clients] <-HTTPS-> [API Gateway (Express)] <-internal RPC/HTTPS-> [Service Layer: Auth, HealthID, Records, Payments]
--> [PostgreSQL (Core Data)]
--> [Redis (cache, sessions)]
--> [FHIR Layer / HAPI FHIR]
--> [Integration: EMR adapters, Payment Gateways (Flutterwave/Paystack), SMS gateway]


---

## Technology Stack (recommended)

- **Frontend (mobile-first):** React Native (Expo or bare), React (web admin).  
- **API / Backend:** Node.js (NestJS or Express) with TypeScript.  
- **Database:** PostgreSQL (relational patient & provider data).  
- **Cache & Queue:** Redis (caching, short-lived consent token store); RabbitMQ or Redis Streams for async tasks.  
- **FHIR / Interop:** HAPI FHIR (self-hosted) or AWS HealthLake / other managed FHIR service (if available).  
- **Auth & Security:** OAuth2 (authorization), JWT for short-lived tokens, TLS everywhere, encryption at rest (AES-256).  
- **Hosting / Infra:** AWS (ECS/EKS + RDS) or DigitalOcean / Heroku for MVP.  
- **Monitoring:** Prometheus + Grafana, Sentry for errors.  
- **CI/CD:** GitHub Actions (lint, test, build, deploy).

---

## Data Model (core tables — simplified)

- **users** (id, name, phone, email, national_id_hash, created_at)  
- **health_ids** (id, user_id, public_identifier, created_at)  
- **records** (id, health_id, type, payload_ref, source_provider_id, verified, created_at)  
- **providers** (id, name, license_id, public_key)  
- **consents** (id, health_id, provider_id, scope, expires_at, granted_by_user_id, created_at)  
- **audit_logs** (id, actor, action, target, timestamp, metadata)

> Clinical payloads (large test results, images) stored in object storage (S3) and referenced from `records.payload_ref`.

---

## Key API endpoints (examples)

- `POST /api/v1/auth/register` — create user account & issue Health ID  
- `POST /api/v1/healthid/verify` — verify national ID (via gov/NIN or KYC partner)  
- `GET  /api/v1/records/:healthId` — (consent required) fetch summary or list of records  
- `POST /api/v1/consent/request` — provider requests access for a given scope  
- `POST /api/v1/consent/grant` — patient grants consent (returns short-lived token)  
- `POST /api/v1/emergency/scan` — QR/NFC scan exchange; returns restricted view (audit logged)

---

## Emergency flow (sequence diagram)

```mermaid
sequenceDiagram
    participant Responder
    participant API
    participant Auth
    participant RecordsDB
    Responder->>API: Scan QR (healthId)
    API->>Auth: validate emergency access token
    Auth-->>API: token OK (scoped)
    API->>RecordsDB: fetch critical fields (allergies, blood, meds)
    RecordsDB-->>API: return critical data
    API-->>Responder: return limited emergency payload
