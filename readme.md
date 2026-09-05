# Kubernetes ML Function Scheduler

An experimental Go control plane for placing machine-learning function variants
on Kubernetes. A request specifies a task, minimum accuracy, and deadline; the
scheduler selects a registered variant, accounts for GPU resources, starts a
pod and service when needed, and dispatches work to a running instance.

> **Project status:** research prototype. It assumes a local MySQL database and
> an accessible Kubernetes cluster, and is not hardened for production use.

## What is implemented

- HTTP API for registering variants and submitting function requests
- Accuracy-aware request queues for ready and blocked work
- Kubernetes node discovery, pod/service creation, and pod monitoring
- Variant placement using GPU memory, GPU cores, startup latency, throughput,
  and colocation information
- Round-robin dispatch across running instances
- Load monitoring and instance scale operations
- Request lifecycle logging to MySQL
- Python workload generators, simulators, plotting scripts, and sample
  variants for object detection, sentiment analysis, and summarization

## Architecture

```mermaid
flowchart LR
  Client -->|POST /run| API[Gin API]
  API --> Queue{Request queues}
  Queue -->|eligible instance| Ready[Ready queue]
  Queue -->|no eligible instance| Blocked[Blocked queue]
  Blocked --> LB[Load balancer]
  LB --> RM[Resource manager]
  RM --> K8s[Kubernetes client]
  K8s --> Pods[Variant pods and services]
  Ready --> Dispatcher
  Dispatcher --> Pods
  RM <--> DB[(MySQL)]
```

`ResourceManager` owns the in-memory task, variant, instance, request, and node
stores. Background loops monitor Kubernetes pods, scale capacity, unblock
requests, and dispatch ready work.

## Requirements

- Go 1.20
- A Kubernetes cluster reachable through the current kubeconfig
- MySQL on `localhost:3306`
- GPU-labelled nodes and compatible container images for GPU variants

The scheduler reads its database connection from environment variables. Copy
the example as a reference, then export the values in your shell or load them
with your preferred environment manager (the Go process does not load `.env`
files itself):

```bash
cp .env.example .env
export MYSQL_ADDRESS=127.0.0.1:3306
export MYSQL_DATABASE=vroom
export MYSQL_USER=vroom
export MYSQL_PASSWORD='replace-with-a-local-password'
```

`MYSQL_ADDRESS`, `MYSQL_DATABASE`, and `MYSQL_USER` have the non-sensitive
defaults shown above. `MYSQL_PASSWORD` has no default and must be set, including
to an explicitly empty value if that is how an isolated local database is
configured. `.env` is ignored by Git.

Create the database and a least-privileged application user with your own
administrator account. Grant that user schema access only to the selected
database; do not reuse administrator or production credentials.

## Build and run

```bash
git clone https://github.com/xzaviourr/kubernetes-ml-function-scheduler.git
cd kubernetes-ml-function-scheduler
go mod download
go build ./...
go run .
```

The API listens on `0.0.0.0:8083`.

## API

Register a model/function variant:

```bash
curl -X POST http://localhost:8083/insert \
  -H 'Content-Type: application/json' \
  -d '{
    "task-identifier": "image-recognition",
    "gpu-memory": 4096,
    "gpu-cores": 1,
    "image": "registry.example/image-recognition:latest",
    "startup-latency": 4,
    "min-latency": 25,
    "mean-latency": 40,
    "max-latency": 80,
    "accuracy": 90,
    "batch-size": 1,
    "end-point": "/predict",
    "port": 8080,
    "capacity": 10
  }'
```

Submit work using the fields defined by `FuncReq` in
[`request.go`](./request.go):

```bash
curl -X POST http://localhost:8083/run \
  -H 'Content-Type: application/json' \
  -d '{"task-identifier":"image-recognition","deadline":2000,"accuracy":85,"request-size":1,"args":"","response-url":""}'
```

## Repository map

| Area | Purpose |
| --- | --- |
| `apiServer.go`, `request.go` | API and request model |
| `scheduler.go`, `dispatcher.go` | queues, scheduling, and request forwarding |
| `resourceManager.go`, `variant.go`, `instance.go`, `node.go` | resource state |
| `kubernetesClient.go` | Kubernetes discovery and workloads |
| `loadBalancer.go` | capacity monitoring and scaling |
| `database.go`, `logger.go` | MySQL persistence |
| `simulator.py`, `workload_generator.py`, `profiler/` | experiments and analysis |
| `variants/` | sample ML-serving containers |

## Validation

```bash
go test ./...
```

There are currently no automated test files; this command compiles all Go
packages. Integration validation additionally requires MySQL, Kubernetes, and
the referenced model images.

## Limitations

- Database settings are environment-driven, but other cluster assumptions
  remain compiled into the prototype.
- Shared scheduler maps are accessed by concurrent goroutines without an
  explicit synchronization layer.
- API authentication, authorization, TLS, retries, and admission controls are
  not implemented.
- Included CSV/PNG files are experiment artifacts, not published benchmarks.
- Run only against a disposable cluster until configuration and security
  controls are added.

## License

Released under the [MIT License](./LICENSE).
