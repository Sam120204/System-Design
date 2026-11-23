# Load Balancing

Vertical scaling gives an instance more resources. Horizontal scaling adds instances and needs a way to direct traffic among them. Load balancing selects an eligible backend based on endpoint discovery, health, and a routing policy.

![Vertical scaling replaces one instance with a larger one, while horizontal scaling adds instances](https://www.hellointerview.com/_next/static/media/gz2YPLIbmlVE.2t1r4_aqc-l8r.svg)

After adding instances, the system needs a way to pick an eligible server for each request or connection.

![A client has three possible backend servers and needs to select one](https://www.hellointerview.com/_next/static/media/A2ooJ0ht9qtM.18tch993vobcn.svg)

## Choose where balancing happens

| Approach | Usual routing unit | What it enables | Operational concern |
| --- | --- | --- | --- |
| Client-side | Connection or request, depending on the client | The client selects from discovered endpoints and can balance calls across its own connections | Every client needs compatible discovery, health information, and balancing behavior |
| L4 balancer | TCP connection or UDP flow | Forwarding based on the network and transport tuple, often with TLS passed through | Multiplexed or long-lived flows can concentrate work on one backend |
| L7 reverse proxy | Application request or stream, depending on the protocol | Route by HTTP host, path, or headers, and apply request-level policies | Requires protocol support and usually TLS termination for HTTPS inspection |

### Client-side balancing

Clients can cache a list of endpoints from service discovery and refresh it as membership changes. This works for controlled internal clients such as a gRPC service. DNS can also spread clients across addresses, with a refresh speed affected by caching and the lifetime of existing connections. [DNS and IP](02-internet-protocol.md) · [gRPC](09-grpc.md)

![Decision tree weighing client count and endpoint update delays for client-side balancing](https://www.hellointerview.com/_next/static/media/pc4frD_W9Vme.17otsj8h5_z62.svg)

### Dedicated L4 and L7 balancers

[![Dedicated load balancer forwarding client traffic to one of three servers](https://www.hellointerview.com/_next/static/media/km3Yj8m6-r4B.2hzl4s-a2f4gk.svg)](images/dedicated-load-balancer.png)

L4 products can forward packets, translate addresses, or proxy connections depending on implementation. An L7 proxy can terminate client TLS and use a separately encrypted hop to the backend. Balance the unit that carries the work. A few connections can carry many gRPC calls, while a WebSocket established for an hour can consume resources even when its request count stays low. [gRPC](09-grpc.md) · [WebSockets](11-websockets.md)

[![L4 balancer sketch showing connection routing without inspecting the HTTP request](https://www.hellointerview.com/_next/static/media/-blLoTPnJ-PE.3k_db_b7zdzpv.svg)](images/layer-4-load-balancing.png)

[![L7 balancer sketch showing an HTTP-aware proxy between the client and backend](https://www.hellointerview.com/_next/static/media/Lgk4gsoL2L_6.2as4camcs-sd7.svg)](images/layer-7-load-balancing.png)

## Backend selection and lifecycle

- **Round robin and weights.** Rotate across endpoints to distribute similar work. Give higher-capacity or newly deployed backends explicit weights to control their share.
- **Least connections or least requests.** Prefer endpoints with less active work where duration varies. Track a metric that fits the protocol and workload, since one multiplexed connection can carry many requests.
- **Hashing and affinity.** Select by a stable key when cache locality or session routing matters. Consistent hashing can reduce remapping as endpoints change. Hot keys can still produce skew. [Consistent Hashing](../consistent-hashing/README.md)
- **Health and draining.** Use readiness checks for new traffic, and consider passive signals such as repeated connection failures. Remove an instance from new selection before terminating it, and give active work a bounded drain period. Size the remaining fleet to absorb unavailable capacity.

## Design example

An API edge uses one public hostname. An L7 proxy routes `/api` to an application pool and `/images` to an image pool. Each pool can scale independently. A separate gRPC-aware balancing policy spreads internal RPCs, and the WebSocket gateway pool is sized by open connections and fan-out throughput. Observe backend error rates, latency, pool saturation, and distribution across instances. [Cloud load balancing note](<../../web development/云负载均衡详解.md>)
