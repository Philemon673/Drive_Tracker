# Codebase Walkthrough Script
## "Microservices with Go" — Ride Sharing System

---

## SECTION 1 — What Are We Building?

We are building the backend for a **Uber-style ride-sharing app**.

The system is made up of multiple independent services that talk to each other
through a message broker (RabbitMQ) and direct calls (gRPC and HTTP).

The full trip flow looks like this:

```
User requests a ride
  → API Gateway receives it
    → Trip Service creates the trip
      → Driver Service finds a driver
        → Driver accepts via WebSocket
          → Payment Service charges the user via Stripe
            → Trip is marked complete
```

Everything is deployed on **Kubernetes**, developed locally with **Tilt**,
and written in **Go**.

---

## SECTION 2 — Monorepo Structure

The entire project lives in one repository with one Go module:

```
go.mod  →  module ride-sharing
```

This means every service and shared package is imported using paths like:
  `ride-sharing/shared/env`
  `ride-sharing/services/trip-service/internal/domain`

Top-level folders:

```
microservices-go-starter/
├── services/       ← the microservices (Go)
├── shared/         ← code shared across all services
├── infra/          ← Docker + Kubernetes configs
├── web/            ← Next.js frontend
├── docs/           ← architecture diagrams
├── tools/          ← developer CLI tools
├── Tiltfile        ← local dev orchestration
├── Makefile        ← proto code generation
└── go.mod          ← single Go module for the whole repo
```

---

## SECTION 3 — The `shared/` Package

These packages are imported by every service. They define the common language
of the entire system.

---

### `shared/env/env.go`

Reads environment variables with a fallback default.

```go
env.GetString("HTTP_ADDR", ":8081")
env.GetInt("PORT", 8080)
env.GetBool("DEBUG", false)
```

Used in: `api-gateway/main.go` to read the server address from the
Kubernetes ConfigMap injected as an environment variable.

---

### `shared/contracts/amqp.go`

Defines the **routing key constants** for RabbitMQ — the names of every
event and command that flows between services.

```go
TripEventCreated             = "trip.event.created"
TripEventDriverAssigned      = "trip.event.driver_assigned"
TripEventNoDriversFound      = "trip.event.no_drivers_found"
DriverCmdTripRequest         = "driver.cmd.trip_request"
DriverCmdTripAccept          = "driver.cmd.trip_accept"
PaymentEventSessionCreated   = "payment.event.session_created"
PaymentEventSuccess          = "payment.event.success"
```

These constants are the **contract between services**. If a service publishes
`trip.event.created`, another service subscribes to exactly that string.
Centralising them here prevents typos and keeps all services in sync.

Also defines `AmqpMessage`:
```go
type AmqpMessage struct {
    OwnerID string `json:"ownerId"`
    Data    []byte `json:"data"`
}
```

---

### `shared/contracts/http.go`

Defines the standard HTTP response envelope used by all services:

```go
type APIResponse struct {
    Data  any       `json:"data,omitempty"`
    Error *APIError `json:"error,omitempty"`
}

type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}
```

Every HTTP handler wraps its response in `APIResponse` so the frontend
always gets a consistent shape.

---

### `shared/contracts/ws.go`

Defines WebSocket message shapes:

```go
type WSMessage struct {
    Type string `json:"type"`
    Data any    `json:"data"`
}

type WSDriverMessage struct {
    Type string          `json:"type"`
    Data json.RawMessage `json:"data"`  // raw bytes, decoded later per type
}
```

`WSDriverMessage` uses `json.RawMessage` so the API Gateway can forward
the raw payload to RabbitMQ without needing to fully decode it first.

---

### `shared/retry/retry.go`

Generic exponential backoff retry utility:

```go
retry.WithBackoff(ctx, retry.DefaultConfig(), func() error {
    return connectToRabbitMQ()
})
```

