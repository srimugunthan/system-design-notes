=

# System-design: How to Practice

**System design**

Master all concepts:

**The "Explain Like I'm 5" Tutor**

**Socratic Quiz Master:**

**Common Questions on design. Generate .md files**

- **Tech Stack Selection:** Ask for recommendations based on your non-functional requirements.
    - *Prompt:* "I'm designing a real-time analytics dashboard that needs to ingest millions of events per second and provide sub-second queries on aggregated data. Compare Apache Kafka, Amazon Kinesis, and RabbitMQ for the ingestion layer. Also, compare Apache Druid, ClickHouse, and Elasticsearch for the storage/query layer. What would be your recommended stack and why?"

**The Devil's Advocate & Risk Analyst:**

- **The Goal:** To proactively identify weaknesses, bottlenecks, and failure points in your designs.
- **How to Leverage AI:**
    - **Stress Test Your Design:** Present your finished design to AI and ask it to try to break it.
        - *Prompt:* "Here is my high-level design for a chat application (describe components: websocket servers, message queue, database, etc.). Please act as a malicious user and a traffic spike. What are the top 5 ways this system could fail? How would you recommend I mitigate each risk?"
    - **Trade-off Analysis:** Challenge your decisions to ensure you've considered all angles.
        - *Prompt:* "I chose a SQL database for this social media feed feature because I need strong consistency for comments. My colleague argues for a NoSQL database for better scalability. Can you provide a balanced argument for both sides, outlining the specific trade-offs in performance, scalability, and development complexity?"
- **Infrastructure as Code (IaC):** Ask AI to generate Terraform or CloudFormation scripts.
    - *Prompt:* "Generate a Terraform script to provision the core infrastructure for our URL shortener on AWS. We'll need a VPC, public subnets, an EC2 instance for the API, and an RDS PostgreSQL instance. Use best practices for security groups."

**Practise Estimation Questions:**

- **The Goal:** To build your intuition for system capacity, a hallmark of a senior architect.
- **How to Leverage AI:**
    - **Traffic and Storage Estimations:** Ask AI to help you with the back-of-the-envelope calculations.
        - *Prompt:* "For a Twitter-like service with 500 million users, 50% of whom are daily active, and each user posts 1 tweet per day on average, with 20% of tweets containing a picture (500KB each). Estimate the daily storage required for tweets and images. Also, estimate the peak writes per second for tweets."
