---
layout: default
title: Consul & .svc Endpoints
parent: Microservices
nav_order: 2
---

## Consul & `.svc` Endpoints

**Context:** Hardcoding service URLs (`http://10.0.3.47:8080`) breaks the moment an instance scales, moves, or dies. A service mesh like [Consul](https://developer.hashicorp.com/consul/docs) maintains a registry of healthy instances and provides stable DNS names so services find each other without caring about infrastructure.

### How It Works

```mermaid
graph TD
    subgraph Client
    A[Billing Service]
    end

    subgraph Discovery
    B[Consul DNS]
    end

    subgraph "Vehicle Service Cluster"
    I1[Instance-1 <br/>10.0.3.47 <br/><b>Healthy</b>]
    I2[Instance-2 <br/>10.0.3.48 <br/><b>Healthy</b>]
    I3[Instance-3 <br/>10.0.3.49 <br/><b>Failing ✗</b>]
    end

    A -- "DNS query: vehicle-service.svc" --> B
    B -- "A record: 10.0.3.47" --> A

    B -. checks .- I1
    B -. checks .- I2
    B -. checks .- I3

    style I1 fill:#d4edda,stroke:#28a745
    style I2 fill:#d4edda,stroke:#28a745
    style I3 fill:#f8d7da,stroke:#dc3545
```

- Services register with Consul on startup and send periodic health checks.
- Consul removes unhealthy instances from DNS responses.
- Consumers use internal `.svc` DNS names - no IPs, no load balancer config changes needed.

### Environment Config

```yaml
# endpoints.yaml (outgoing calls)
vehicle_service:
    base_url: "http://vehicle-service.svc:8080"

pricing_service:
    base_url: "http://pricing-service.svc:8080"
```

### Consul Health Check (Registered by the Service)

```json
{
  "service": {
    "name": "vehicle-service",
    "port": 8080,
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s",
      "timeout": "2s",
      "deregister_critical_service_after": "30s"
    }
  }
}
```

> **Gotcha:** If your health check endpoint hits the database or a downstream dependency, a DB outage will mark your service as unhealthy - even though the service process itself is fine. Keep `/health` lightweight (return 200 if the process is up). Use a separate `/health/ready` for deep checks that include dependencies.