`DefaultConfig()` gives: 3 retries, starting at 1s, capped at 10s.
Each failed attempt doubles the wait. Context cancellation is respected.

Used when connecting to external dependencies (RabbitMQ, MongoDB) that
may not be ready immediately on startup.

---

### `shared/types/types.go`

Shared geographic types used across services and the frontend:

```go
type Route struct {
    Distance float64     `json:"distance"`
    Duration float64     `json:"duration"`
    Geometry []*Geometry `json:"geometry"`
}

type Geometry struct {
    Coordinates []*Coordinate `json:"coordinates"`
}

type Coordinate struct {
    Latitude  float64 `json:"latitude"`
    Longitude float64 `json:"longitude"`
}
```

These come from the OSRM routing API and are passed through Trip Service
back to the frontend to draw the route on the map.

---

### `shared/util/util.go`

Small helper for generating driver avatar URLs:

```go
func GetRandomAvatar(index int) string {
    return fmt.Sprintf("https://randomuser.me/api/portraits/lego/%d.jpg", index)
}
```

Used when registering mock drivers in development.

---

## SECTION 4 — Trip Service (Clean Architecture Deep Dive)

This is the most architecturally complete service. It strictly follows
**Clean Architecture** — dependencies only point inward toward the domain.

```
cmd/main.go                        ← composition root (wires everything)
internal/
  domain/                          ← Layer 1: interfaces + models
    trip.go
    ride_fare.go
  service/                         ← Layer 2: business logic
    service.go
  infrastructure/                  ← Layer 3: external dependencies
    repository/
      inmem.go                     ← in-memory DB (dev placeholder)
    events/                        ← RabbitMQ (not yet implemented)
    grpc/                          ← gRPC server (not yet implemented)
pkg/types/                         ← public types (not yet implemented)
```

---

### Layer 1 — `internal/domain/`

