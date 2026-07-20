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
