# High-Availability Kubernetes Platform

## System Architecture

Two identical Kubernetes clusters, designated "hot" and "standby". A Load Balancer (CloudFlare) acts as the primary entry point, directing traffic based on health checks.

### Components

1. Kubernetes Clusters (Hot & Standby): Two fully provisioned clusters. The hot cluster serves live traffic, while the standby cluster is fully synced and ready to take over.
2. Load Balancer (CloudFlare): A DNS-based load balancing service that directs traffic to the healthy cluster. This will handle the failover mechanism.
3. NGINX Ingress Controller: Deployed in each cluster to manage external access to the services in the cluster. It handles TLS termination and path-based routing as required.
4. CI/CD Pipeline: An automated pipeline (GitHub Actions) for building, and deploying applications consistently across both clusters. For the purpose of this exercise, the build is not pushed to the cluster, and I will use a publicly available image like nginx and traefik whoami.
5. Monitoring & Logging Stack: A centralized observability stack (Prometheus, Grafana, Loki) deployed to both clusters for real-time information.

### Architecture Diagram

      +---------------------------+
      |       End Users           |
      +-------------+-------------+
                    |
                    | (host fqdn)
                    v
      +---------------------------+
      | CloudFlare Load Balancer  |
      | (with Health Checks)      |
      +-------------+-------------+
                    |
          +-----------------+-----------------+
          | (Active Traffic)|                 | (Standby)
          v                 v                 v
+--------------------------+      +------------------------+
|      Hot K8s Cluster     |      |   Standby K8s Cluster  |
| +------------------+     |      | +------------------+   |
| | NGINX Ingress    |     |      | | NGINX Ingress    |   |
| +------------------+     |      | +------------------+   |
|          |               |      |          |             |
|          v               |      |          v             |
| +------------------+     |      | +------------------+   |
| | Backend Service  |     |      | | Backend Service  |   |
| +------------------+     |      | +------------------+   |
+--------------------------+      +------------------------+
          ^                                 ^
          | (Deployment via CI/CD)          |
          +---------------------------------+

### Traffic Flow

1. A user makes a DNS request to the hostname we define in the `ingress.yml`.
2. The LB resolves the DNS query, returning the IP address of the NGINX Ingress Controller in the hot cluster, which is currently passing health checks.
3. The request hits the NGINX Ingress in the hot cluster.
4. NGINX routes the traffic to the appropriate backend ClusterIP Service.
5. The Service load balances the request to one of the available application pods.

### Assumptions and Design Choices
1. DNS-based Failover: I chose this method for its simplicity and reliability. It avoids complex routing or other proprietary networking solutions.
2. Identical Clusters: Both clusters are kept identical in terms of configuration and deployed applications, to simplify failover and management.
3. Automation-First: All configuration and deployment processes are automated using Ansible and Github Actions.

# Operational Documentation
## Failover Mechanism
The failover process is automated and managed entirely by a CloudFlare Load Balancer, which relies on active health checking.

### Configuration Steps:
1. DNS Record Setup: Create a Failover routing policy for the domain api.yourcompany.com.
2. Primary Record (Hot Cluster):
    - Points to the public IP/hostname of the NGINX Load Balancer in the hot cluster.
    - A health check is associated with this record.
3. Secondary Record (Standby Cluster):
    - Points to the public IP/hostname of the NGINX Load Balancer in the standby cluster.
    - An identical health check is associated with this record.
4. Health Check Configuration:
    - Endpoint: A publicly accessible health check endpoint on the Ingress controller (`/health` on the default backend).
    - Failure Threshold: The number of consecutive failed checks required to mark an endpoint as unhealthy (e.g., 3).
    - Interval: How often to perform the check (every 10 seconds).

### Failover Process:
1. The CloudFlare constantly sends requests to the health check endpoint of both clusters.
2. If the hot cluster's endpoint fails to respond correctly for the configured threshold, the CloudFlare marks it as unhealthy, and routes the traffic to the standby cluster.

## Networking and Security
I implemented a simple zero-trust network model within each cluster's `developer-services` namespace.
- Default Deny: A policy is in place to deny all ingress and egress traffic by default.
- Explicit Allow Rules: Policies are added to specifically allow necessary traffic flows, such as allowing ingress from the NGINX controller pods to the application pods.
- Secrets must be stored in proper solutions (Vault, AWS).
- Base Images: Application images should be built from trusted base images like alpine, to reduce the attack surface.
- Use Non-Root User.

## Monitoring and Logging

- Monitoring: Prometheus & Grafana
- Logging: Loki & Promtail
- Deployment: The entire stack is deployed via Helm charts into the dedicated monitoring namespace in both clusters.

### Monitoring (Prometheus & Grafana)
- Prometheus: Scrapes metrics from various sources via service discovery:
- Grafana: Provides visualization and alerting, using pre-built dashboard that are installed automatically.

### Logging (Loki & Promtail)
- Promtail: Deployed as a DaemonSet to run on every node. It automatically discovers logging sources, collects logs from all pods.
- Loki: A log aggregation system optimized for the Prometheus label-based query model.

# Useful Commands

Create proxy to access grafana
`kubectl port-forward --namespace monitoring svc/prometheus-grafana 8080:80`

