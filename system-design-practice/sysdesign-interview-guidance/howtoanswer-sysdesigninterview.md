# Structuring the system design interview

**System design**

**Step1: Clarify the Problem Scope and Requirements**

In my early interview days, I learned the hard way that jumping straight into solutions without fully understanding the problem is a red flag. Just like in coding interviews, taking a moment to clarify can make all the difference. Asking detailed questions to grasp what the interviewer expects ensures that you start on the right foot.

**Step2 : Pause and Organize Your Thoughts**

Thinking of a solution on the fly can make anyone nervous. My trick is to request a one- or two-minute break to organize thoughts and draft a solution. This simple act helped me calm down and think more clearly. It’s perfectly acceptable, and I’ve seen many candidates do the same thing. Pausing can be the key to moving forward more effectively.

**Step3: Provide the Outline of the Solution**

By outlining the solution first, I can show that I’ve considered the problem from all angles in the big picture. This sets a solid foundation for a deeper dive into each point. This approach demonstrates my logical thought process and reassures the interviewer that I have a comprehensive plan. I also use the outline to communicate with the interviewer about which areas to focus on more and what the remaining parts entail.

**Step4: Lead the Conversation**

Unlike other interview rounds that are more Q&A-based, ML system design interviews require you to take the lead. After clarifying the problem, it’s important to proactively explain your thoughts and considerations instead of waiting for the interviewers to ask you “what if” questions. I found that regularly pausing to ask if the interviewer had any questions or needed further elaboration was crucial. This ensured that they followed my reasoning and kept the conversation on track. It’s essential to be mindful of their cues and be prepared to dive deeper or move on based on their feedback.

**Step5: Provide Trade-offs and Rationales**

There’s rarely a single best solution in ML system design. During the interviews, I make sure to discuss the trade-offs and rationales behind my decisions. This approach demonstrates not only my knowledge but also my ability to think critically about different options. By explaining why you chose one solution over another, you show a deep and broad understanding of the subject, which can significantly impress the interviewer.

--
### **System Design Interview Framework: Four Steps**

### **Step 1: Understand the Problem and Establish Design Scope** [01:36] (Approx. 5 minutes)

The goal is to fully understand the vague, open-ended problem and set the focus before proposing a solution [01:45].

- **Functional Requirements:** Clarify the requirements by asking:
    - Why are we building the system?
    - Who are the users?
    - What features are needed (e.g., one-on-one chat vs. group chat)? [02:07]
    - Focus on the top few features in **priority order** and get the interviewer to agree to the list [02:34].
- **Non-Functional Requirements (NFRs):** Clarify NFRs, focusing on **scale and performance** [02:48].
    - NFRs are what make a design unique and challenging (e.g., designing for hundreds of millions of users) [02:57].
    - For senior roles, demonstrating an ability to handle NFRs is critical [03:13].
- **Back-of-the-Envelope Calculations:** Perform rough estimations to get a general sense of the system's scale and potential challenges/bottlenecks [03:27].
    - The goal is to get the correct **order of magnitude** [03:41].
- **Deliverable:** A short list of features and important non-functional requirements [03:47].

### **Step 2: Propose High-Level Design and Get Buy-in** [03:53] (Approx. 20 minutes)

The goal is to develop a top-down design and reach an agreement with the interviewer [04:02].

- **Start with APIs:**
    - Establish the contract between end-users and the backend systems [04:09].
    - Follow the **RESTful convention** and define the input parameters and output responses for each API [04:20].
    - Verify the APIs satisfy all functional requirements [04:35].
    - Consider **WebSockets** if two-way communication is needed, but be prepared to discuss the challenges of managing this stateful service at scale [04:43].
- **Lay out the High-Level Design Diagram:**
    - Start with a **Load Balancer** or **API Gateway** [05:28].
    - Include the core services that satisfy the feature requirements [05:36].
    - Introduce the **data storage layer** for persistence, deferring the exact technology choice to the Deep Dive section [05:40].
    - *Pro Tip:* Maintain a list of discussion points and **resist the temptation to dive into details too early** [06:20].
- **Data Model and Schema:**
    - Discuss the data access patterns, read/write ratio, database selection, and indexing options [06:40].
    - Review the high-level design to ensure every feature is complete end-to-end [07:08].

### **Step 3: Design Deep Dive** [07:21] (Approx. 15 minutes)

The goal is to demonstrate an ability to identify and solve potential problems, focusing on non-functional requirements and discussing trade-offs [07:28].

- **Identify Focus Areas:** Work with the interviewer to select one or two areas that need in-depth discussion, especially those related to scale and performance [07:35].
    - Be mindful of the interviewer's body language or clues that indicate dissatisfaction with certain aspects of the design [07:52].
    - List out your reasons for choosing a particular solution and ask if the interviewer has any questions or concerns [08:05].
- **Mini-Guidelines for Deep Dive Discussions:**
    1. **Clearly articulate the problem** (e.g., "The right QPS is too high for a single database") [08:26].
    2. **Come up with at least two solutions** (e.g., reduce update frequency or choose a NoSQL database) [08:41].
    3. **Discuss the trade-offs** of the solutions, using numbers to back up your design [08:57].
    4. **Pick a solution** and discuss it with the interviewer [09:02].
- **Limit:** In a typical interview, you should only have time to dive deeper into the top two or three issues [09:05].

### **Step 4: Wrap Up** [09:11] (Approx. 5 minutes)

The goal is to conclude the discussion smoothly and professionally.

