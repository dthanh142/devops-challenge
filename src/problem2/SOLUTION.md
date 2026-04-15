
<img width="1919" height="1123" alt="Screenshot 2026-04-15 at 19 40 40" src="https://github.com/user-attachments/assets/451ed9b3-40db-4108-80eb-0ae510db5ee5" />



## 1. Architecture:
- Microservice-based, event-driven trading platform
- Multi AZ deployment to ensure high availability
- Seperated subnet for each layer

### Layer 1: Edge/CDN/Security
| Service | Role | Alternatives |
| --- | --- | --- |
| Route 53 | DNS with healthcheck and latency-based routing | Cloudflare DNS |
| CloudFront | Caching at edge for static files. Can be integrated with WAF and Shield for better protection | Akamai |
| AWS WAF | Web firewall, blocking common web (SQL injection, XSS, bot traffic...). Sits in front of API gateway to filter the malicious traffics | Cloudflare WAF |
| AWS Shield | DDoS protection | Cloudflare Spectrum |
| ACM | SSL/TLS certificate management | Let's Encrypt |
| Cognito | User authentication, authorization, and identity management |  |
| Secrets Manager | Secure secret storage and rotation for tokens, API keys, and DB credentials | HashiCorp Vault, SSM Parameter Store |

### Layer 2: Application
| Service | Role | Alternatives |
| --- | --- | --- |
| API gateway | Centralized API management, authentication, throttling, request transformation | Kong |
| AWS ALB/NLB | Load balancing traffic to EKS, SSL termination, health checks, failover | NGINX Load Balancer, HAProxy |
| EKS cluster | Container orchestration for microservices (Auth, Order, Price, Balance services) | ECS, self-managed K8s |
| Karpenter | Auto-scaling for EKS nodes based on workload | Cluster Autoscaler, KEDA |
| Lambda | Serverless functions for event processing and lightweight tasks |  |

### Layer 3: Event streaming and messaging
| Service | Role | Alternatives |
| --- | --- | --- |
| AWS Managed Kafka | Real-time event streaming for order matching, price updates, trade events, and market data feeds, connects to EKS microservices for event-driven workflows | AWS Kinesis |
| SQS+SNS | Asynchronous messaging for notifications, decoupling services | RabbitMQ |


### Layer 4: Database
| Service | Role | Alternatives |
| --- | --- | --- |
| Aurora | Primary relational database for transactional data (orders, user data) | RDS MySQL, Google Cloud SQL |
| ElastiCache | In-memory caching for sessions, balances, real-time data, also supports pub/sub messaging for real-time notifications, session storage | Memcached |
| DynamoDB | NoSQL database for high-throughput, flexible data (e.g., trade history) | MongoDB Atlas, Cassandra |
| S3 | Object storage for logs, backups, static assets | Google Cloud Storage, Azure Blob |


### Layer 5: Observability and security
| Service | Role | Alternatives |
| --- | --- | --- |
| CloudWatch | Monitoring metrics, logs, alerts for performance and errors | Prometheus, ELK |
| CloudTrail | Audit logging for API calls and account activity | Splunk |
| VPC flow logs | Network traffic monitoring for security analysis |  |
| X-Ray | Distributed tracing for request flow and latency debugging | Jaeger, Zipkin |
| Guard Duty | Threat detection service; monitors EKS clusters, IAM activity, and VPC traffic for suspicious behavior such as cryptomining, unauthorized API calls, or compromised credentials; integrates with CloudWatch for alerting and SNS for notifications | Wiz |


### Layer 6: CI/CD and Delivery
| Service | Role | Alternatives |
| --- | --- | --- |
| CodeBuild | Managed build service for compiling and testing code | GitHub Actions, Jenkins |
| CodePipeline | CI/CD pipeline orchestration for automated deployments | GitLab CI, CircleCI |
| ECR | Container registry for storing and managing Docker images | Docker Hub, Google Container Registry |
| CodeDeploy | Automated deployment to EKS or EC2 | ArgoCD, Flux |


---
## 2. Scaling and Recovery plan:

### Scaling Strategies
- **Horizontal Scaling**: 
  - EKS pods auto-scale via HPA based on CPU/memory (target: 70% utilization).
  - Karpenter provisions nodes dynamically for burst traffic.
  - ALB scales automatically with traffic; add more AZs for global distribution.
  - DynamoDB and Aurora scale read replicas on demand.

- **Vertical Scaling**:
  - Increase EKS node sizes (e.g., from m5.large to m5.xlarge) for sustained load.
  - Upgrade RDS Aurora instances for higher IOPS.
  - Add shards to Managed Kafka for throughput beyond 500 RPS.

- **Event-Driven Scaling**:
  - Use CloudWatch alarms to trigger scaling (e.g., >80% CPU triggers pod scale-out).


- **Growth Beyond Current Setup**:
  - **For 10x RPS (5,000 RPS)**: Implement multi-region deployment with Route 53 geo-routing.
    - Use AWS Global Accelerator for intelligent traffic routing and edge optimization.
    - Shard Aurora databases by region or customer; implement read/write splitting.
    - DynamoDB global tables for cross-region replication and low-latency reads.
    - Consider RDS Proxy for connection pooling at scale.
    - Increase Kafka brokers to handle 10x throughput; partition topics by symbol or region.
    - Cluster ElastiCache across regions; use Redis replication for failover.
 

### Recovery and Failover
- Multi-AZ deployment for EKS, RDS, ElastiCache.
- Route 53 health checks redirect traffic to healthy regions.
- Automated RDS snapshots (daily) and S3 cross-region replication.
- Use AWS Backup for point-in-time recovery.
- CodeDeploy blue/green deployments for zero-downtime updates.
- Aurora failover in <30s; DynamoDB global tables for cross-region sync.

### Cost Optimization:
- Use Reserved Capacity for all critical services (RDS, Kafka, DynamoDB).
- Scale down during off-peak
- Use AWS Compute Savings Plans for EKS nodes.
- Use Spot Instances for non-critical workloads via Karpenter.
- Set budgets in CloudWatch; optimize S3 storage classes.
- Consider moving to opensource solution for CICD or obsevasion services like: ELK, prometheus, grafana, Jenkins, GitlabCI...
