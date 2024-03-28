# Postgress on EKS
## Highly scalable, fault-tolerant and low latency Postgress DB on AWS EKS Cluster

1. Spread accross multiple AZs
2. Multiple read replicas
3. Persistent storage with AWS EBS Multi-AZ deployment for low latency, elastic volume to auto grow storage
4. Caching with Redis Operator to improve performance
5. PostgreSQL Operator CRD - Zalando to simplify deployment, management, and scaling of database cluster
6. Cluster Autoscaler to auto-scale the the EKS
7. HPA to auto-scale pods

## Deployment Steps
### 1. Set up AWS EKS Cluster:
Create an EKS cluster spanning multiple Availability Zones (AZs) in your desired AWS region.
Configure the necessary IAM roles, VPC, and security groups for the cluster.

### 2. Install Redis Operator:
Install the Redis Operator in your EKS cluster using Helm:

```bash
helm repo add redis-operator https://charts.spotahome.com
helm install redis-operator redis-operator/redis-operator
```

### 3. Create Redis Cluster:
Create a Redis Cluster with the desired configuration (e.g., number of masters, replicas, storage) using the Redis Operator Custom Resource Definition (CRD):

```yaml
apiVersion: redis-operator.spotahome.com/v1
kind: RedisCluster
metadata:
  name: poeks-redis-cluster
spec:
  replicas: 3
  redisConfig:
    maxmemory: 1Gi
  storage:
    persistentVolumeClaim:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
```

### 4. Deploy PostgreSQL Operator:
Install the PostgreSQL Operator (e.g., Zalando's Postgres Operator) using Helm:

```bash
helm repo add postgres-operator https://postgres-operator.github.io/charts/
helm install postgres-operator postgres-operator/postgres-operator
```

### 5. Create PostgreSQL Cluster:
Create a PostgreSQL cluster with a primary instance and read replicas across multiple AZs using the PostgreSQL Operator CRD:

```yaml
apiVersion: acid.zalan.do/v1
kind: PostgresCluster
metadata:
  name: poeks-postgres-cluster
spec:
  teamId: "acid"
  volume:
    size: 10Gi
    storageClass: "gp2"
  numberOfInstances: 3
  users:
    poeksadmin:
      - superuser
      - createdb
  databases:
    poeksdb: poeksadmin
  replicas: 2
  serviceAnnotations:
    service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "3600"
```

### 6. Configure Persistent Storage:
- The PostgreSQL Operator will provision and attach EBS Elastic Volumes with AutoGrow enabled for persistent storage of PostgreSQL data.
- Configure the AutoGrow parameters (e.g., MaxSize, GrowthIncrementPercent, GrowthThresholdPercentage) according to your storage requirements.

### 7. Integrate Caching:
- Update your application to use the Redis Cluster deployed by the Redis Operator as the caching layer for frequently accessed data from the PostgreSQL database.
- Use the Redis Cluster's service endpoint or DNS name to connect to the Redis instances.

### 8. Set up Monitoring and Logging:
- Configure AWS CloudWatch to monitor PostgreSQL and Redis metrics using Prometheus exporters or CloudWatch Agent.
- Set up CloudWatch Logs to collect and analyze logs from the PostgreSQL and Redis clusters.

### 9. Configure Backups:
- Set up automated backups of your PostgreSQL and Redis data using AWS Backup or a third-party backup solution.
- Configure backup schedules, retention policies, and backup storage locations.

### 10. Secure the Cluster:
- Enable network policies to restrict access to the PostgreSQL and Redis clusters.
- Use AWS IAM and RBAC for controlling access to Kubernetes resources and the PostgreSQL and Redis Operators.
- Enable SSL/TLS communication for the PostgreSQL and Redis connections using AWS Certificate Manager or self-signed certificates.
- Store sensitive data (e.g., database credentials) in AWS Secrets Manager and mount them as Kubernetes Secrets in your pods.

### 11. Configure Load Balancing:
- Deploy a Kubernetes Service of type LoadBalancer to expose your PostgreSQL cluster to external clients.
- Configure routing rules in the load balancer to direct write operations (e.g., POST, PUT, DELETE requests) to the Endpoint Slice or Subset containing the primary PostgreSQL instance.
- Configure routing rules to direct read operations (e.g., GET requests) to the Endpoint Slice or Subset containing the read replicas.
- Implement load balancing across the read replicas using an appropriate algorithm (e.g., round-robin, least connections).

### 12. Automate Scaling:
- Configure Horizontal Pod Autoscaler (HPA) and Cluster Autoscaler to automatically scale your PostgreSQL and Redis clusters based on resource utilization or custom metrics.
- Use Topology Spread Constraints to distribute the PostgreSQL and Redis pods evenly across multiple AZs.
- Configure PV topology to provision PVs in the same AZ as the node where the PostgreSQL pod is scheduled.

### 13. Implement Preemptive Node Draining:
- Set up a preemptive node draining strategy using tools like the Kubernetes Node Termination Handler or cloud provider's node lifecycle management capabilities.
- This ensures graceful eviction of pods from nodes before they are terminated, minimizing disruptions and data loss.
