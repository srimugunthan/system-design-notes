The transcript for the YouTube video **"How UPI Works Behind the Scenes – Backend Architecture Explained"** has been reorganized into the following key points:

### **Overview of the UPI Backend Architecture**

- The speed of UPI transactions (completing in seconds) is powered by an engineered back-end system that handles identity, routing, settlement, and fraud prevention in real-time [00:01].
- At the core is the **NPCI switch**, a centralized router that connects your Payment Service Provider (PSP) to the receiver's bank, ensuring every transaction flows through it [00:32].
- Despite interacting with various components—PSBs, identity verifiers, fraud engines, bank APIs, and messaging buses—the entire transaction journey takes less than 3 seconds [00:44].

### **User Identity and Security**

- Each user has a **Virtual Payment Address (VPA)**, like `name@bank`, which is securely mapped to their actual bank account by the PSP [00:59].
- The VPA allows the system to locate the bank without exposing sensitive account details [01:10].
- Authentication is secured through device fingerprinting, PIN entry, and sometimes biometric verification, balancing security and convenience [01:17].

### **Transaction Flow and Routing**

- The payment request from your UPI app is first routed through your PSP's API gateway [01:31].
- This gateway acts as a security and validation layer, verifying signatures, checking rate limits, validating request payloads, and protecting against replay attacks before the request reaches the NPCI switch [01:37].
- Once at the NPCI switch, it identifies the beneficiary bank and simultaneously sends a **debit request** to the sender's bank and a **credit request** to the receiver's bank [01:58].
- These instructions are handled **synchronously** to maintain **atomicity**, meaning both the debit and credit must succeed, or neither does, ensuring consistency [02:14].

### **Bank Integration and Processing**

- Banks use UPI-compliant APIs that plug into their core banking systems [02:25].
- Many banks use internal message queues to separate UPI traffic from core systems, which helps maintain speed even during high load [02:30].
- The bank verifies the balance, performs the debit or credit, and responds instantly through its APIs [02:40].

### **Ensuring Consistency with Idempotency**

- Every transaction carries a unique **Unique Transaction Reference (UTR) ID** [02:49].
- PSBs and banks store this ID to implement **idempotency**, allowing the backend to know if a retried transaction (due to a network error) has already been processed [02:58].
- This mechanism prevents duplicate debits and ensures consistency even in high-volume or unstable network conditions [03:05].

### **Real-time Confirmation and Background Settlement**

- After confirmation of debit and credit, the NPCI switch sends an **asynchronous callback** to the PSP with the transaction status, timestamps, and error codes [03:16].
- The PSP uses this data to update the user's screen in real-time, often via websockets or push notifications, making the experience feel instant [03:30].
- The actual interbank settlement of funds happens in **batches** in the background, not instantly with the payment [03:44].
- NPCI collects records and sends them to the Reserve Bank of India (RBI), which then settles funds between banks via their nodal accounts, ensuring liquidity [03:55].

### **Fraud Prevention and System Uptime**

- Security includes two-factor authentication, PIN retry limits, and transaction timeouts [04:10].
- AI-powered fraud detection engines analyze behavioral patterns, device signals, and transaction velocity to flag suspicious activity in milliseconds [04:22].
- The system is built for uptime using **redundant data centers**, automatic failovers, retry queues, and health checks [04:36].
- PSBs use defense mechanisms like **circuit breakers** and **exponential backoff strategies** to prevent cascading failures and keep the network online even when individual components fail [04:47].

===

### System Design of UPI Payments

This video explains how the Unified Payments Interface (UPI), India's largest payment infrastructure, works by detailing the underlying systems and transaction flow.

### I. Overview of Traditional Digital Payments (Pre-UPI)

Before UPI, digital payments relied on a complex system involving multiple banks regulated by the Reserve Bank of India (RBI) [01:07].

