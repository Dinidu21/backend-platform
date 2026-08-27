# EventSphere Backend Platform

Platform infrastructure layer for the EventSphere microservices architecture. This directory contains the foundational services that enable service discovery, centralized configuration, and API routing across all backend microservices.

## Student Information

- **Student Name:** Dinidu Sachintha
- **Student Number:** 241711028
- **Slack Handle:** [U0BF767MA4S](https://ijse-eca-hdse-71-72.slack.com/team/U0BF767MA4S)
- **GCP Project ID:** eventsphere-504909

## Components

### Eureka Server
**Port:** 8761

Netflix Eureka service registry that enables dynamic service discovery. All microservices register with Eureka on startup and resolve downstream dependencies through the registry rather than hardcoded URLs.

Key features:
- Service registration and discovery
- Health-aware load balancing
- Self-preservation mode (disabled in development)
- Dashboard available at `http://localhost:8761`

### Config Server
**Port:** 8888

Spring Cloud Config Server that provides centralized externalized configuration. Reads YAML configuration files from the `config-repo` Git repository and serves them to client services at runtime.

Key features:
- Git-backed configuration repository
- Runtime configuration refresh without redeployment
- Environment-specific profiles support
- Encrypted property value support

### API Gateway
**Port:** 8080

Spring Cloud Gateway edge server that serves as the single entry point for all client requests. Routes traffic to downstream microservices using load-balanced URLs, applies global retry and CORS policies, and exposes actuator endpoints for monitoring.

Key features:
- Dynamic routing based on path predicates
- Load-balanced service resolution via Eureka
- Global retry filter with exponential backoff
- CORS configuration for frontend origins
- Actuator endpoints for routes, metrics, and health

### Config Repo

Git repository containing all YAML configuration files for the microservices platform. The Config Server fetches configuration directly from this repository.

Configuration files:
- `eureka-server.yml` — Eureka registry settings
- `config-server.yml` — Config server Git settings
- `api-gateway.yml` — Routing, CORS, resilience
- `user-service.yml` — PostgreSQL datasource, JPA
- `event-booking-service.yml` — PostgreSQL datasource, JPA
- `review-notification-service.yml` — GCP Firestore and Cloud Storage settings

## Platform Architecture

```
                         ┌──────────────────┐
                         │   API Gateway     │
                         │   (Port 8080)     │
                         └────────┬─────────┘
                                  │ lb://
                         ┌────────▼─────────┐
                         │  Eureka Server    │
                         │  (Port 8761)      │
                         └────────┬─────────┘
                                  │
                         ┌────────▼─────────┐
                         │  Config Server    │
                         │  (Port 8888)      │
                         └────────┬─────────┘
                                  │
                         ┌────────▼─────────┐
                         │   Config Repo     │
                         │   (GitHub)        │
                         └──────────────────┘
```

## Service Registry

The Eureka Server registers the following services:

| Service | Port | Description |
|---------|------|-------------|
| API Gateway | 8080 | Edge server and router |
| User Service | 8081 | Identity and access management |
| Event Booking Service | 8082 | Events, venues, tickets, bookings |
| Review & Notification Service | 8083 | Reviews, media, notifications |
| Config Server | 8888 | Centralized configuration |

## Routing Configuration

The API Gateway routes requests based on path patterns:

| Path Pattern | Target Service | Filter |
|-------------|----------------|--------|
| `/api/users/**` | User Service | StripPrefix=1 |
| `/api/events/**` | Event Booking Service | StripPrefix=1 |
| `/api/ticket-types/**` | Event Booking Service | StripPrefix=1 |
| `/api/bookings/**` | Event Booking Service | StripPrefix=1 |
| `/api/reviews/**` | Review & Notification Service | StripPrefix=1 |
| `POST /api/reviews` | Review & Notification Service | StripPrefix=1 |

## Resilience

Global retry filter applied to all gateway routes:

- **Retries:** 3
- **Retry on:** `BAD_GATEWAY`, `SERVICE_UNAVAILABLE`, `GATEWAY_TIMEOUT`
- **Methods:** `GET`, `POST`
- **Backoff:** Exponential (100ms initial, 500ms max, factor 2)

## CORS Policy

Allowed origins:
- `http://localhost:3000` (local development)
- `http://136.68.42.194` (frontend IP)
- `https://eventsphere-frontend-149096254626.asia-south1.run.app` (Cloud Run production)

Allowed methods: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS`

## Running Locally

### Prerequisites

- Java 25
- Maven 3.9+
- Git installed

### Start Order

Services must be started in the following order:

1. **Eureka Server** — Must be available before other services register
2. **Config Server** — Must be available before other services fetch configuration
3. **API Gateway** — Must be available to route client traffic
4. **Business Services** — Register with Eureka and fetch config on startup

### Steps

```bash
# Start Eureka Server
cd eureka-server
./mvnw spring-boot:run

# Start Config Server
cd config-server
./mvnw spring-boot:run

# Start API Gateway
cd api-gateway
./mvnw spring-boot:run
```

## Environment Variables

| Variable | Service | Description |
|----------|---------|-------------|
| `EUREKA_HOSTNAME` | Eureka Server | Hostname for Eureka instance |
| `EUREKA_PEER_URL` | Eureka Server | Comma-separated peer URLs |
| `DB_PASSWORD` | SQL Services | PostgreSQL database password |

## Actuator Endpoints

All platform services expose actuator endpoints for monitoring:

- `GET /actuator/health` — Health check
- `GET /actuator/info` — Application info
- `POST /actuator/refresh` — Config Server only, refreshes configuration

Gateway-specific:
- `GET /actuator/gateway` — Gateway routes and filters
- `GET /actuator/metrics` — Application metrics