This layer has **zero external dependencies** (except MongoDB's ObjectID type).
It defines what the system IS, not how it works.

**`domain/ride_fare.go`** — the input to create a trip:

```go
type RideFareModel struct {
    ID               primitive.ObjectID
    UserID           string
    PackageSlug      string   // "van", "luxury", "sedan", "suv"
    TotalPriceIncent float64
}
```

**`domain/trip.go`** — the trip entity + the two core interfaces:

```go
type TripModel struct {
    ID       primitive.ObjectID
    UserID   string
    Status   string           // "pending", "active", "completed"
    RideFare *RideFareModel
}

// What the database layer must implement
type TripRepository interface {
    CreateTrip(ctx context.Context, trip *TripModel) (*TripModel, error)
}

// What the service layer must implement
type TripService interface {
    CreateTrip(ctx context.Context, fare *RideFareModel) (*TripModel, error)
}
```

These interfaces are the **contracts**. The service layer depends on
`TripRepository` but never imports the repository package directly.
This is the Dependency Inversion Principle.

---

### Layer 2 — `internal/service/service.go`

Implements `TripService`. Contains the business logic.

```go
type service struct {
    repo domain.TripRepository   // depends on the interface, not the implementation
}

func NewService(repo domain.TripRepository) domain.TripService {
    return &service{repo: repo}
}

func (s *service) CreateTrip(ctx context.Context, fare *domain.RideFareModel) (*domain.TripModel, error) {
    t := &domain.TripModel{
        ID:       primitive.NewObjectID(),  // generate unique ID
        UserID:   fare.UserID,
        Status:   "pending",                // initial status
        RideFare: fare,
    }
    return s.repo.CreateTrip(ctx, t)        // delegate persistence to repo
}
```

Notice: `service.go` imports `domain` but never imports `repository`.
It does not know or care whether the repo is in-memory, MongoDB, or Postgres.

---

### Layer 3 — `internal/infrastructure/repository/inmem.go`

Implements `TripRepository` using a plain Go map. This is the dev/test
stand-in before a real database is connected.

```go
type inmemRepository struct {
    trips     map[string]*domain.TripModel
    rideFares map[string]*domain.RideFareModel
}

func NewInmemRepository() domain.TripRepository {
    return &inmemRepository{
        trips:     make(map[string]*domain.TripModel),
        rideFares: make(map[string]*domain.RideFareModel),
    }
}

func (r *inmemRepository) CreateTrip(ctx context.Context, trip *domain.TripModel) (*domain.TripModel, error) {
    r.trips[trip.ID.Hex()] = trip   // store by MongoDB ObjectID hex string
    return trip, nil
}
```

When MongoDB is added later, a new file `mongo.go` will implement the same
`TripRepository` interface. The service layer requires zero changes.

---

### Composition Root — `cmd/main.go`

This is where all layers are wired together. It is the only place in the
service that knows about all three layers simultaneously.

```go
func main() {
    ctx := context.Background()

    // Layer 3: create the infrastructure implementation
    inmemRepo := repository.NewInmemRepository()

    // Layer 2: inject the repo into the service
    svc := service.NewService(inmemRepo)

    // Use the service
    fare := &domain.RideFareModel{UserID: "42"}
    t, err := svc.CreateTrip(ctx, fare)
    if err != nil {
        log.Println(err)
    }
    log.Println(t)

    // Placeholder: keeps the process alive (will become a gRPC server)
    for {
        time.Sleep(time.Second)
    }
}
```

Dependency flow — arrows show "imports":

```
main.go  →  repository  →  domain
main.go  →  service     →  domain
                            ↑
                     (shared interface)
```

`service` and `repository` never import each other. They only share `domain`.

---

## SECTION 5 — API Gateway (`services/api-gateway/main.go`)

Currently a minimal HTTP server — the skeleton for what will become the
central hub of the system.

```go
var httpAddr = env.GetString("HTTP_ADDR", ":8081")

func main() {
    log.Println("Starting API Gateway")

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("Hello from API Gateway"))
    })

    http.ListenAndServe(httpAddr, nil)
}
```

The address comes from the `GATEWAY_HTTP_ADDR` env var, injected by the
Kubernetes ConfigMap (`app-config.yaml`).

Planned responsibilities (from architecture docs):
- `POST /trip/preview` → gRPC call to Trip Service → OSRM route calculation
- `POST /trip/start`   → gRPC call to Trip Service → creates trip in DB
- `WS /riders`         → WebSocket for rider clients
- `WS /drivers`        → WebSocket for driver clients
- Stripe webhook receiver → publishes `payment.event.success` to RabbitMQ

---

## SECTION 6 — Web Frontend (`web/src/`)

A Next.js app. The key files for understanding the backend contract are:

---

### `web/src/types.ts`

Mirrors the Go domain models on the frontend:

```typescript
interface Trip {
    id: string;
    userID: string;
    status: string;
    selectedFare: RouteFare;
    route: Route;
    driver?: Driver;
}

enum CarPackageSlug {
    SEDAN = "sedan",
    SUV = "suv",
    VAN = "van",
    LUXURY = "luxury",
}

interface Driver {
    id: string;
    location: Coordinate;
    geohash: string;
    name: string;
    profilePicture: string;
    carPlate: string;
}
```

---

### `web/src/contracts.ts`

Defines the exact API endpoints and WebSocket event types the frontend
expects the backend to expose:

```typescript
enum BackendEndpoints {
    PREVIEW_TRIP = "/trip/preview",
    START_TRIP   = "/trip/start",
    WS_DRIVERS   = "/drivers",
    WS_RIDERS    = "/riders",
}

enum TripEvents {
    Created              = "trip.event.created",
    DriverAssigned       = "trip.event.driver_assigned",
    NoDriversFound       = "trip.event.no_drivers_found",
    DriverTripRequest    = "driver.cmd.trip_request",
    DriverTripAccept     = "driver.cmd.trip_accept",
    PaymentSessionCreated = "payment.event.session_created",
    // ...
}
```

These string values are **identical** to the constants in
`shared/contracts/amqp.go`. The frontend and backend share the same
event vocabulary — just in different languages.

---

## SECTION 7 — Infrastructure & Deployment

### Dockerfiles (`infra/development/docker/`)

Both services use the same minimal pattern:

```dockerfile
FROM alpine
WORKDIR /app
ADD shared shared      # copy shared Go packages
ADD build build        # copy pre-compiled binary
ENTRYPOINT build/api-gateway
```

The binary is compiled **outside** Docker (by Tilt on the host), then
copied in. This keeps image builds fast — no Go toolchain inside the image.

---

### Kubernetes Manifests (`infra/development/k8s/`)

**`app-config.yaml`** — ConfigMap injected into pods as env vars:
```yaml
data:
  ENVIRONMENT: "development"
  GATEWAY_HTTP_ADDR: ":8081"
```

**`api-gateway-deployment.yaml`** — `LoadBalancer` service (externally
accessible). Port 8081. Reads `GATEWAY_HTTP_ADDR` from the ConfigMap.

**`trip-service-deployment.yaml`** — `ClusterIP` service (internal only).
Port 8083. Not directly reachable from outside the cluster.

This means: only the API Gateway is exposed to the internet. All other
services are internal and only reachable via the gateway.

---

### `Tiltfile` — Local Dev Loop

Tilt automates the full local development cycle:

```
1. Compile Go binary locally (fast, uses host Go toolchain)
2. Build Docker image (copies binary into alpine image)
3. Apply Kubernetes manifest
4. Port-forward to localhost
5. Watch for file changes → recompile → restart container
```

For each service:
```python
# Step 1: compile
local_resource('api-gateway-compile',
  'CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o build/api-gateway ./services/api-gateway',
  deps=['./services/api-gateway', './shared'])

# Step 2+3: build image and deploy
docker_build_with_restart('ride-sharing/api-gateway', '.',
  entrypoint=['/app/build/api-gateway'],
  dockerfile='./infra/development/docker/api-gateway.Dockerfile',
  live_update=[
    sync('./build', '/app/build'),    # sync binary on change
    sync('./shared', '/app/shared'),  # sync shared code on change
  ])

# Step 4: port-forward
k8s_resource('api-gateway', port_forwards=8081)
```

`deps` tells Tilt which directories to watch. Any change in `./services/api-gateway`
or `./shared` triggers a recompile and hot-reload.

---

### `Makefile`

Single target for regenerating gRPC code from `.proto` files:

```makefile
generate-proto:
    protoc \
        --proto_path=$(PROTO_DIR) \
        --go_out=$(GO_OUT) \
        --go-grpc_out=$(GO_OUT) \
        $(PROTO_SRC)
```

Run `make generate-proto` after editing any `.proto` file to regenerate
the Go gRPC client/server stubs.

---

## SECTION 8 — `tools/create_service.go`

A CLI scaffolding tool. Run it to create a new service with the correct
Clean Architecture structure:

```bash
go run tools/create_service.go -name payment
```

Creates:
```
services/payment-service/
├── cmd/
├── internal/
│   ├── domain/
│   ├── service/
│   └── infrastructure/
│       ├── events/
│       ├── grpc/
│       └── repository/
├── pkg/types/
└── README.md
```

This enforces that every new service follows the same architectural pattern
from day one.

---

## SECTION 9 — Full Message Flow (End to End)

From `docs/architecture/trip-creation-flow-v1.md`:

```
1.  User         →  API Gateway      HTTP: POST /trip/preview
2.  API Gateway  →  Trip Service     gRPC: PreviewTrip
3.  Trip Service →  OSRM API         HTTP: calculate route
4.  OSRM         →  Trip Service     HTTP: route coordinates
5.  Trip Service →  API Gateway      gRPC response: route + fares
6.  API Gateway  →  User             display map preview

7.  User         →  API Gateway      HTTP: POST /trip/start
8.  API Gateway  →  Trip Service     gRPC: CreateTrip
9.  Trip Service →  RabbitMQ         PUBLISH: trip.event.created
10. Driver Svc   →  RabbitMQ         CONSUME: trip.event.created
11. Driver Svc   →  Driver           WS: driver.cmd.trip_request
12. Driver       →  API Gateway      WS: driver.cmd.trip_accept
13. API Gateway  →  RabbitMQ         PUBLISH: driver.cmd.trip_accept
14. Trip Service →  RabbitMQ         CONSUME: driver.cmd.trip_accept
15. Trip Service →  RabbitMQ         PUBLISH: trip.event.driver_assigned
16. Payment Svc  →  RabbitMQ         CONSUME: trip.event.driver_assigned
17. Payment Svc  →  Stripe           create checkout session
18. Stripe       →  Payment Svc      session created
19. Payment Svc  →  RabbitMQ         PUBLISH: payment.event.session_created
20. API Gateway  →  RabbitMQ         CONSUME: payment.event.session_created
21. API Gateway  →  User             WS: show payment UI
22. User         →  Stripe           complete payment
23. Stripe       →  API Gateway      webhook: payment success
24. API Gateway  →  RabbitMQ         PUBLISH: payment.event.success
25. Trip Service →  RabbitMQ         CONSUME: payment.event.success
26. Trip Service                     mark trip as complete
```

---

## SECTION 10 — What Is Not Yet Implemented

| Location | What's Missing |
|---|---|
| `internal/infrastructure/events/` | RabbitMQ publisher and consumer |
| `internal/infrastructure/grpc/` | gRPC server handler for CreateTrip |
| `services/trip-service/pkg/types/` | Public types for other services to import |
| `api-gateway/main.go` | gRPC client, WebSocket handlers, Stripe webhook, routing |
| `cmd/main.go` infinite loop | Will become a real gRPC + HTTP server |
| MongoDB repository | `inmem.go` is the placeholder |
| Driver Service | Not yet scaffolded |
| Payment Service | Not yet scaffolded |

The architecture is fully designed. The contracts, interfaces, and
infrastructure scaffolding are all in place. The course builds out
each missing piece step by step.

---

## Summary — How Every File Connects

```
go.mod
  └── single module "ride-sharing" — all imports start here

shared/
  contracts/amqp.go   ← routing key constants (used by ALL services + frontend)
  contracts/http.go   ← HTTP response shape (used by API Gateway)
  contracts/ws.go     ← WebSocket message shape (used by API Gateway)
  env/env.go          ← env var reader (used by API Gateway)
  retry/retry.go      ← retry logic (used when connecting to RabbitMQ/MongoDB)
  types/types.go      ← geo types Route/Coordinate (used by Trip Service + frontend)
  util/util.go        ← avatar helper (used by Driver Service)

services/trip-service/
  internal/domain/    ← defines TripModel, RideFareModel, TripRepository, TripService
  internal/service/   ← implements TripService, depends on TripRepository interface
  infrastructure/repository/inmem.go  ← implements TripRepository with a map
  cmd/main.go         ← wires repo → service, runs the process

services/api-gateway/
  main.go             ← HTTP server skeleton, reads HTTP_ADDR from env

web/src/
  types.ts            ← frontend mirror of Go domain models
  contracts.ts        ← frontend mirror of shared/contracts (endpoints + event names)

infra/development/
  docker/             ← alpine Dockerfiles, copy binary + shared into image
  k8s/app-config.yaml ← ConfigMap: env vars injected into pods
  k8s/*-deployment.yaml ← Deployment + Service for each microservice

Tiltfile              ← compile → build image → deploy → port-forward → watch
Makefile              ← protoc code generation for gRPC
tools/create_service.go ← scaffolds new services with correct folder structure
```
