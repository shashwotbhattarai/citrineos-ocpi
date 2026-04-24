# CitrineOS OCPI - Deployment & Integration Guide

**Last Updated**: April 24, 2026

## Architecture Overview

CitrineOS OCPI is a **standalone microservice** that adds OCPI 2.2.1 roaming capability on top of CitrineOS Core. It runs as a separate container and connects to the same infrastructure.

```
Partner eMSPs ──OCPI 2.2.1──> citrineos-ocpi (:8085)
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Shared PostgreSQL   Hasura GraphQL   RabbitMQ
              (same RDS)         (queries core)   (event-driven push)
                    ▲               ▲               ▲
                    └───────────────┼───────────────┘
                                    │
                              citrineos-core (:8080)
                                    │
                              Charging Stations (OCPP)
```

### Integration Channels
1. **Shared PostgreSQL** (AWS RDS) — OCPI adds notification tables, reads core's Location/Transaction/Tariff data
2. **Hasura GraphQL** — OCPI queries core data via Hasura
3. **RabbitMQ** — Core publishes events, OCPI subscribes and broadcasts to partner eMSPs
4. **PostgreSQL LISTEN/NOTIFY** — Real-time notifications for data changes
5. **Core REST API** — OCPI calls core's OCPP endpoints for partner remote start/stop commands

### OCPI Modules Implemented
| Module | Role | Purpose |
|--------|------|---------|
| Versions | Discovery | Partners discover your OCPI endpoints |
| Credentials | Registration | Token handshake for partner onboarding |
| Locations | SENDER | Share charger locations with partners |
| Sessions | SENDER | Real-time charging session data |
| CDRs | SENDER | Charge Detail Records for billing |
| Tariffs | SENDER | Pricing information |
| Tokens | RECEIVER | Accept partner customer tokens |
| Commands | RECEIVER | Partner remote start/stop/unlock |
| ChargingProfiles | RECEIVER | Cross-network charging profiles |

---

## Production Deployment

### EC2 Instance
- **IP**: `35.154.162.35` (public), `172.31.42.188` (private)
- **User**: `ubuntu`
- **Stack directory**: `~/yatri-citrine-ocpi`

### Services Running
| Service | Port | Image |
|---------|------|-------|
| CitrineOS Core | 8080 | `yatridockerhub/yatri-energy-citrine:cicd` |
| OCPP 2.0.1 WS | 8081 | (same) |
| OCPP 1.6 WS | 8092-8094 | (same) |
| Hasura GraphQL | 8090 | `yatridockerhub/yatri-energy-hasura:cicd` |
| **OCPI Server** | **8085** | **`yatridockerhub/yatri-energy-ocpi:cicd`** |
| Midlayer API | 4000 | `yatridockerhub/yatri-energy-midlayer:dev` |
| RabbitMQ (citrine) | 5672/15672 | rabbitmq:3-management |
| RabbitMQ (midlayer) | 5673/15673 | rabbitmq:3-management |
| Redis | 6379 | redis:7-alpine |
| Watchtower | - | containrrr/watchtower |

### Docker Compose Location
- **Citrine + OCPI + Hasura**: `~/yatri-citrine-ocpi/docker-compose.yml`
- **Midlayer**: `~/yatri-midlayer/docker-compose.yml`

### CI/CD Pipeline
- **Core repo**: `.github/workflows/yatri-energy-citrine-cicd.yml` — builds on push to `cicd` branch
- **OCPI repo**: `.github/workflows/yatri-energy-ocpi-cicd.yml` — builds on push to `cicd` branch
- Both push to Docker Hub, Watchtower auto-deploys on EC2
- GitHub secrets needed: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`

### Environment Variables (OCPI Service)
OCPI reuses the same `.env` as core (`BOOTSTRAP_*` variables). Key mappings:

| OCPI Env Var | Source |
|---|---|
| `DB_HOST` | `${BOOTSTRAP_RDS_HOST}` |
| `DB_PORT` | `${BOOTSTRAP_RDS_PORT}` |
| `DB_NAME` | `${BOOTSTRAP_RDS_DATABASE}` |
| `DB_USER` | `${BOOTSTRAP_RDS_USERNAME}` |
| `DB_PASS` | `${BOOTSTRAP_RDS_PASSWORD}` |
| `DB_SSL` | `${BOOTSTRAP_RDS_SSL}` |
| `GRAPHQL_ENDPOINT` | `http://graphql-engine:8080/v1/graphql` |
| `AMQP_URL` | `amqp://...@amqp-broker:5672` |
| `AMQP_EXCHANGE` | `${BOOTSTRAP_AMQP_EXCHANGE}` |
| `COMMANDS_CORE_HEADERS` | `{"X-API-Key":"${BOOTSTRAP_CITRINEOS_API_KEY}"}` |