- **Details Required for Transfer:** To send money from one bank (e.g., ICICI) to another (e.g., HDFC), a user needed comprehensive details about the recipient:
    - Account Number [02:14]
    - Bank Name [02:14]
    - Branch Code [02:18]
    - IFSC Code (Indian Financial System Code) [02:23]
- **Transfer Mechanisms:** Traditional systems used specific protocols based on urgency and amount:
    - **IMPS (Immediate Payment Service):** Used for smaller, instant payments [03:32].
    - **NEFT (National Electronic Funds Transfer):** Transfers were not immediate, typically taking 2-3 hours to reflect, used for larger amounts (e.g., ₹5 lakh, ₹10 lakh) [03:53].
    - **RTGS (Real Time Gross Settlement):** Used for large, real-time transfers (e.g., ₹5 lakh to ₹20 lakh) [04:25].

### II. Introduction to UPI and its Core Components

UPI's main goal was to simplify money transfers—just open the app, enter an ID, and the transfer is instant [05:11].

- **NPCI (National Payments Corporation of India):** UPI operates on a central system called NPCI, which acts as the **infrastructure** and the **backbone** that facilitates payments [05:28].
    - The NPCI network is a **secure, closed, and trusted environment** [07:05]. It only communicates directly with trusted partner banks (e.g., ICICI, HDFC, SBI) [06:06].
- **VPA (Virtual Payment Address):** Instead of using complex bank details, UPI uses a simple, shareable VPA for transactions [08:50].
    - The VPA format is a unique username followed by a UPI handle (e.g., `username@upi_handle`) [09:10].
- **Customer PSP (Payment Service Provider):** These are the mobile applications you use (e.g., Google Pay, PhonePe, Paytm). They provide the User Interface (UI) for initiating payments [07:37].
    - Customer PSPs cannot communicate directly with NPCI; they must **partner with a bank** to enter the NPCI network [10:32].

### III. The UPI Transaction Flow (Push Payment)

The following outlines a payment transfer from one user (Payer) to another (Recipient):

1. **Intent Creation:** The Payer uses their Customer PSP app (e.g., PhonePe) to create a payment intent, specifying the amount and the Recipient's VPA [11:15].
2. **Request Routing:** The Payer's PSP sends this request to its **Partner Bank** (e.g., Yes Bank) [11:46].
3. **NPCI Communication:** The Partner Bank, being a trusted entity, forwards the payment request to the **NPCI network** [12:02].
4. **Verification and Authentication:**
    - NPCI requests the Payer's Partner Bank (Yes Bank) to **verify the payer's balance** [12:20].
    - The Partner Bank requests the user's **UPI PIN** for authentication via the PSP app [12:47].
5. **Fund Transfer:**
    - Upon successful authentication, the Payer's Partner Bank **Debits** the amount from the payer's account [13:07].
    - NPCI then instructs the **Recipient's Bank** (e.g., ICICI) to **Credit** the amount to the recipient's account [13:32].
6. **Acknowledgement and Completion:**
    - The transaction is only marked complete after both the debit and credit banks **acknowledge** the transaction to NPCI [17:47].
    - If the recipient's bank cannot acknowledge the transfer (e.g., server busy), the transaction is **Rolled Back** (the money is credited back to the payer's account) [17:54].
7. **Notification:** Once complete, the Recipient's Bank pushes a notification to the Recipient's PSP (e.g., Google Pay), which is then displayed to the user [18:43].

### IV. System Scale and Complexity

The underlying NPCI infrastructure is a massive engineering feat [24:34].

- The system is designed to be **fault-tolerant**, handling an immense load and processing **billions of transactions every second** [24:43].
- There is also a **Pull Mechanism** (e.g., payment requests, online merchant payments) that also works through the NPCI network [22:23].

---

The video is available here: System Design of UPI Payments

System Design of UPI PaymentsPiyush Garg · https://www.youtube.com/watch?v=fqySz1Me2pI


