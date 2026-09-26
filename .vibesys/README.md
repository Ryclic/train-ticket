# VibeSys tasks

This fork of [FudanSELab/train-ticket](https://github.com/FudanSELab/train-ticket)
carries repository-native VibeSys tasks. The `vibesys` branch is based on
upstream `350f62000e6658e0e543730580c599d8558253e7`, whose services store data
in MongoDB. The packaged Train Ticket oracle targets that API contract; later
upstream revisions use different entity identifiers and need a different
oracle.

Run VibeSys from the repository root and select a task under `tasks/`:

| Task | Command | Target |
| --- | --- | --- |
| `default` | `vibesys --task default` | An already running Train Ticket deployment |
| `kubernetes` | `vibesys --local --run-environment local --profiler none --task kubernetes` | Six services built from this repository and deployed to kind |

Task definitions are authored input. VibeSys owns generated run data under
`state/`; local-only data under `state/local/` is ignored by Git.

The evaluator is the `servicebench` package in a VibeSys checkout
(`resources/evaluators/microservice`). Commands below use `$VIBESYS_REPO` for
that checkout.

## `default` task

Accuracy and workload inputs for a running Train Ticket deployment. The shared
Go service evaluator runs both modes; independent Train Ticket benchmark and
accuracy adapters retain separate application-specific semantic oracles. Both
modes can run without a full `vibesys` optimization run.

Expected targets:

- UI proxy: `http://localhost:8080`
- Gateway directly: `http://localhost:18888`
- Kubernetes NodePort UI: `http://<node-ip>:32677`
- Direct service ports (config 15679, station 12345, train 14567, travel 12346,
  route 11178, price 16579)

The scripts call `/api/v1/...` endpoints through whichever base URL you pass.

```bash
go -C "$VIBESYS_REPO/resources/evaluators/microservice" run ./cmd/servicebench \
  --mode accuracy \
  --workload "$PWD/.vibesys/tasks/default/benchmark/workload.toml" \
  --base-url http://localhost:8080 \
  --seed random
go -C "$VIBESYS_REPO/resources/evaluators/microservice" run ./cmd/servicebench \
  --workload "$PWD/.vibesys/tasks/default/benchmark/workload.toml" \
  --base-url http://localhost:8080 \
  --duration 30 \
  --output-json /tmp/train_ticket_bench.json
```

Both modes require Go; dependencies are pinned by `go.sum` in that package.

The checker requires the exact v0.2.0 seed catalog and does not silently accept
empty or structurally different startup data. The committed benchmark workload
runs three independent repetitions and reports their median as
`operations_per_second`, so normal host variance does not steer optimization
from a single trial.

### Local cluster helper

`scripts/start-local-cluster.sh` is a developer helper that no run command
invokes. It starts a minimal Docker Compose cluster: Nacos, Redis, the gateway,
and the config, station, train, travel, route, and price services with local
MongoDB and MySQL dependencies. `check` and `bench` need `VIBESYS_REPO` set.

```bash
export VIBESYS_REPO=/path/to/vibesys
.vibesys/scripts/start-local-cluster.sh start
.vibesys/scripts/start-local-cluster.sh check
.vibesys/scripts/start-local-cluster.sh bench
.vibesys/scripts/start-local-cluster.sh stop
```

Defaults:

- Gateway: `http://localhost:18888`
- Images: `codewisdom/<service>:0.2.0` (gateway: `codewisdom/ts-gateway-service:latest`,
  the only tag published for it)

Override with `TT_GATEWAY_PORT`, `TT_NAMESPACE`, `TT_TAG`, or `TT_GATEWAY_TAG`.
Services outside the six-service contract are excluded from the checker and
benchmark, and the published dashboard image is not started because its nginx
config references services outside this cluster. All services self-seed
reference data on startup. `stop` removes the containers and their anonymous
data volumes, so each `start` reseeds from scratch.

The prebuilt `codewisdom` 0.2.0 service images predate Nacos support and never
register with the discovery server, so the published gateway image starts but
returns `503` for every route. Use the helper `check` and `bench` commands,
which address the direct service ports.

The MySQL containers and `*_MYSQL_*` env are only used by later upstream
revisions; they are started so that either image set works. Source at this
revision uses MongoDB and has no gateway or Nacos.

#### Build from source

The public prebuilt image set is stale: some images referenced by the upstream
compose file are no longer published with the expected tags, and the gateway is
not published under the `0.2.0` tag. To build the service images from this
repository:

```bash
.vibesys/scripts/start-local-cluster.sh build-source
```

That packages the Java services with a Dockerized Maven/JDK 8 build, then runs
the upstream `hack/build-image.sh` with image namespace `localtrain` and tag
`source`. It also creates local compatibility tags for old upstream
Dockerfiles: `java:8-jre` -> `eclipse-temurin:8-jre` and `python:3` ->
`python:3.8-bullseye`. Deploy the result with:

```bash
TT_NAMESPACE=localtrain TT_TAG=source TT_SKIP_PULL=1 \
  .vibesys/scripts/start-local-cluster.sh start
```

`TT_SKIP_PULL=1` is required for local images; otherwise Docker Compose tries
to pull `localtrain/...` from Docker Hub. `start` waits for the six direct
service `welcome` endpoints.

## `kubernetes` task

Deploys the six-service workload to a local kind cluster and runs the same
accuracy checker and benchmark. The minimal topology is six MongoDB instances
and the config, station, train, travel, route, and price services, built from
this repository into local images. The evaluator reaches the six services
through direct port-forwards. The gateway, Redis, and other Train Ticket
services are outside this workload's contract.

Prepare kind cluster `vibesys-k8s-train` with context `kind-vibesys-k8s-train`,
then run the command in the table above. Each invocation creates and removes
its own namespace. Profiling stays disabled until Kubernetes telemetry capture
is available.

This task supports the local run environment only: its evaluator invokes host
`kubectl`, Docker, and kind and reads the host Kubernetes context. Container and
Modal evaluator environments and Kubernetes profiling are not supported.

The assets have configuration and packaging tests in VibeSys. They have not yet
run a live Kubernetes accuracy or benchmark campaign. Dependency image tags are
not all pinned by digest. See `tasks/kubernetes/README.md`.