- **Summarize the design** briefly, focusing on the parts that are unique or challenging to the problem [09:20].
- **Leave enough room** at the end of the interview for the interviewer to ask questions about the company or role [09:26].

#
******
# System Design Interview: The 45-Minute Blueprint

*A structured breakdown of the video "[How to Pass a System Design Interview (The 45-Minute Blueprint)](https://www.youtube.com/watch?v=HcC9Du6RWwk)" by Code with Lucian*

---

## 🎯 Key Mindset & Overview `[00:00:00]`

- **The Sifter Round:** While HR passes ~75% and coding rounds pass ~20%, system design cuts the remaining pool in half—making it the primary differentiator for senior placement and salary compensation.

- **Signal over Noise:** Interviewers evaluate how you handle ambiguity, justify trade-offs, structure thoughts, and handle system stress—not how many buzzwords you throw around.

- **Drive the Conversation:** Successful candidates take control immediately and proactively lead the discussion rather than waiting passively for questions.

---

## Phase 1: Minutes 0–5 — Scope & Requirement Boundaries `[00:01:05]`

### Avoid the Scope Trap
Don't start drawing database schemas immediately when given a vague prompt (e.g., "Design Uber").

### Functional Requirements (Keep it Tight)
Pick max 3 core features for the initial design (e.g., rider requests ride, driver accepts, real-time location tracking).

### Non-Functional Requirements & SLAs
- Clarify consistency expectations (e.g., strong consistency for payments vs. eventual consistency for driver locations)
- Establish latency budgets (e.g., location tracking latency < 500 ms)
- Define access patterns like the read-to-write ratio (e.g., 100:1 read-heavy vs. 99% write-heavy IoT logging)

### Avoid Premature Overengineering
Don't propose multi-region database sharding before establishing scale.

---

## Phase 2: Minutes 5–10 — Estimations, API Contracts & Storage `[00:03:10]`

### Order of Magnitude Estimations
- Interviewers test architectural justification, not exact arithmetic—round numbers aggressively
- **Example:** 10M daily users doing 10 actions/day = 100M events. Dividing by 100,000 seconds/day yields an average of 1,000 QPS (peak ≈ 2,000 QPS)
- **Network Bandwidth Check:** 2,000 QPS × 10 KB payload = 20 MB/s, which fits on a single instance without immediate partitioning needs

### API Contracts & Protocols
- Use **REST over HTTPS** for stateless transactional operations (e.g., ride booking)
- Use **WebSockets or Server-Sent Events (SSE)** for real-time bidirectional location streaming to minimize HTTP polling overhead

### Primary Data Storage
- **Relational (e.g., PostgreSQL):** For ACID compliance and transactional integrity (e.g., payments)
- **NoSQL / In-Memory (e.g., Redis, Cassandra):** For ultra-low latency, key-value lookups, or geospatial indexing (e.g., driver coordinates)

---

## Phase 3: Minutes 10–20 — High-Level Architecture `[00:05:29]`

### Start Minimal
Design day 1 before day 1000. Begin with a simple 5-box foundation:

```
Client → Load Balancer → API Gateway → Application Services → Database
```

### Trace the Happy Path
Walk the interviewer step-by-step through a single end-to-end request.

### Explain Component Purpose Out Loud
Never add a component without stating its exact purpose (e.g., "Adding a Load Balancer here to distribute peak 2,000 QPS traffic across stateless app servers and run health checks").

### Separate Read & Write Paths
Direct high-volume reads (e.g., map views) away from primary transactional databases toward read replicas or in-memory caches.

---

## Phase 4: Minutes 20–35 — Deep Dives, Bottlenecks & Hard Trade-Offs `[00:07:15]`

### Caching Strategies
- Implement **Cache-Aside** with Redis for read bottlenecks
- Manage memory using TTLs and **LRU (Least Recently Used)** eviction policies

### Database Scaling & Sharding
- Scale reads with **Asynchronous Read Replicas**
- Scale writes with **Horizontal Sharding**
- **Sharding Key Warning:** Avoid hot spots (e.g., sharding by `City_ID` breaks during New Year's Eve in NYC). Use composite keys like `City_ID + Hash(Driver_ID)`

### Asynchronous Decoupling
Use distributed message queues (e.g., Kafka) to handle write surges (e.g., driver location updates) and provide backpressure protection.

### CAP Theorem Trade-Offs
- **Location Tracking:** Prioritize **Availability (AP)** with eventual consistency (a 2-second location delay is acceptable)
- **Payment Transactions:** Prioritize **Consistency (CP)** (safely fail a transaction rather than double-charge)

### Single Points of Failure (SPOF)
Introduce multi-region automated failovers with health-check heartbeats and warm standby replicas.

---

## Phase 5: Minutes 35–45 — Resilience, Observability & Wrap-Up `[00:11:05]`

- **Distributed Tracing:** Inject a unique Trace ID at the API Gateway to track requests across microservices
- **Circuit Breakers:** Prevent cascading failures when downstream external dependencies fail or slow down
- **System Metrics:** Unmonitored systems are broken systems waiting to happen

---

## 🚨 3 Critical Red Flags That Lead to Rejection `[00:11:38]`

1. **Overengineering too early:** Adding complex distributed patterns before establishing baseline constraints
2. **Getting defensive:** Treating bottleneck questions as attacks rather than opportunities to discuss architectural trade-offs
3. **Poor time management:** Spending 25 minutes on math calculations and running out of time to build the actual architecture diagram