### SSL Support (Added for AWS RDS)
SSL was added to 4 files to support RDS `sslmode=require`:
- `00_Base/src/config/ocpi.types.ts` — `ssl` field in Zod database schema
- `Server/src/config/envs/docker.ts` — reads `DB_SSL` env var
- `00_Base/src/events/pgNotify/subscriber.ts` — pg.Client SSL option
- `Server/src/config/sequelize.bridge.config.ts` — Sequelize `dialectOptions.ssl`

---

## OCPI Registration Flow

### Concepts
- **CPO** (Charge Point Operator) = You (Yatri Energy) — own the chargers
- **eMSP** (e-Mobility Service Provider) = Partner — has an app with customers who want to charge

### Your CPO Identity (Tenant 1)
```
partyId: "YEN"
countryCode: "NP"
role: CPO
name: "Yatri Energy"
```

`serverProfileOCPI` is set on Tenant 1 with version 2.2.1 endpoints for:
credentials, locations (SENDER), tariffs (SENDER), sessions (SENDER), cdrs (SENDER), tokens (RECEIVER), commands (RECEIVER)

### Token Handshake (3-Token Exchange)
```
YOU (CPO)                                    PARTNER (eMSP)
   │                                              │
   │  1. Admin generates Token A                  │
   │     POST /2.2.1/credentials/generate-credentials-token-a
   │                                              │
   │──── Share Token A via email/secure chat ────>│
   │                                              │
   │<─── Partner calls POST /2.2.1/credentials ───│
   │     with Token A + their endpoints           │
   │                                              │
   │  2. Server generates Token B                 │
   │     fetches partner's endpoints              │
   │                                              │
   │──── Sends Token B to partner's API ────────>│
   │                                              │
   │<─── Partner responds with Token C ───────────│
   │                                              │
   │  DONE: Partner uses Token B, you use Token C │
```

### Authentication
- **Admin endpoints** (`generate-credentials-token-a`, `register-credentials-token-a`, etc.): Protected by OIDC if configured. **Currently no OIDC = open access.**
- **Registration endpoints** (`versions`, `credentials`): Require `Authorization: Token <token>` header matching a TenantPartner's `serverCredentials.token`
- **Functional endpoints** (`locations`, `sessions`, etc.): Require Token + OCPI routing headers (`X-OCPI-From-Country-Code`, `X-OCPI-From-Party-ID`, `X-OCPI-To-Country-Code`, `X-OCPI-To-Party-ID`)

### Step-by-Step: Register a New Partner

**Step 1: Create TenantPartner record in Hasura**

```graphql
mutation CreatePartner($profile: jsonb!) {
  insert_TenantPartners_one(object: {
    tenantId: 1,
    partyId: "TST",
    countryCode: "US",
    partnerProfileOCPI: $profile,
    createdAt: "2026-04-24T00:00:00.000Z",
    updatedAt: "2026-04-24T00:00:00.000Z"
  }) {
    id
    partyId
    countryCode
  }
}
```
Variables:
```json
{
  "profile": {
    "roles": [{
      "country_code": "US",
      "party_id": "TST",
      "role": "EMSP",
      "business_details": {
        "name": "Test eMSP",
        "website": "https://test-emsp.com"
      }
    }]
  }
}
```

**Step 2: Generate Token A (admin endpoint, no auth)**

```bash
curl -X POST 'http://35.154.162.35:8085/ocpi/2.2.1/credentials/generate-credentials-token-a' \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "http://35.154.162.35:8085/ocpi/versions/1",
    "role": {
      "country_code": "NP",
      "party_id": "YEN",
      "role": "CPO",
      "business_details": {
        "name": "Yatri Energy",
        "website": "https://yatrimotorcycle.com"
      }
    },
    "mspCountryCode": "US",
    "mspPartyId": "TST"
  }'
```

**Step 3: Share Token A with partner** (email, secure channel)

**Step 4: Partner calls your credentials endpoint with Token A**

```bash
# Partner sends this to your server
curl -X POST 'http://35.154.162.35:8085/ocpi/2.2.1/credentials' \
  -H 'Authorization: Token <TOKEN_A>' \
  -H 'X-Request-ID: unique-id' \
  -H 'X-Correlation-ID: unique-id' \
  -H 'Content-Type: application/json' \
  -d '{
    "token": "<PARTNERS_TOKEN_A>",
    "url": "https://partner-emsp.com/ocpi/versions",
    "roles": [{
      "country_code": "US",
      "party_id": "TST",
      "role": "EMSP",
      "business_details": {
        "name": "Test eMSP"
      }
    }]
  }'
```

