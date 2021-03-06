# kong-k3s-demo

## Installation

1. Install a local `k3s` cluster by `k3d`.
2. Install `Kong Ingress Controller` on k3s:

    [Kong for Kubernetes with Kong Enterprise (Free) in dbless mode](install/install-ent-free-dbless.md)

    [Kong for Kubernetes with Kong Enterprise (Free) in db-backed mode](install/install-ent-free-postgres.md)

    [Kong for Kubernetes in dbless mode](install/install-oss-dbless.md)

## Demo

Simple steps to [demonstrate](demo/demo.md) the features of `Kong Ingress Controller` and `Kong Gateway`:

1. Proxy service by Kong
2. Protect API by API Key
3. Limit API traffic
4. Restrict IP Addresses of API consumer
5. Setup Backend Services Health Check and Circuit-Breaker