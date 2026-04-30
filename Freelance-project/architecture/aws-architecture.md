# AWS Architecture - Healthcare ERP Cloud Migration

```text
                                 +-------------------------+
                                 |      Internet Users     |
                                 +------------+------------+
                                              |
                                              v
                                 +-------------------------+
                                 |        Route 53         |
                                 +------------+------------+
                                              |
                                              v
                      Public Subnet  +-------------------------+
  +------------------------------->  |      EC2 Instance       |
                                     | (IAM Role Attached)     |
                                     +------------+------------+
                                                  |
                                                  v
                                     +-------------------------+
                                     |          Nginx          |
                                     |  Reverse Proxy :80      |
                                     +-----+-------------+-----+
                                           |             |
                                           v             v
                                 +----------------+  +----------------+
                                 | Frontend       |  | Backend        |
                                 | Container:3000 |  | Container:8000 |
                                 +----------------+  +--------+-------+
                                                              |
                                                              v
                    Private Subnet                 +-------------------------+
  +----------------------------------------------> |   RDS PostgreSQL       |
                                                   |     (No Public IP)      |
                                                   +-------------------------+

        +------------------------+        +-------------------------+
        |    GitHub Actions      |------> |       Amazon ECR        |
        | Build + Deploy Runner  |        | frontend/backend images |
        +-----------+------------+        +------------+------------+
                    |                                  |
                    +--------------------------------->|
                                                       v
                                             +-------------------------+
                                             | EC2 pulls by git SHA    |
                                             +-------------------------+

     +----------------------+        +----------------------+
     | AWS Secrets Manager  |<-------| Backend Container    |
     +----------------------+        +----------------------+

     +----------------------+        +----------------------+
     |      CloudWatch      |<-------|   EC2 + RDS Metrics  |
     +----------------------+        +----------------------+
```

## Component Explanation

- **Route 53** provides DNS routing from the public domain to the EC2 instance.
- **EC2 in public subnet** hosts the container runtime and Nginx, with an attached IAM role granting scoped AWS API access.
- **Nginx** acts as the entry point, routing `/` to frontend and `/api` to backend.
- **Frontend + Backend containers** run as independent services and are deployed together via Docker Compose.
- **RDS PostgreSQL in private subnet** is only reachable from the EC2 security group, preventing internet-origin connections.
- **GitHub Actions + ECR** form the delivery pipeline: build immutable images tagged with git SHA, push to ECR, deploy to EC2.
- **Secrets Manager** stores runtime credentials and tokens consumed by backend startup logic.
- **CloudWatch** monitors EC2 and RDS resource health for performance and availability alerts.
