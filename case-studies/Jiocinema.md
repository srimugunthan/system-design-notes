This is a reorganization of the key points discussed in the YouTube video "How JioCinema live streams IPL to 20 million concurrent devices w/ Prachi Sharma | Ep 7" by Arpit Bhayani, featuring Prachi Sharma, Senior Engineering Director at JioCinema.

The discussion focuses on the extensive engineering, preparation, and operational strategies required to manage a live streaming event at the scale of the Indian Premier League (IPL).

### **1. Preparation and Planning**

- **Timeline:** Planning for an IPL season begins approximately **3 to 4 months** before the first match (e.g., November/December for a March start) [05:22].
- **Day One Effort:** The first match of any season takes a "village" to run, though the goal is to make subsequent days run as a "business as usual" operation with minimal manual intervention [01:43].
- **Audit Review:** The preparation starts with an extensive **Audit Review** where service owners must define the **breaking point** of their respective systems based on CPU, memory, CDN, network, and database capacity [07:04].
- **Partner Coordination:** The audit process is not limited to internal systems; it also involves ensuring all external partners (Cloud Providers, CDNs, Payment Gateways) are equally prepared for the anticipated traffic [08:12].

### **2. Operations on Match Day (The War Room)**

A typical match day routine kicks off a couple of hours before the 7:30 PM start time [02:02]:

- **War Room Setup:** All on-call engineers, game marshals, and partners (cloud, CDN, payment) join a central call or physical war room [02:16].
- **Scale-Up:** Infrastructure (compute and databases) is scaled up in advance of the traffic spike [02:35].
- **Metrics Review:** Every on-call team reviews all dashboards and alerts to ensure every system is in a "green" (healthy) state [02:41].
- **Post-Match Debrief:** A call is held after the game to discuss any issues and determine immediate improvements for the next day [03:02].

### **3. Front-End Strategy and Resilience**

The front-end (client apps) has a robust strategy to ensure the live stream plays, even if auxiliary features fail:

- **Code Freezes:** Client systems enter a code freeze state days before the first match, with the most stable build pushed to 100% of the audience [09:07].
- **Feature Flags:** All new features are deployed behind **feature flags** to allow engineers to quickly disable a feature on a specific platform (e.g., Android) if it causes stability issues, without shipping a new app version [10:27].
    - Flags can be tuned based on **App Version**, **Geography**, or **Platform** [14:56].
- **Simulations:** Engineers use tools like Charles to simulate real-world failures such as API 5xx errors, high latency, and DNS failures, ensuring the app remains usable for the customer [11:08].
- **Graceful Degradation:** Features are classified to prioritize the core user experience:
    - **P-Zero (Critical):** Live stream playback and Ad delivery. This path must not fail [15:30].
    - **P-One/P-Two (Non-Critical):** Extra features, like personalization in banners, which can be turned off or allowed to fail silently without showing the customer an error message [12:47].
- **Exponential Backoff:** The client avoids aggressive retries (e.g., retrying an API call three times quickly) to prevent the client layer from overwhelming the already "running hot" back-end systems [13:21]. A strategy like **exponential backoff** with a retry ceiling is used instead [13:46].

### **4. Back-End and Infrastructure Scaling**

Scaling databases is identified as the "most brittle component" of the architecture [21:05].

- **Auto-Scaling is Inadequate:** Auto-scaling is not a viable option because the hockey-stick traffic spikes are too fast, and the time required for a database to spin up new nodes (e.g., 45 minutes) is too long for a 3.5-hour game [18:21].
- **Pre-Scaling:** Capacity is provisioned and scaled up manually well ahead of the match after performing a **back-of-the-envelope calculation** [20:20].
    - Calculations start by determining the traffic a single Kubernetes pod can handle at a safe capacity (e.g., 60% CPU) and then projecting the total nodes needed for the expected peak concurrency [19:18].
- **"Panic Modes" (Static Snapshot Failover):**
    - To safeguard against database collapse, a pre-calculated, static snapshot of the API response (e.g., for the catalog/home page) is taken before the match and stored in static storage [23:13].
    - If the database system fails mid-match, the CDN is automatically switched to serve the static snapshot, ensuring the customer can still open the app and find the live match without seeing an error [23:41].
- **Scale-Down Policy:** Scaling down capacity is **never done during the match** [26:21]. It occurs after the match in carefully planned steps ("ladders") based on the drop in customer count [26:45].

### **5. Multi-CDN and Caching Strategy**

- **Multi-CDN:** Multiple CDNs are used to prevent reliance on a single provider, as CDNs also have capacity limitations [28:15].
- **Multi-CDN Optimizer:** An in-house service constantly monitors the health and load of each CDN and actively tunes traffic distribution, deciding which CDN should serve requests from different regions [28:48].
- **Cache Offload:** The "real engineering" is focused on achieving a **90%+ cache offload** on the CDN layer, which saves significant load on the origin servers and reduces cost [29:20].
- **Image Stack Incident:** An unexpected surge of **10 million RPS** on the image CDN was traced to a new feature showing 50 stickers per customer request [30:18].
    - **Fix:** The images were bundled together and sent to the client as **Base64-encoded assets** in one API call, reducing the RPS to 100K [31:52].

### **6. Ads and Asynchronous Processing**

- **Ads (Server-Side Ad Insertion):** Ad delivery is considered a P-Zero (critical) feature [40:17].
    - A human operator listens to the stream director and manually triggers the ads insertion during breaks (e.g., strategic timeout) [41:10].
    - JioCinema uses **Server-Side Ad Insertion (SSAI)** and creates **cohorts** of people (e.g., by geography) to deliver a *little bit* of personalization/targeting for advertisers [43:22].
- **Asynchronous Systems (Kafka):**
    - Lower-priority (P1/P2) background jobs can be temporarily paused, and their data can be stored locally, to be processed **after the match** when the system load is lower [35:19].
    - P-Zero use cases, like the live view counter, are prioritized and allowed to run, as they can tolerate being eventually consistent [36:14].

### **7. War Story: DNS Failures**

- **Incident:** Customer support reported that an entire geographic area/ISP was unable to open the app [38:09].
- **Root Cause:** A smaller ISP was not refreshing its DNS entries, causing a DNS failure for JioCinema's domains [38:23].
- **Mitigation:** The client application was updated to detect DNS failure and switch to an **alternate public DNS resolver** (e.g., 8.8.8.8) to resolve the domains, allowing the app to open [38:52].
