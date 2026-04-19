The video **"Scaling Hotstar for 25 million concurrent viewers"** features Gaurav, a Cloud Architect at Hotstar, explaining the infrastructure challenges and solutions used to handle record-breaking traffic during the 2019 ICC Cricket World Cup.

Here is the reorganization of the transcript into key points:

### **1. Record-Breaking Traffic Patterns**
* **The "Tsunami" Effect:** Traffic is highly volatile, shifting from 2 million to 15 million or 25 million users in minutes based on match events like a player coming to bat or a wicket falling [[03:05](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=185)].
* **Dhoni’s Impact:** During the India vs. New Zealand semi-final, the platform added **1.1 million users per minute** when MS Dhoni was batting, eventually peaking at **25.3 million concurrent viewers** [[04:12](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=252)].
* **Bandwidth Dominance:** At its peak, Hotstar consumed **10 terabytes of video bandwidth per second**, accounting for roughly **70–75% of India's total internet bandwidth** at that moment [[07:13](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=433)].

### **2. Scaling Strategy & Infrastructure**
* **Proactive Scaling:** Traditional auto-scaling (e.g., AWS Auto Scaling) is too slow for 1 million new users per minute. Hotstar scales up **proactively in advance** based on match context [[13:06](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=786)].
* **Custom Scaling Tools:** They developed an in-house tool that scales based on **concurrency and request rates** rather than CPU or memory, which are lagging indicators [[21:46](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=1306)].
* **Fully Baked AMIs:** To minimize boot time, they use fully baked Images (AMIs) and container images. They avoid configuration tools like Chef or Puppet during scaling because they add several minutes of delay [[15:03](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=903)].
* **Diversification:** To avoid "Insufficient Capacity" errors in public clouds, they use **Spot Fleets** and **Secondary Auto Scaling Groups** with different instance types across multiple Availability Zones [[24:21](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=1461)].

### **3. Preparation via "Project Cult"**
* **Massive Load Testing:** They use an in-house project called **Project Cult** to simulate massive loads using over **3,000 high-end machines** (c5.9xlarge) [[09:52](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=592)].
* **Geo-Distributed Testing:** To avoid overwhelming a single region or CDN edge, they distribute load generation across **eight different AWS regions** globally [[10:39](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=639)].
* **ML for Traffic Prediction:** Machine learning is used to analyze historical data to predict how traffic will spike depending on which teams are playing or who is batting [[11:28](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=688)].

### **4. Chaos Engineering & Resilience**
* **Breaking the System:** They perform "Game Days" where they intentionally delete route tables, kill network connectivity, or shut down DBs to see how the system reacts [[25:51](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=1551)].
* **Cascading Failures:** A major focus is preventing a latency increase in one non-critical service (like recommendations) from slowing down the entire app [[26:42](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=1602)].
* **Handling the "Exit" Spike:** When a match ends or a key player gets out, users often hit the "back" button simultaneously. This shifts load from video playback to the **homepage API**, which must be scaled to handle 25 million hits at once [[05:46](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=346)].

### **5. "Panic Mode" & Graceful Degradation**
* **Service Prioritization:** Services are ranked by importance. **P0 services** (Video, Ads, Payments, Subscription) must always stay up [[35:40](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=2140)].
* **Turning Off Features:** In "Panic Mode," non-essential features like **personalization and recommendations** are turned off to save resources for video delivery [[35:03](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=2103)].
* **Bypassing Failures:** If the login or payment system fails, they may put it in panic mode to allow users to **watch for free/without login** rather than showing an error message, prioritizing user experience over immediate business rules [[37:39](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=2259)].

### **Key Takeaways**
* **Understand the User Journey:** You must know every API call triggered by a user action to build a resilient system [[39:30](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=2370)].
* **Failure is Inevitable:** Design the system to **degrade gracefully** so that the user never sees a "connection failed" message [[40:08](http://www.youtube.com/watch?v=QjvyiyH4rr0&t=2408)].



http://googleusercontent.com/youtube_content/0