Server validates Token A, fetches partner endpoints, generates Token B, exchanges tokens.

**Step 5: Registration complete** — both sides can now authenticate

### Current Status (April 24, 2026)
- [x] OCPI server deployed and running on EC2
- [x] All event subscriptions active (Location, ChargingStation, EVSE, Connector, Transaction, MeterValue, Tariff notifications)
- [x] Tenant 1 (Yatri Energy) configured with OCPI identity (YEN/NP)
- [x] Test TenantPartner created (TST/US, id: 2)
- [ ] **NEXT**: Generate Token A for test partner and complete registration flow
- [ ] Configure OIDC to secure admin endpoints in production
- [ ] Set up real partner eMSP registration

---

## Key Endpoints

| Endpoint | Purpose |
|----------|---------|
| `http://35.154.162.35:8085/docs` | Swagger UI |
| `http://35.154.162.35:8085/ocpi/versions/:tenant_id` | Version discovery (needs Token) |
| `http://35.154.162.35:8085/ocpi/2.2.1/credentials` | Credential exchange (needs Token) |
| `http://35.154.162.35:8085/ocpi/2.2.1/credentials/generate-credentials-token-a` | Admin: generate Token A |
| `http://35.154.162.35:8085/ocpi/2.2.1/credentials/register-credentials-token-a` | Admin: register partner |
| `http://35.154.162.35:8085/ocpi/2.2.1/credentials/regenerate-credentials-token` | Admin: refresh Token B |
| `http://35.154.162.35:8085/ocpi/2.2.1/credentials/unregister-client` | Admin: remove partner |

---

## Key Code Locations

| Component | Path |
|-----------|------|
| Server entry point | `Server/src/index.ts` |
| Docker config (env vars) | `Server/src/config/envs/docker.ts` |
| Local config | `Server/src/config/envs/local.ts` |
| OCPI config schema (Zod) | `00_Base/src/config/ocpi.types.ts` |
| Auth middleware | `00_Base/src/util/middleware/AuthMiddleware.ts` |
| Admin endpoint decorator | `00_Base/src/util/decorators/AsAdminEndpoint.ts` |
| Registration decorator | `00_Base/src/util/decorators/AsOcpiRegistrationEndpoint.ts` |
| Credentials controller | `03_Modules/Credentials/src/module/CredentialsModuleApi.ts` |
| Credentials service | `00_Base/src/services/CredentialsService.ts` |
| Versions controller | `03_Modules/Versions/src/module/VersionsModuleApi.ts` |
| PG LISTEN/NOTIFY | `00_Base/src/events/pgNotify/subscriber.ts` |
| GraphQL queries | `00_Base/src/graphql/queries/` |
| Sequelize migrations | `migrations/` |
| Default seeders | `seeders/` |

---

## Hasura Console

**URL**: `http://35.154.162.35:8090/console`

### Useful Queries

```graphql
# List tenants
query { Tenants { id name partyId countryCode serverProfileOCPI } }

# List partners
query { TenantPartners { id tenantId partyId countryCode partnerProfileOCPI } }

# List locations (for OCPI sharing)
query { Locations(where: { tenantId: { _eq: 1 } }) { id name address } }
```

---

## Troubleshooting

### OCPI returns "Not Authorized"
- Registration/versions endpoints require `Authorization: Token <token>` header
- Token must match a TenantPartner's `partnerProfileOCPI.serverCredentials.token`
- Admin endpoints are open (no OIDC configured)

### OCPI can't connect to RDS
- Check `DB_SSL` is set to `true` in docker-compose
- SSL support was added to `ocpi.types.ts`, `docker.ts`, `subscriber.ts`, `sequelize.bridge.config.ts`

### OCPI can't reach Core API (for commands)
- Core must be healthy before OCPI starts (`depends_on: citrine: condition: service_healthy`)
- `COMMANDS_CORE_HEADERS` must contain valid API key matching core's `CITRINEOS_API_KEY`
- Command URLs use Docker container name `citrine` (e.g., `http://citrine:8080/ocpp/...`)

### Container startup order
1. amqp-broker (RabbitMQ) — must be healthy first
2. citrine (Core) — runs DB migrations, must be healthy
3. graphql-engine (Hasura) — applies metadata, must be healthy
4. citrineos-ocpi (OCPI) — connects to all three above
