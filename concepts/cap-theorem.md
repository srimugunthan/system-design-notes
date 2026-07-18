### **CAP Theorem in System Design Interviews**

The video discusses the CAP Theorem, defining it and explaining its crucial role in determining non-functional requirements during a system design interview.

---

### **1. Defining the CAP Theorem** [00:41]

The CAP theorem states that a distributed system can only guarantee two out of the following three properties:

- **C - Consistency:** All users see the same data at the same time. [01:02:00]
- **A - Availability:** Every request gets a response, whether the operation was successful or not. [01:07:00]
- **P - Partition Tolerance:** The system continues to operate despite network failures between nodes. [01:14:00]

---

### **2. Why the CAP Theorem Matters in an Interview** [01:21:00]

When discussing non-functional requirements (qualities of the system) with an interviewer:

- **Partition Tolerance (P) is a Must:** In a distributed system (which is almost always the case in system design interviews), partition tolerance is a necessity. [01:51:00]
- **The Core Decision is C vs. A:** Your first major decision is to choose whether to prioritize **Consistency (C)** or **Availability (A)**. [02:02:00]
- This decision will have a **significant influence** on your overall system design later in the interview. [02:15:00]

---

### **3. The Conflict Between Consistency and Availability** [02:15:00]

The choice between C and A becomes necessary in the event of a **network partition** (failure).

- **Scenario:** User A updates data on Server A, but a network failure occurs before the data is replicated to Server B. [03:05:00]
- **Decision Point for User B reading from Server B:**
    - **Prioritize Consistency (C):** Stop serving the data and give an error because the data is stale. [03:39:00]
    - **Prioritize Availability (A):** Continue serving the stale data, risking the user seeing incorrect information for a period of time. [03:54:00]

---

### **4. Choosing Consistency (CA) or Availability (AP)**

| **Prioritize Consistency (CA) - Cannot risk stale data** | **Prioritize Availability (AP) - Can risk stale data** |
| --- | --- |
| **Ticket Booking Platforms:** To prevent double-booking of seats or tickets. [04:08:00] | **Social Media/Profile Data:** Seeing an old name or a post update late is acceptable. [05:53:00] |
| **Inventory Systems:** To prevent selling the last item to multiple users. [04:42:00] | **Review Services (e.g., Yelp):** Slightly out-of-date business information is fine. [06:03:00] |
| **Financial Systems:** The order book must be up-to-date (strong consistency). [05:04:00] | **Streaming Services (e.g., Netflix):** Not seeing a description or movie update immediately is acceptable. [06:26:00] |

---

### **5. Design Influence: Implementing the Choice** [07:16:00]

| **If you chose Strong Consistency (C)** | **If you chose Availability (A)** |
| --- | --- |
| **Implementation:** May need to implement **distributed transactions** between components (e.g., cache and database). [07:21:00] | **Implementation:** Use **multiple replicas** and scale out the system. [08:50:00] |
| **Latency:** Must **accept higher latency** while waiting for data propagation. [08:10:00] | **Consistency Model:** **Eventual consistency** is acceptable. [09:00:00] |
| **Tools:** Traditional RDBMS (Postgres, SQL), Google Spanner, or NoSQL databases configured for strong consistency. [08:21:00] | **Tools:** DynamoDB (in multi-AZ configuration without strong consistency), Cassandra, or using Change Data Capture (CDC). [09:04:00] |

---

### **6. Advanced Nuance (Senior/Staff Level)** [09:22:00]

- **Mixed Requirements:** Different parts of the *same* system can prioritize different requirements.
    - **Ticketmaster Example:** Consistency for **booking tickets**, but Availability for **viewing event details**. [09:38:00]
    - **Tinder Example:** Consistency for **matching** (immediate match notification), but Availability for **viewing/updating profile data**. [10:26:00]

- **Levels of Consistency:** You can be more specific than just "Strong Consistency":
    - **Causal Consistency:** Ensures related events appear in the correct order (e.g., a reply comment is shown after the original comment). [11:39:00]
    - **Read Your Own Writes Consistency:** Guarantees that a user immediately sees the results of their *own* update, but other users may still see stale data. [12:11:00]
    - **Eventual Consistency:** The system will eventually become consistent after a period of time. This is the model when prioritizing Availability. [12:41:00]

---

**Video Details:**

- **Title:** CAP Theorem in System Design Interviews
- **Channel:** Hello Interview - SWE Interview Preparation
- **URL:** http://www.youtube.com/watch?v=VdrEq0cODu4
