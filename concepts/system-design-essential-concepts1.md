In the video "system design stuff you should know," Arjay McCandless categorizes various system design concepts and tools into a tier list based on their importance for software engineers to learn and implement.

### **S-Tier: Essential Fundamentals**
* **REST APIs:** One of the most important patterns; widely understood and used in almost every application. [[04:26](http://www.youtube.com/watch?v=UWNDvomXKEg&t=266)]
* **Databases:** Every application needs persistent storage; he recommends starting with **Postgres** as it is the industry standard. [[05:08](http://www.youtube.com/watch?v=UWNDvomXKEg&t=308)]
* **Load Balancers:** Essential for horizontal scaling, which is the only way to achieve massive scale. [[03:56](http://www.youtube.com/watch?v=UWNDvomXKEg&t=236)]
* **CDNs (Content Delivery Networks):** Crucial for serving static data quickly and cheaply by bringing it closer to users. [[04:45](http://www.youtube.com/watch?v=UWNDvomXKEg&t=285)]
* **Monitoring:** Vital for understanding system health; without it, you are just waiting for customers to report problems. [[06:04](http://www.youtube.com/watch?v=UWNDvomXKEg&t=364)]
* **CI/CD:** Allows for instant shipping of features, which translates directly to business value and fewer production bugs. [[06:46](http://www.youtube.com/watch?v=UWNDvomXKEg&t=406)]
* **Containers (Docker):** Fundamental for deployment, scaling, and testing; highly recommended to learn by building one. [[07:18](http://www.youtube.com/watch?v=UWNDvomXKEg&t=438)]

### **A-Tier: Highly Valuable**
* **Object Storage (e.g., S3):** Used for storing files and images; it is cheap, durable, and reliable. [[03:05](http://www.youtube.com/watch?v=UWNDvomXKEg&t=185)]
* **Queues:** Useful for handling asynchronous workloads and buffering spiky traffic. [[03:25](http://www.youtube.com/watch?v=UWNDvomXKEg&t=205)]
* **Authentication:** Essential but warns against "rolling your own"; suggests using providers like Auth0, Clerk, or Supabase. [[06:31](http://www.youtube.com/watch?v=UWNDvomXKEg&t=391)]
* **Data Pipelines:** Increasingly important with the rise of AI, as they facilitate moving data from point A to point B. [[08:11](http://www.youtube.com/watch?v=UWNDvomXKEg&t=491)]

### **B-Tier: Useful, but Use with Caution**
* **Serverless Architecture:** Great for mid-sized companies, but can become impractical and expensive at a very large scale. [[02:18](http://www.youtube.com/watch?v=UWNDvomXKEg&t=138)]
* **Search (Elasticsearch/OpenSearch):** Powerful for fuzzy search, but often implemented before it is truly necessary. [[01:41](http://www.youtube.com/watch?v=UWNDvomXKEg&t=101)]
* **Real-time Architecture:** Often over-prioritized; many use cases are better served by simpler asynchronous processes. [[02:02](http://www.youtube.com/watch?v=UWNDvomXKEg&t=122)]
* **Caching:** Powerful but adds significant complexity; he advises using it only when you are sure you need it. [[05:25](http://www.youtube.com/watch?v=UWNDvomXKEg&t=325)]
* **Websockets:** Useful for specific real-time needs but difficult to maintain; often over-utilized. [[05:45](http://www.youtube.com/watch?v=UWNDvomXKEg&t=345)]

### **C-Tier & D-Tier: Niche or Overrated**
* **Microservices (C-Tier):** Mostly beneficial for high development velocity in very large teams; often unnecessary for solo developers. [[01:13](http://www.youtube.com/watch?v=UWNDvomXKEg&t=73)]
* **Data Warehouses (C-Tier):** Useful for business intelligence but usually handled by dedicated data engineers. [[07:33](http://www.youtube.com/watch?v=UWNDvomXKEg&t=453)]
* **Feature Flags (C-Tier):** Useful in massive systems, but can be annoying to implement and aren't a priority for new devs. [[07:47](http://www.youtube.com/watch?v=UWNDvomXKEg&t=467)]
* **GraphQL (D-Tier):** An alternative to REST that often adds more complexity than it solves; 99% of apps should stick to REST. [[00:48](http://www.youtube.com/watch?v=UWNDvomXKEg&t=48)]



http://googleusercontent.com/youtube_content/0
