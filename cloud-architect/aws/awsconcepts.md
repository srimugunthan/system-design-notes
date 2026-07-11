This video by *Web Dev Cody* provides a comprehensive overview of core **AWS architecture** for building scalable SaaS and web applications. While noting that AWS may not be the cheapest option compared to a simple VPS, he emphasizes its suitability for enterprises due to built-in scalability, reliability, and security features.

### **Core AWS Services Overview:**

* **Compute & Containers:**
    * **ECS (Elastic Container Service):** Recommended for running containerized applications, utilizing *Fargate* for simple or heavy compute tasks (1:33 - 2:04).
    * **ECR (Elastic Container Registry):** Used to store container images for ECS (2:24 - 2:34).
    * **AWS Lambda:** A serverless environment for running asynchronous or event-driven jobs, often paired with **SQS** (Simple Queue Service) for handling long-running tasks (4:02 - 5:07).

* **Storage & Databases:**
    * **RDS (Relational Database Service):** The primary recommendation for relational databases like *Postgres*, offering managed backups and automated updates (2:47 - 4:02).
    * **S3:** A cost-effective, scalable solution for file storage, which can trigger other services like Lambda (8:11 - 8:55).

* **Networking & Security:**
    * **CloudFront:** A CDN used to cache content and improve application performance (7:32 - 8:08).
    * **Route 53 & ACM:** Manages domain names and SSL/TLS certificates for HTTPS (9:57 - 11:15).
    * **IAM (Identity and Access Management):** Critical for managing permissions, roles, and security policies (11:53 - 12:42).
    * **Secrets Manager/SSM Parameter Store:** Secure locations for storing API keys and sensitive configuration data (12:45 - 13:23).

* **Observability & Management:**
    * **CloudWatch:** Essential for logging, monitoring metrics, and setting up alerts for the system (9:06 - 9:55).

* **Advanced Considerations:**
    * The speaker discusses using **WebSockets** (via API Gateway) for real-time notifications (6:11 - 6:28), and the importance of environment isolation (staging/production accounts) to ensure reliable deployments (15:21 - 16:06).
