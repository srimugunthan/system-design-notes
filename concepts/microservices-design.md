# Microservices design

- **Monolithic Architecture vs. Microservices:**
    - **Monolith:** One large codebase that becomes hard to manage and scale for large systems.
    - **Microservices:** Application broken down into smaller, independent services, each with its own database and logic, allowing them to scale and deploy independently.


# Section 1: Microservices Mental Model

### **SLIDE 1: Section Intro**

**Section 1: Microservices Mental Model**

Before learning how to design microservices, you need a clear mental model of what they actually are, what they promise, what they cost, and why they are fundamentally an organizational decision, not just a technical one.

Most engineers can say "microservices are small, independent services." That definition is incomplete and leads to poor design decisions. This section resets your understanding so that everything in the rest of the course builds on a solid foundation.

**What We Will Cover:**

1. What microservices actually mean (the precise definition that matters)
2. What microservices do not mean (common misconceptions that hurt you in interviews)
3. The three core promises: independent deployment, scaling, and ownership
4. The five core costs: network latency, partial failure, data inconsistency, debugging complexity, operational overhead
5. Microservices as an organizational pattern (Conway's Law)
6. How to frame microservices correctly in a system design interview

**Where This Appears in Real Systems:**

| Company-Type Example | How Microservices Mental Model Applies |
| --- | --- |
| E-commerce platform like Amazon | Hundreds of teams, each owning a service (catalog, cart, checkout, payments, shipping) |
| Streaming platform like Netflix | Separate services for user profiles, recommendations, playback, billing, content encoding |
| Messaging platform like WhatsApp | Independent services for message delivery, presence, media storage, notifications |

Every section in this course builds on the mental model established here. If you understand the promises, costs, and organizational nature of microservices, the decomposition, communication, and data ownership decisions in later sections will make intuitive sense.

---

### **SLIDE 2: What Microservices Actually Mean**

A microservice is a service that meets all four of these criteria simultaneously:

1. **Independently deployable** - Ships without coordinating with other services
2. **Owns its own data** - Has a private database that no other service can access directly
3. **Aligned to a business capability** - Represents a real business function, not a technical layer
4. **Owned by a single team** - One team builds, deploys, monitors, and is on-call for it

If a service meets only two or three of these, it is not a true microservice. It might be a well-structured component, but it does not deliver the full benefits of the architecture.

**Testing each property with a real example:**

Consider an Order Service in a food delivery platform like DoorDash.

| Property | Test Question | Passes? |
| --- | --- | --- |
| Independently deployable | Can the Order Service ship a new feature without the Restaurant Service deploying too? | Yes, if APIs are versioned and backward compatible |
| Owns its own data | Does the Order Service have its own orders database that no other service queries directly? | Yes, other services call the Order API, not the orders table |
| Business capability | Does "order management" represent a real business function? | Yes, creating, tracking, and completing orders is a core capability |
| Single team ownership | Is there one team responsible for the Order Service end-to-end? | Yes, the orders team owns the code, the database, and the on-call rotation |

**Common Mistake:** Many teams build services that share a database. The Payments Service and the Orders Service both read from the same `orders` table. They can be deployed separately, but they are not truly independent because a schema change to the `orders` table breaks both services. This fails the "owns its own data" test.

**Interview Relevance:** When you say "I would use microservices" in an interview, the interviewer may ask "what makes something a microservice?" Being able to state these four properties clearly puts you ahead of most candidates.

---

### **SLIDE 3: What Microservices Do Not Mean**

There are several common misconceptions about microservices that lead to poor architectural decisions, especially in interviews.

**Misconception 1: "Microservices means small services"**

The "micro" in microservices does not refer to lines of code. A service with 50,000 lines of code can be a well-designed microservice if it owns a single business capability end-to-end. A service with 200 lines of code can be a terrible microservice if it cannot function without calling three other services for every request.

**Misconception 2: "Microservices means Docker and Kubernetes"**

Docker and Kubernetes are deployment tools. You can run a monolith in Kubernetes. You can run microservices on bare metal VMs. Containers make microservices easier to operate, but they are not what makes something a microservice.

**Misconception 3: "Microservices means splitting code into separate repos"**

Separate repositories do not create service boundaries. You can have ten repos that are all tightly coupled and must be deployed together. That is a distributed monolith, not microservices. The boundary is about independent deployability and data ownership, not repository structure.

**Misconception 4: "One microservice per database table"**

This is one of the most common mistakes in interviews. If you split your User table into a User Service and your Address table into an Address Service, you have not decomposed by business capability. You have decomposed by database schema, which creates chatty services that cannot do anything useful alone.

**Interview Relevance:** Interviewers test whether you understand what microservices actually are. Mentioning Docker or Kubernetes as a defining trait signals a shallow understanding of the architecture.

---

### **SLIDE 4: The Core Promise of Microservices**

Microservices make three promises. Every architectural benefit flows from these three properties.

**Promise 1: Independent Deployment**

Each service can be built, tested, and deployed without coordinating with other teams. If the Notification Service needs a bug fix, only the notifications team deploys. No release train. No waiting for the Order Service team to finish their sprint.

**Promise 2: Independent Scaling**

Each service scales based on its own load. In an e-commerce platform like Amazon, the Product Catalog Service handles 50x more reads than the Checkout Service. With microservices, you scale catalog reads horizontally without wasting resources on checkout infrastructure that is mostly idle.

**Promise 3: Independent Ownership**

One team owns one service end-to-end: the code, the data, the on-call rotation, the deployment pipeline. Ownership boundaries reduce coordination overhead. A team of 4-8 engineers can move fast because they do not need approval from six other teams to ship.

**The trade-off for each promise:**

- Independent deployment requires backward-compatible APIs between services
- Independent scaling requires infrastructure complexity and more moving parts
- Independent ownership requires accepting data duplication and cross-team communication overhead

**Interview Tip:** When an interviewer asks "why microservices?", do not just say "scalability." State all three promises and then explain which specific promise justifies the choice for the system you are designing. A system with one team and uniform load does not benefit from any of these promises.

**Interview Relevance:** Interviewers want to hear you justify microservices with specific architectural benefits, not repeat buzzwords. Stating the three promises and mapping them to the problem shows maturity.

---

### **SLIDE 5: The Core Cost of Microservices**

Every benefit of microservices comes with a corresponding cost. These are not edge cases. These are guaranteed consequences of distributing a system across a network.

!image.png

Every arrow in this diagram is a network call. Every network call can fail, time out, or return stale data. In a monolith, these would be function calls that take nanoseconds and never fail due to network issues.

**The five costs:**

1. **Network latency** - Every service call adds 1-10ms round trip. A monolith function call takes nanoseconds.
2. **Partial failure** - Payment succeeds but inventory reservation fails. In a monolith, a single transaction guarantees all-or-nothing.
3. **Data inconsistency** - The Order Service and Inventory Service can temporarily disagree about stock levels. A single database is always consistent.
4. **Debugging complexity** - One user request touches 5 services across 5 log streams. In a monolith, you have one process, one log file, one stack trace.
5. **Operational overhead** - 20 services means 20 deployment pipelines, 20 monitoring dashboards, 20 potential points of failure.

**The honest math:**

A monolith making 4 internal function calls to process a checkout has near-zero failure probability from those calls. The same checkout across 4 microservices making network calls, each with 99.9% availability, has a combined availability of 99.9%^4 = 99.6%. That gap of 0.4% means roughly 3.5 additional hours of downtime per year, purely from the architecture choice.

**Interview Relevance:** Interviewers specifically test whether candidates understand the costs of their proposals. Proposing microservices without acknowledging distributed complexity is a red flag at any level.

---

### **SLIDE 5B: Note It Down**

**Five Costs of Microservices (memorize these):**

1. **Network latency** - 1-10ms per hop vs nanoseconds for in-process calls
2. **Partial failure** - One service can fail while others succeed
3. **Data inconsistency** - No shared transaction across services
4. **Debugging complexity** - One request, many services, many log streams
5. **Operational overhead** - Each service needs its own pipeline, monitoring, and on-call

**When to bring these up in an interview:**

Do not wait for the interviewer to challenge your microservices proposal. State the costs proactively and then explain how you will mitigate each one. This is what separates a senior-level answer from a junior-level answer.

**Interview Tip:** A strong pattern is: "I am choosing microservices here because [specific promise matters for this system], and I will handle the distributed complexity by [specific mitigation for each relevant cost]."

**Interview Relevance:** Being able to list these five costs without hesitation demonstrates practical experience with distributed systems, not just textbook knowledge.

---

### **SLIDE 6: Microservices as an Organizational Pattern (Conway's Law)**

Conway's Law states: "Organizations design systems that mirror their own communication structure."

This is not a suggestion. It is an observation that has held true across decades of software engineering. If you have three teams, you will end up with three major services or three major components, regardless of what the architecture diagram says.

**What this means for microservices:**

Service boundaries should follow team boundaries. If the payments team and the inventory team are separate groups with separate managers, their code should live in separate services. Forcing them into one monolith creates constant merge conflicts, shared on-call rotations, and deployment dependencies that slow everyone down.

**Reverse Conway Maneuver:**

Smart organizations do this intentionally. Instead of letting the org chart accidentally dictate architecture, they design the ideal service boundaries first, then structure teams to match. The architecture drives the org chart, not the other way around.

**Real-world example:**

A company like Spotify organizes into "squads" that each own a specific product area (search, playback, recommendations). Each squad owns its own services, data, and deployment pipeline. The architecture mirrors the team structure by design, not by accident. When they need a new capability like podcast recommendations, they form a new squad and that squad builds and owns the new service.

**Interview Relevance:** Mentioning Conway's Law shows the interviewer you understand that microservices are an organizational decision, not just a technical one. This separates senior-level answers from junior-level answers.

---

### **SLIDE 6B: Interview Practice**

**Question:**

You are in a system design interview. The interviewer asks you to design a food delivery platform and you immediately say "I would use a microservices architecture." The interviewer responds: "Why? Convince me."

Take 60 seconds to think about your answer before reading below.

**Weak Answer:**

"Microservices are the industry standard. Companies like Uber and DoorDash use them. They are more scalable and modern than monoliths."

**Why this is weak:** It does not explain why this specific system needs microservices. "Industry standard" and "modern" are not architectural justifications. The interviewer wants to hear reasoning specific to the problem, not an appeal to popularity.

**Strong Answer:**

"A food delivery platform has at least four distinct scaling profiles: restaurant search is read-heavy and spiky during meal times, order processing is write-heavy and needs strong consistency, driver dispatch is real-time and location-intensive, and payment processing has low throughput but high reliability requirements. These also map to different team specializations. Microservices let us scale each independently, deploy the dispatch system without risking the payment flow, and let specialized teams own their domain. That said, if this is an early-stage startup with one engineering team, I would start with a modular monolith and extract services as the team and load grow."

**What makes this strong:** The candidate identifies specific scaling profiles, maps them to independent deployment needs, acknowledges team structure, and shows maturity by not defaulting to microservices unconditionally. The closing sentence about starting with a modular monolith demonstrates real-world judgment.

---

### **SLIDE 7: Interview Framing**

In a system design interview, microservices should never be your default starting point. They should be a justified design choice that you arrive at through reasoning.

**The decision flow:**

!image.png

**What interviewers look for:**

- You do not jump to microservices by default
- You ask about team size, product maturity, and scaling requirements before choosing
- You can articulate which specific promise of microservices justifies the complexity
- You acknowledge the costs and explain how you will handle them
- You know that a modular monolith is a valid and often better starting point

**Common Mistake:** Starting every system design answer with "I would use microservices with an API gateway, message queue, and load balancer." This sounds rehearsed and shows no reasoning. The interviewer wants to see you think through the trade-offs, not recite an architecture template.

**Interview Relevance:** The way you frame the architecture choice in the first 2-3 minutes of an interview sets the tone. A thoughtful framing builds credibility. A default "microservices" answer without justification puts you on the defensive for the rest of the interview.

---

### **SLIDE 8: Section Summary and Interview Tips**

**Section 1 Summary:**

| Concept | Key Takeaway |
| --- | --- |
| Microservice definition | Independently deployable, owns its data, aligned to business capability, owned by one team |
| What microservices are not | Not small services, not Docker, not separate repos, not one service per table |
| Core promise | Independent deployment, independent scaling, independent ownership |
| Core cost | Network latency, partial failure, data inconsistency, debugging complexity, operational overhead |
| Conway's Law | Team structure dictates system architecture. Design teams intentionally. |
| Interview framing | Microservices are a design choice with trade-offs, not the default answer |

**Common Interview Mistakes:**

| Mistake | Why It Hurts You | Stronger Approach |
| --- | --- | --- |
| "I would use microservices because they are scalable" | Does not explain what specifically needs independent scaling | "The search service handles 10x more traffic than checkout, so independent scaling matters here" |
| "Microservices are the modern best practice" | Sounds like repeating a blog post, not reasoning | "Microservices make sense here because we have four separate teams and three distinct scaling profiles" |
| Jumping straight to microservices without considering alternatives | Shows you only know one architecture | "For an early-stage product with one team, I would start with a modular monolith and extract services as the domain matures" |
| Defining microservices as "small services" | Misses the actual properties that matter | "A microservice is independently deployable, owns its data, and is aligned to a business capability" |

**Key Numbers to Remember:**

| Metric | Value |
| --- | --- |
| Network call latency per hop | 1-10ms (vs nanoseconds for in-process calls) |
| Availability math | 4 services at 99.9% each = 99.6% combined |
| Downtime at 99.9% | 8.7 hours/year |
| Downtime at 99.6% | 35 hours/year |
| Typical team size per service | 4-8 engineers |

**How This Connects to Other Sections:**

Section 2 (Monolith to Microservices) builds directly on this mental model. You will see how monoliths, modular monoliths, and distributed monoliths compare against the four properties from Slide 2. Section 3 (Service Decomposition) uses the "aligned to a business capability" property as the foundation for finding service boundaries. Section 5 (Data Ownership) expands on the "owns its own data" property and explores what happens when services need each other's data. Section 6 (Sagas and Consistency) addresses the "partial failure" cost from Slide 5 and teaches you how to handle transactions that span multiple services.


---
# Section 2: Monolith to Microservices

## SLIDE 9: Section Intro

### Section 2: Monolith to Microservices

The biggest mistake engineers make with microservices is treating them as the default architecture. In reality, microservices sit at one end of a spectrum, and understanding the full spectrum is what separates thoughtful design from cargo-cult architecture.

This section walks through the four points on that spectrum: traditional monolith, modular monolith, distributed monolith (the anti-pattern), and true microservices. By the end, you will know when each architecture is the right choice, how to migrate between them, and how to avoid the traps that turn a well-intentioned microservices adoption into a distributed mess.

#### What We Will Cover:

- The traditional monolith and why it works longer than most people think
- The modular monolith as the most underused architecture in the industry
- The distributed monolith anti-pattern and how to detect it
- True microservices and how they differ from everything else
- A comparison framework for choosing between architectures
- The migration path from monolith to microservices
- The Strangler Fig pattern for incremental migration
- Common anti-patterns that derail decomposition

#### Where This Appears in Real Systems:

Every major platform you use today started as a monolith. A platform like Shopify scaled to billions in GMV on a Rails monolith. A platform like GitHub ran as a single Rails application for years before selectively extracting services. The question is never "should we use microservices?" The question is "when is the right time to move, and how do we do it without breaking everything?"

---

## SLIDE 10: The Traditional Monolith

A monolith is a single deployable unit where all features, business logic, and data access live in one codebase and deploy as one artifact.

```
┌─────────────────────────────────────────────────────────────┐
│                       MONOLITH                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   Order     │  │  Inventory  │  │    User     │       │
│  │   Module    │  │   Module    │  │   Module    │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         │                │                │               │
│         └────────────────┼────────────────┘               │
│                          │                                │
│                   ┌──────▼──────┐                         │
│                   │  Single     │                         │
│                   │  Database   │                         │
│                   └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

All four modules deploy together, scale together, and share one database. A function call from the Order Module to the Inventory Module is an in-process call. It takes nanoseconds. It never fails because of a network timeout. It participates in a single database transaction that guarantees consistency.

**This is not a weakness. This is an enormous advantage that you give up the moment you distribute your system across a network.**

#### What deployment looks like:

One team runs `git push`, the CI pipeline builds one artifact, tests run against one application, and one deployment goes to production. If something breaks, one rollback fixes everything. There is no question of "which service caused the outage?" because there is only one service.

> **Interview Relevance:** Interviewers respect candidates who can explain monolith strengths with the same depth they explain microservices. It shows you choose architectures based on reasoning, not trends.

---

## SLIDE 11: Monolith Strengths and When It Works

Monoliths work well for far longer than most engineers assume. Here is why.

#### Simplicity of operations:
One deployment pipeline. One monitoring dashboard. One on-call rotation. One log stream to search when something breaks at 3am. For a team of 5-15 engineers, this operational simplicity is not a small benefit. It is a massive productivity advantage.

#### Full ACID transactions:
When a user places an order, you can debit inventory, create the order record, and charge the payment in a single database transaction. If any step fails, everything rolls back automatically. In microservices, achieving this same guarantee requires sagas, compensating transactions, and idempotency keys, which you will learn about in Section 6.

#### Fast iteration speed:
Change the order logic, the payment logic, and the notification logic in one pull request. Deploy it all at once. See the results immediately. No need to coordinate deployments across teams, no backward-compatibility concerns between service versions, no integration testing across network boundaries.

#### Real-world proof:
A platform like Shopify processed over $200 billion in GMV in 2023 running primarily on a Rails monolith. They have extracted some services over time, but the core remains monolithic. A platform like Stack Overflow serves millions of developers from a monolith running on remarkably few servers.

> **Interview Tip:** If an interviewer asks you to design a system for a 5-person startup, proposing microservices is the wrong answer. The right answer is a well-structured monolith that can be modularized later. Say this explicitly and the interviewer will know you have real-world judgment.

> **Interview Relevance:** "When would you not use microservices?" is a common interview question. Having concrete examples of successful monoliths at scale makes your answer credible.

---

## SLIDE 12: The Modular Monolith

A modular monolith is a monolith with enforced internal boundaries. It deploys as a single unit but is organized into distinct modules that communicate through well-defined interfaces.

```
┌─────────────────────────────────────────────────────────────┐
│                   MODULAR MONOLITH                         │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   Order     │  │  Inventory  │  │    User     │       │
│  │   Module    │  │   Module    │  │   Module    │       │
│  │             │  │             │  │             │       │
│  │  ┌────────┐ │  │  ┌────────┐ │  │  ┌────────┐ │       │
│  │  │ Order  │ │  │  │Invent. │ │  │  │ User   │ │       │
│  │  │ Table  │ │  │  │ Table  │ │  │  │ Table  │ │       │
│  │  └────────┘ │  │  └────────┘ │  │  └────────┘ │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
│         ▲                ▲                ▲               │
│         └────────────────┼────────────────┘               │
│                          │                                │
│                   ┌──────▼──────┐                         │
│                   │  Single     │                         │
│                   │  Database   │                         │
│                   └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

Notice the difference from the traditional monolith. Each module still lives in the same database, but each module only accesses its own tables. The Order Module cannot directly query the User tables. It must call the User Module's public interface to get user data.

#### Why this matters:

The modular monolith gives you the boundary discipline of microservices without the operational cost of distribution. You practice data ownership, define clean interfaces, and discover where the real domain boundaries are, all while keeping the simplicity of a single deployment.

When you eventually extract a module into a microservice, the interface already exists. You replace the in-process function call with a network call to the new service. The contract stays the same.

> **Interview Relevance:** Mentioning the modular monolith as a deliberate architectural choice shows the interviewer you understand the spectrum of options between monolith and microservices, not just the two extremes.

---

## SLIDE 13: Modular Monolith, Enforcing Boundaries in Code

The difference between a modular monolith and a messy monolith is enforcement. Without enforcement, modules slowly reach into each other's internals, and the boundaries erode within months.

#### How boundaries are enforced:

```python
class OrderModule:
    def __init__(self, user_module, inventory_module):
        self.user_module = user_module
        self.inventory_module = inventory_module

    def place_order(self, user_id, product_id, quantity):
        user = self.user_module.get_user(user_id)
        available = self.inventory_module.check_availability(product_id, quantity)

        if not available:
            raise InsufficientInventoryError(product_id, quantity)

        order = Order(user_id=user_id, product_id=product_id, quantity=quantity)
        self.order_repository.save(order)
        return order
```

The Order Module calls `user_module.get_user()` and `inventory_module.check_availability()`. It does not import the User model directly. It does not write SQL queries against the inventory tables. It goes through the public interface of each module.

#### Enforcement mechanisms:

- **Package-level access control** - In Java, use separate packages with package-private classes. In Python, use explicit `__init__.py` exports.
- **Architecture tests** - Tools like ArchUnit (Java) or custom linting rules that fail the build if a module imports from another module's internals.
- **Code review discipline** - Reviewers reject any PR that crosses module boundaries without going through the public interface.

**The critical rule:** If the Order Module can reach into the inventory database and run `SELECT * FROM inventory WHERE product_id = ?` directly, you do not have a modular monolith. You have a monolith with folders.

> **Interview Relevance:** Interviewers testing for senior-level thinking may ask how you would prepare a monolith for future microservices extraction. The modular monolith with enforced boundaries is the textbook answer.

---

## SLIDE 14: The Distributed Monolith Anti-Pattern

A distributed monolith is the worst of both worlds. You pay the full cost of a distributed system but get none of the benefits.

#### How it happens:

A team splits their monolith into 5 services, but all 5 services still share the same database. Deploying Service A requires deploying Service B at the same time because they depend on each other's internal data formats. Every feature requires changes across 3 or 4 services simultaneously. There is a "release train" where all services must be deployed together on Thursday at 2pm.

You now have all the network latency, all the partial failure scenarios, all the debugging complexity of microservices. But you have none of the independent deployment, none of the independent scaling, none of the independent ownership.

#### How to detect a distributed monolith:

- **Shared database** - Multiple services read from and write to the same tables
- **Coordinated deployments** - "We need to deploy services A, B, and C together"
- **Circular dependencies** - Service A calls B, B calls C, C calls A
- **Chatty communication** - 10+ network calls between services for a single user request
- **Cross-service ownership** - One team owns multiple services, or one feature requires changes in 4 services
- **Shared data models** - Services import the same ORM models or share protobuf definitions with business logic embedded

**The key test:** Ask this question about each service: "Can this service be deployed independently, right now, without any other service also deploying?" If the answer is no for most services, you have a distributed monolith.

> **Interview Relevance:** If an interviewer suspects your design is a distributed monolith (shared database, coordinated deploys, circular dependencies), they will push back hard. Being able to identify and name this anti-pattern is a senior-level skill.

---

## SLIDE 14B: Interview Practice

### Question:

An interviewer shows you this architecture and asks: "What is wrong with this design?"

The design has 4 services: User Service, Order Service, Payment Service, and Notification Service. All four services connect to a single shared PostgreSQL database. The Order Service calls the Payment Service synchronously for every order. The Payment Service calls back to the Order Service to update order status. The Notification Service queries the orders table directly to find orders that need notifications.

*Take 60 seconds to identify the problems before reading below.*

---

### Weak Answer:

> "It needs more services. I would also add an Inventory Service and a Shipping Service."

**Why this is weak:** Adding more services makes the problem worse, not better. The candidate missed the fundamental issues and jumped to adding complexity.

---

### Strong Answer:

> "This is a distributed monolith. There are three specific problems. First, all four services share a single database, so a schema migration to the orders table could break the Order Service, Payment Service, and Notification Service simultaneously. There is no true data ownership. Second, the Order Service and Payment Service have a circular dependency. Order calls Payment, Payment calls back to Order. This creates tight coupling that prevents independent deployment. Third, the Notification Service directly queries the orders table instead of consuming events or calling an API, which means it is coupled to the Order Service's internal schema. To fix this, each service needs its own database, the circular dependency needs to be broken with asynchronous events, and the Notification Service should consume order events from a message queue instead of querying the orders table."

**What makes this strong:** The candidate names the anti-pattern, identifies three specific structural problems, explains why each is harmful, and proposes concrete fixes.

---

## SLIDE 15: True Microservices Architecture

A true microservices architecture looks fundamentally different from a distributed monolith. The key difference is not the number of services. It is the independence between them.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Order         │     │   Payment       │     │   Notification  │
│   Service       │     │   Service       │     │   Service       │
│                 │     │                 │     │                 │
│  ┌───────────┐  │     │  ┌───────────┐  │     │  ┌───────────┐  │
│  │  Order    │  │     │  │  Payment  │  │     │  │Notific.   │  │
│  │  Database │  │     │  │  Database │  │     │  │ Database  │  │
│  └───────────┘  │     │  └───────────┘  │     │  └───────────┘  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                       │                       │
         │    ┌──────────────┐   │    ┌──────────────┐   │
         └───►│  Message     │◄──┘    │  API Gateway │◄──┘
              │  Queue       │        └──────────────┘
              └──────────────┘
```

What makes this different from the distributed monolith:

- **Each service has its own database.** The Order Service cannot query the User DB directly. It calls the User Service API or holds a cached copy of the user data it needs.
- **Communication between services is primarily asynchronous.** The Order Service publishes an "order created" event. The Payment Service and Notification Service consume that event independently.
- **No circular dependencies.** Data flows in one direction for any given workflow.
- **Each service can be deployed independently.** A new version of the Payment Service does not require the Order Service to deploy at the same time.

#### The independence test (revisiting Section 1):

Apply the four properties from Slide 2 to each service. If every service passes all four tests (independently deployable, owns its data, aligned to a business capability, owned by one team), you have true microservices.

> **Interview Tip:** When you draw a microservices diagram in an interview, always show separate databases for each service. This immediately signals to the interviewer that you understand data ownership. Drawing multiple services pointing to one database is one of the fastest ways to lose credibility.

> **Interview Relevance:** The visual difference between a true microservices diagram and a distributed monolith diagram is something interviewers look for in the first minute of your whiteboard drawing.

---

## SLIDE 16: Monolith vs Modular Monolith vs Microservices

| Dimension | Monolith | Modular Monolith | Microservices |
|-----------|----------|------------------|---------------|
| **Deployment** | Single artifact | Single artifact | Independent per service |
| **Internal boundaries** | Weak or none | Strong, enforced in code | Network boundaries |
| **Database** | Single shared | Single, but module-scoped access | Separate per service |
| **Inter-module calls** | Direct function calls | Function calls through interfaces | Network calls (REST, gRPC, events) |
| **Team scaling** | 1-15 engineers | 5-40 engineers | 30+ engineers, multiple teams |
| **Failure isolation** | One bug can crash everything | One bug can crash everything | Failure contained to one service |
| **Consistency** | Full ACID transactions | Full ACID transactions | Eventual consistency, sagas |
| **Operational complexity** | Low | Low | High |
| **Best starting point** | Early stage, small team | Growing team, maturing domain | Large org, proven boundaries |

Note that the distributed monolith is intentionally excluded from this table. It is not a valid architectural choice. It is an anti-pattern that happens when microservices are adopted without the discipline of data ownership and independent deployment.

> **Interview Relevance:** This comparison framework is one of the highest-value tools in the entire course. Interviewers expect you to reason about when each architecture is appropriate, not to default to microservices every time.

---

## SLIDE 17: When Monolith Beats Microservices

There are specific, concrete situations where a monolith is the better architecture. These are not compromises. These are the correct engineering decision.

### Situation 1: Small team, early product

You have 3-10 engineers. The product is pre-product-market-fit. Requirements change weekly. In this environment, the ability to change anything and deploy everything in minutes is worth more than independent scaling. The coordination overhead of microservices would slow this team down, not speed it up.

### Situation 2: Unclear domain boundaries

You are building a new product and you do not yet know where the real boundaries are. Is "pricing" part of the "catalog" domain or the "checkout" domain? Should "user preferences" live with "user profiles" or with "recommendations"? Getting these boundaries wrong in a microservices architecture is expensive to fix. Getting them wrong in a monolith is a refactor.

### Situation 3: Strong consistency requirements everywhere

If nearly every operation in your system requires ACID transactions across multiple data entities, microservices make this dramatically harder. A banking ledger system where every transaction must be immediately consistent across accounts, balances, and audit logs may be simpler and safer as a monolith with a single relational database.

### Situation 4: Uniform scaling profile

If every part of your system has the same load pattern and scales the same way, independent scaling per service adds infrastructure complexity without benefit. You can scale a monolith horizontally by running multiple instances behind a load balancer.

> **Interview Relevance:** Answering "when would you not use microservices?" with specific scenarios and concrete reasoning is one of the strongest signals of architectural maturity an interviewer can see.

---

## SLIDE 18: When Microservices Are Justified

Microservices earn their complexity when specific organizational and technical conditions are met. These conditions, not trends or blog posts, should drive the decision.

### Condition 1: Multiple teams need to ship independently

You have 30+ engineers across 4-8 teams. Each team has its own roadmap and release cadence. Waiting for a shared release train slows everyone down. Independent deployment becomes a genuine productivity multiplier.

### Condition 2: Different parts of the system have different scaling needs

In a platform like YouTube, the video upload pipeline handles 500 hours of video per minute but can tolerate latency. The video playback service handles millions of concurrent streams and needs sub-100ms latency. The recommendation engine is compute-intensive and bursty. Scaling these behind one monolith wastes resources. Separate services let you allocate infrastructure precisely.

### Condition 3: Domain boundaries are well understood and stable

The team has been working on this product for 1-2 years. The domain model is mature. Boundaries between "orders," "payments," "inventory," and "shipping" are clear and rarely change. Extracting these into services carries low risk of getting the boundaries wrong.

### Condition 4: Fault isolation is critical

A bug in the recommendation engine should not take down the checkout flow. A surge in notification sending should not slow down order processing. When failures in one domain must not cascade to unrelated domains, service boundaries provide natural fault isolation.

> **Interview Tip:** In an interview, state these conditions explicitly before proposing microservices. "I am choosing microservices for this design because we have multiple teams, different scaling profiles for search vs checkout, and well-understood domain boundaries." This shows the interviewer you are reasoning from first principles, not defaults.

> **Interview Relevance:** The gap between a junior and senior answer is justification. Junior candidates say "microservices." Senior candidates say "microservices because these specific conditions are met."

---

## SLIDE 18B: Interview Practice

### Question:

You are designing a system for a startup that just raised its Series A. They have 8 engineers, one product, and about 10,000 daily active users. The CEO asks you to design the backend architecture. The CTO mentions they want to "start with microservices so we do not have to rewrite later." What do you recommend?

*Take 60 seconds to form your answer before reading below.*

---

### Weak Answer:

> "The CTO is right. Starting with microservices avoids technical debt and prevents a painful rewrite later. I would create separate services for users, orders, payments, and notifications."

**Why this is weak:** It agrees with the premise without examining it. With 8 engineers and 10,000 DAU, microservices add operational overhead (4+ deployment pipelines, distributed debugging, eventual consistency) that the team is not staffed to handle. The "avoid rewrite" argument ignores that wrong service boundaries chosen now will require a more painful rewrite than refactoring a monolith.

---

### Strong Answer:

> "I would recommend starting with a modular monolith. With 8 engineers, the team does not have the capacity to operate multiple independent services, maintain separate deployment pipelines, and handle distributed debugging. More importantly, at 10,000 DAU with a product still finding its market, the domain boundaries are not yet stable. If we split into microservices now, we are likely to draw the boundaries in the wrong places, and re-drawing service boundaries is far more expensive than refactoring modules within a monolith. I would structure the monolith with clear internal modules for users, orders, payments, and notifications, each with enforced interfaces and module-scoped database access. When the team grows to 20-30 engineers and the domain stabilizes, we can extract the modules that need independent scaling or deployment into services. The modular monolith makes that future extraction straightforward because the interfaces already exist."

**What makes this strong:** The candidate respectfully disagrees with the CTO using concrete reasoning (team size, operational capacity, domain maturity). They propose a specific alternative (modular monolith) and explain how it sets up a future migration path. They demonstrate that "avoiding a rewrite" is better achieved by modular design now than by premature distribution.

---

## SLIDE 19: The Migration Path

Moving from monolith to microservices is not a binary switch. It is a staged migration that should happen incrementally over months or years.

### The three stages:

#### Stage 1: Monolith to Modular Monolith

Identify the major business domains in your codebase. Draw boundaries around them. Enforce those boundaries with package structure, interface definitions, and architecture tests. Each module gets its own set of database tables, and no module directly accesses another module's tables. This stage involves zero infrastructure changes. It is purely a code organization exercise.

#### Stage 2: Modular Monolith to First Extraction

Pick the module with the strongest case for extraction. This is usually the module with the most distinct scaling profile or the one owned by a team that is most bottlenecked by shared deployments. Extract it into a separate service with its own database. Replace the in-process function calls with API calls or event consumption. Keep everything else in the monolith.

#### Stage 3: Incremental Extraction

Extract additional modules one at a time, each time following the same pattern: separate database, separate deployment, replace in-process calls with network calls. Stop extracting when the remaining modules do not have a strong case for independence. Not everything needs to be a microservice.

**Critical principle:** Extract the module with the highest pain, not the easiest one. The easiest module to extract teaches you the least. The one causing deployment bottlenecks or scaling problems teaches you the most and delivers the most value.

> **Interview Relevance:** Describing a staged migration path shows the interviewer you have practical experience with system evolution, not just greenfield design.

---

## SLIDE 20: The Strangler Fig Pattern

The Strangler Fig pattern is the standard technique for incrementally migrating a monolith to microservices without a big-bang rewrite. It is named after the strangler fig tree, which grows around an existing tree and gradually replaces it.

### How it works:

```
                    ┌─────────────────┐
                    │  API Gateway /  │
                    │  Routing Proxy  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────────┐
              │              │                  │
              ▼              ▼                  ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │   Monolith   │ │   Payment    │ │   Order      │
     │  (Existing)  │ │   Service    │ │   Service    │
     │              │ │   (New)      │ │   (New)      │
     │ /users/*     │ │ /payments/*  │ │ /orders/*    │
     │ /orders/*    │ └──────────────┘ └──────────────┘
     │ /inventory/* │
     └──────────────┘
```

A routing proxy sits in front of the monolith. Initially, all traffic goes to the monolith. When you extract the Payment Service, you update the proxy to route `/payments/*` requests to the new service. The monolith still handles users, orders, and inventory. Over time, more routes shift to new services until the monolith handles nothing and can be retired.

#### Why this works:

- Zero downtime during migration
- Each extraction is independently testable and reversible
- The monolith continues working for everything that has not been extracted yet
- Risk is contained to one service at a time

> **Interview Relevance:** When an interviewer asks "how would you migrate from a monolith to microservices?", the Strangler Fig pattern is the expected answer. Proposing a big-bang rewrite is a red flag.

---

## SLIDE 20B: Strangler Fig, Worked Example

### Scenario:

An e-commerce platform like Shopify has a Rails monolith handling 50,000 requests per second. The payments module processes $500M in monthly transactions. The team wants to extract payments into a separate service.

#### Step 1: Add the routing proxy

Place an API gateway (or reverse proxy like Nginx) in front of the monolith. Configure it to forward all traffic to the monolith unchanged. Verify that adding the proxy does not affect latency or reliability. This step changes zero behavior.

#### Step 2: Build the new Payment Service

Build the Payment Service as a standalone application with its own database. Implement the same payment API that the monolith currently exposes. Deploy it alongside the monolith but do not route any traffic to it yet.

#### Step 3: Shadow traffic

Route a copy of payment requests to both the monolith and the new service. The monolith's response goes to the client. The new service's response is logged and compared. This catches any discrepancies before real traffic moves over.

#### Step 4: Canary rollout

Route 1% of payment traffic to the new service. Monitor error rates, latency, and transaction success rates. If metrics are healthy, increase to 5%, then 25%, then 50%, then 100%.

#### Step 5: Decommission the old payment code

Once 100% of payment traffic runs through the new service, remove the payment module from the monolith. Remove the payment tables from the monolith database. The monolith is now smaller, and the Payment Service is fully independent.

**Timeline:** This process typically takes 2-4 months for a critical service like payments. Rushing it is not worth the risk.

> **Interview Tip:** Walking through these specific steps in an interview demonstrates that you have done this before, or at least understand it at a practical level. Most candidates stop at "use the Strangler Fig pattern" without explaining the actual mechanics.

> **Interview Relevance:** The level of detail you provide when describing a migration separates "I read about this" from "I have done this."

---

## SLIDE 21: Common Decomposition Anti-Patterns

These are the mistakes that turn a well-intentioned microservices migration into a distributed monolith. Each one violates the principles from Section 1.

### Anti-pattern 1: Splitting by technical layer

Creating a "Frontend Service," "Business Logic Service," and "Database Service." This creates three tightly coupled services that must change together for every feature. Service boundaries should follow business capabilities (orders, payments, inventory), not technical layers.

### Anti-pattern 2: One service per database table

Creating a User Service, Address Service, Phone Number Service, and Email Service because each has its own table. These are not independent business capabilities. They are fragments of a single "User Profile" capability that should be one service.

### Anti-pattern 3: Premature extraction

Extracting services before the domain is understood. The team splits "catalog" and "pricing" into separate services, then discovers that every pricing change requires a catalog change. They now have two services that must always deploy together, which is a distributed monolith.

### Anti-pattern 4: The "god service"

One service accumulates business logic that should belong elsewhere. The Order Service starts handling payment validation, inventory checks, shipping calculations, and notification formatting. It becomes a monolith with a microservice label.

### Anti-pattern 5: Shared database backdoor

Services officially communicate through APIs, but three of them still read from a shared "reference data" database table. Any schema change to that table breaks all three services. This is a hidden coupling that defeats the purpose of separate services.

> **Interview Relevance:** Being able to name these anti-patterns and explain why they are harmful shows the interviewer you understand the principles behind microservices, not just the structure.

---

## SLIDE 21B: Note It Down

### Five decomposition anti-patterns to memorize:

1. **Splitting by technical layer** - Services should follow business capabilities, not frontend/backend/database layers
2. **One service per table** - A database table is not a business capability. Group related data under one service.
3. **Premature extraction** - Extract services only when domain boundaries are stable and the pain of coupling is real
4. **God service** - If one service does everything, you have a monolith with extra network calls
5. **Shared database backdoor** - If services share a database table, they are not truly independent

### The test to apply:

For each service in your design, ask: "Can this service be developed, deployed, and operated by one team without coordinating with other teams?" If the answer is no, you have one of these anti-patterns.

> **Interview Relevance:** Interviewers may describe a flawed architecture and ask you to identify problems. Being able to name these anti-patterns by name and explain the fix is a strong signal of experience.

---

## SLIDE 22: Section Summary and Interview Tips

### Section 2 Summary:

| Concept | Key Takeaway |
|---------|--------------|
| **Traditional monolith** | Single deployable, shared database, simple operations, works well for small teams |
| **Modular monolith** | Enforced internal boundaries, single deployment, best stepping stone to microservices |
| **Distributed monolith** | Anti-pattern: distributed system costs with no independence benefits |
| **True microservices** | Independent deployment, private databases, business-aligned, team-owned |
| **When to stay monolith** | Small team, unclear domain, uniform scaling, strong consistency needs |
| **When to go microservices** | Multiple teams, different scaling profiles, stable domain, fault isolation needs |
| **Migration path** | Monolith → modular monolith → incremental extraction |
| **Strangler Fig pattern** | Incremental migration through routing proxy, shadow traffic, canary rollout |
| **Decomposition anti-patterns** | Technical layers, one-per-table, premature extraction, god service, shared DB |

### Common Interview Mistakes:

| Mistake | Why It Hurts You | Stronger Approach |
|---------|------------------|-------------------|
| "Always start with microservices to avoid rewriting later" | Shows you do not understand that wrong boundaries are harder to fix than monolith refactoring | "Start with a modular monolith. Extract services when domain boundaries stabilize and team size justifies the overhead." |
| "Our monolith cannot scale" | Monoliths scale horizontally. The bottleneck is usually team coordination, not performance. | "The monolith scales fine technically. The problem is 6 teams blocking each other on deployments." |
| Proposing a big-bang rewrite | Almost always fails. High risk, long timeline, business stops getting features. | "I would use the Strangler Fig pattern to extract one service at a time over 6-12 months." |
| Drawing multiple services pointing to one database | Immediately signals distributed monolith to the interviewer | "Each service has its own database. Cross-service data access goes through APIs or events." |

### Key Numbers to Remember:

| Metric | Value |
|--------|-------|
| Monolith team ceiling | ~15 engineers before coordination becomes painful |
| Modular monolith ceiling | ~40 engineers with strong module discipline |
| Microservices starting point | 30+ engineers, multiple teams with independent roadmaps |
| Strangler Fig migration timeline | 2-4 months per service extraction for critical services |
| Shadow traffic comparison phase | Typically 1-2 weeks before canary rollout begins |

### How This Connects to Other Sections:

**Section 3 (Service Decomposition)** takes the "business capability alignment" principle from this section and turns it into a practical methodology using bounded contexts, aggregates, and domain-driven design. The anti-patterns from Slide 21 will reappear there as concrete examples of what happens when decomposition goes wrong.

**Section 4 (Communication Patterns)** addresses how services talk to each other once they are extracted, directly building on the "replace in-process calls with network calls" step from the migration path.

**Section 5 (Data Ownership)** expands on the "each service owns its database" principle and tackles the hard question: what happens when Service A needs data that Service B owns?

---
# Section 3: Service Decomposition and Boundaries

## SLIDE 23: Section Intro

### Section 3: Service Decomposition and Boundaries

If you only master one skill from this entire course, it should be service decomposition. This is the single most important skill in microservices architecture, both in production and in interviews.

Getting boundaries right means services that are independent, cohesive, and easy to evolve. Getting boundaries wrong means a distributed monolith that is harder to change than the original monolith ever was. Section 2 taught you when to use microservices. This section teaches you how to find the right boundaries when you do.

#### What We Will Cover:

1. Business capability decomposition and why it is the starting point
2. Domain-driven design fundamentals: bounded contexts and aggregates
3. Cohesion and coupling as the guiding principles for every boundary decision
4. Team ownership and service granularity
5. Three decomposition approaches: by capability, by workflow, by read/write pattern
6. Service boundary anti-patterns and how to avoid them
7. Boundary discovery techniques including event storming
8. Real-world decomposition examples for e-commerce, ride-sharing, streaming, and banking

#### Where This Appears in Real Systems:

Every microservices architecture that works well has boundaries drawn around business capabilities. Every microservices architecture that struggles has boundaries drawn around database tables, technical layers, or arbitrary code splits. The difference between these two outcomes is the skill taught in this section.

---

## SLIDE 24: Business Capability Decomposition

The most reliable way to find service boundaries is to decompose by business capability. A business capability is something the organization does to generate value, independent of how it is implemented in code.

#### Examples of business capabilities in an e-commerce platform like Amazon:

- **Catalog Management** - Maintaining product listings, descriptions, images, categories
- **Inventory Management** - Tracking stock levels, reserving inventory, restocking
- **Order Management** - Creating, tracking, and completing customer orders
- **Payment Processing** - Authorizing charges, processing refunds, managing payment methods
- **Shipping and Fulfillment** - Calculating shipping options, generating labels, tracking deliveries
- **Customer Support** - Handling returns, resolving disputes, managing tickets

Each of these capabilities can be owned by a separate team, has its own data, and changes for its own business reasons. When the marketing team wants to add video to product listings, only the Catalog Service changes. When the finance team wants to support a new payment provider, only the Payment Service changes.

**The key test:** If you change one capability, does it force a change in another capability? If yes, they might belong together. If no, they are good candidates for separate services.

**Why this beats technical decomposition:** A "Database Service" or "Validation Service" does not map to a business capability. Every feature touches the database. Every feature needs validation. Technical services become bottlenecks that every team depends on, which is the opposite of independence.

> **Interview Relevance:** When an interviewer asks "how would you split this system into services?", starting with business capabilities is the expected approach. Starting with technical layers is a red flag.

---

## SLIDE 25: Domain-Driven Design Fundamentals

Domain-driven design (DDD) provides the vocabulary and methodology for finding service boundaries. You do not need to adopt all of DDD to benefit from it. Three concepts are directly relevant to microservices decomposition.

#### Concept 1: Ubiquitous Language

Every domain has its own vocabulary. The words used by the business should be the same words used in the code. If the business calls it an "order," the code should have an Order class, not a "TransactionRequest." If the business distinguishes between "shipment" and "delivery," the code should too.

Why this matters for microservices: when two teams use different words for the same concept, or the same word for different concepts, that is a signal that they operate in different domains and may need different services.

#### Concept 2: Bounded Context

A bounded context is a boundary within which a particular domain model is defined and consistent. The same word can mean different things in different bounded contexts. We will explore this in depth on the next slide.

#### Concept 3: Aggregates

An aggregate is a cluster of related entities that are treated as a single unit for data changes. The aggregate defines the consistency boundary. We will explore this in Slide 27.

**The connection to microservices:** A bounded context often maps to a microservice. An aggregate often maps to the primary entity that a microservice manages. These are not rigid rules, but they are strong starting heuristics.

> **Interview Relevance:** Mentioning bounded contexts and aggregates in an interview signals that you approach decomposition methodically, not by gut feeling.

---

## SLIDE 26: Bounded Contexts

A bounded context is the most important concept in DDD for microservices. It defines a boundary within which a term has one specific, unambiguous meaning.

**The core insight:** The same word means different things in different parts of the business. "Product" in the catalog domain means a listing with a title, description, images, and categories. "Product" in the inventory domain means a SKU with a stock count, warehouse location, and reorder threshold. "Product" in the shipping domain means a physical item with weight, dimensions, and fragility classification.

If you try to build one "Product" model that satisfies all three domains, you get a bloated, tightly coupled entity that every team fights over. If you let each domain define "Product" in its own way, each service stays focused and independent.

**The rule:** Each bounded context has its own model, its own database representation, and its own definition of shared terms. When data needs to cross context boundaries, it is translated at the boundary through APIs or events, not shared through a common database model.

**Why bounded contexts map to microservices:** Each bounded context represents a self-contained domain model with clear boundaries. That is exactly what a microservice should be: a self-contained service with clear boundaries and its own data.

> **Interview Relevance:** If you can explain bounded contexts with a concrete example of the same word meaning different things in different services, you demonstrate a level of design thinking most candidates do not show.

---

## SLIDE 26B: Bounded Contexts, Worked Example

**Scenario:** An e-commerce platform like Amazon has four teams: Catalog, Orders, Shipping, and Payments. All four teams work with the concept of a "Customer." But "Customer" means something different to each team.

| Bounded Context | What "Customer" Means | Key Attributes |
|-----------------|----------------------|----------------|
| **Catalog** | A browser who views and searches products | Browsing history, wishlist, preferences, recommended categories |
| **Orders** | A buyer who places and tracks orders | Shipping address, order history, saved payment methods |
| **Shipping** | A recipient who receives packages | Delivery address, delivery instructions, signature requirements |
| **Payments** | A financial entity who pays for goods | Billing address, credit card tokens, fraud risk score, payment history |

If you build one Customer Service that holds all of these attributes, every team depends on it. A change to add "delivery instructions" for the Shipping team risks breaking the Payments team's fraud scoring logic. Deployments require coordination across all four teams.

**The better design:** Each bounded context owns its own representation of the customer. The Catalog context stores browsing preferences. The Orders context stores order history. The Shipping context stores delivery details. The Payments context stores financial information. When the Orders context needs the customer's delivery address, it calls the Shipping context's API or receives it as part of an event.

**The trade-off:** Yes, this means some customer data is duplicated across services. The customer's name might exist in all four databases. This duplication is the price of independence, and it is almost always worth paying. Section 5 (Data Ownership) will explore this trade-off in depth.

> **Interview Relevance:** Walking through this specific example in an interview, showing the same entity meaning different things in different contexts, is one of the strongest demonstrations of microservices design thinking.

---

## SLIDE 27: Aggregates and Consistency Boundaries

An aggregate is a cluster of related domain objects that must be changed together to maintain consistency. The aggregate is the unit of transaction within a service.

**Why aggregates matter for microservices:** The aggregate defines where your consistency boundary lives. Everything inside an aggregate is guaranteed to be consistent (ACID transaction within one database). Everything outside the aggregate is eventually consistent (across services, handled by events or sagas).

**Example: The Order Aggregate**

An Order is not just a single row in a database. It is a cluster of related objects:

- The Order itself (order ID, status, creation time)
- Order Line Items (product, quantity, price at time of purchase)
- Order Totals (subtotal, tax, shipping cost, total)

These objects must be consistent with each other at all times. You cannot have an Order Total that does not match the sum of the Line Items. You cannot have a Line Item that references a non-existent Order. They change together as a single transaction.

**The aggregate rule:** Changes to objects inside an aggregate happen in a single transaction within one service. Changes that span multiple aggregates (or multiple services) happen through events, sagas, or eventual consistency. This rule prevents distributed transactions.

> **Interview Relevance:** Understanding aggregates helps you answer the question "where do you draw the transaction boundary?" in an interview. The answer is: at the aggregate boundary.

---

## SLIDE 27B: Aggregates, Worked Example

**Scenario:** A customer adds 3 items to their cart and places an order on a platform like Amazon.

**The Order Aggregate:**

```python
class Order:
    def __init__(self, order_id, customer_id):
        self.order_id = order_id
        self.customer_id = customer_id
        self.line_items = []
        self.status = "CREATED"

    def add_line_item(self, product_id, product_name, quantity, unit_price):
        line_item = LineItem(product_id, product_name, quantity, unit_price)
        self.line_items.append(line_item)

    def calculate_total(self):
        return sum(item.quantity * item.unit_price for item in self.line_items)

    def confirm(self):
        if not self.line_items:
            raise EmptyOrderError(self.order_id)
        self.status = "CONFIRMED"
```

The Order object controls all modifications to its line items and total. No external service reaches into the Order and modifies a line item directly. All changes go through the Order's methods, which enforce business rules (like "you cannot confirm an empty order").

**What is inside vs outside the aggregate:**

- **Inside (one transaction):** Creating the order, adding line items, calculating the total, confirming the order. All of this happens in a single database transaction within the Order Service.
- **Outside (eventual consistency):** Reserving inventory, authorizing payment, scheduling shipment. These involve other services and happen through events. If payment authorization fails, a compensating action cancels the order. This is a saga, which you will learn in Section 6.

> **Interview Tip:** When designing a service in an interview, explicitly state what is inside your aggregate (consistent) and what is outside (eventually consistent). This shows the interviewer you understand where transactions end and distributed coordination begins.

> **Interview Relevance:** The ability to define aggregate boundaries is what separates candidates who can design services from candidates who can only name them.

---

## SLIDE 27C: Interview Practice

### Question:

You are designing a microservices architecture for an online food ordering platform like DoorDash. The interviewer asks: "How would you determine the service boundaries?"

*Take 60 seconds to think about your approach before reading below.*

---

### Weak Answer:

> "I would create a Restaurant Service, a Menu Service, a Food Item Service, an Order Service, a Delivery Service, a Driver Service, a Payment Service, a Rating Service, and a Notification Service."

**Why this is weak:** The candidate listed 9 services without explaining the reasoning behind any of them. "Menu Service" and "Food Item Service" are almost certainly too granular. They did not mention business capabilities, bounded contexts, team ownership, or any methodology. This looks like guessing, not designing.

---

### Strong Answer:

> "I would start by identifying the core business capabilities. Food ordering has at least five distinct capabilities: restaurant management (onboarding restaurants, managing menus, managing hours), order management (placing orders, tracking status, handling cancellations), dispatch (matching orders to drivers, optimizing routes), payment processing (charging customers, paying restaurants, paying drivers), and customer experience (ratings, reviews, support tickets).
>
> Each of these capabilities has different data, changes for different business reasons, and maps to a different team. The restaurant team manages menus and hours. The order team manages the order lifecycle. The dispatch team handles real-time matching with very different scaling and latency requirements than the order team.
>
> I would validate these boundaries by checking: does each service own its own data? Can each service deploy independently? Does a change in one service rarely force a change in another? If yes, these are good boundaries. If two capabilities frequently change together, I would merge them into one service."

**What makes this strong:** The candidate explains a methodology (business capability decomposition), identifies specific capabilities with reasoning, considers team ownership, and applies a validation test to the boundaries.

---

## SLIDE 28: Cohesion

Cohesion measures how strongly related the responsibilities within a service are. High cohesion means everything inside the service serves a single, focused purpose. Low cohesion means the service is a grab bag of unrelated functionality.

**High cohesion example:** A Payment Service that handles payment authorization, refund processing, payment method management, and payment history. All of these relate to the single business capability of processing payments. When the business adds a new payment provider, all changes happen within this one service.

**Low cohesion example:** A "Utility Service" that handles email sending, PDF generation, currency conversion, and address validation. These have nothing to do with each other. They change for completely different reasons. Email sending changes when the notification requirements change. PDF generation changes when report formats change. There is no business capability that unifies them.

**The cohesion test:** List every responsibility of a service. Can you describe what the service does in one sentence without using the word "and"? If you can ("The Payment Service processes payments"), cohesion is high. If you cannot ("The Utility Service sends emails and generates PDFs and converts currencies and validates addresses"), cohesion is low and the service should be split.

**Why cohesion matters for microservices:** A highly cohesive service changes for one reason, owned by one team, deployed independently. A low-cohesion service changes for many reasons, involves multiple teams, and becomes a deployment bottleneck, which is exactly what microservices are supposed to prevent.

> **Interview Relevance:** When justifying your service boundaries in an interview, saying "I grouped these together because they have high cohesion, they all relate to the payment domain and change together" is a strong, principled answer.

---

## SLIDE 29: Coupling

Coupling measures how much one service depends on another. Low coupling means services can change independently. High coupling means a change in one service forces changes in others.

#### Types of coupling, from worst to most acceptable:

**Database coupling (worst):** Two services read from and write to the same database table. Any schema change breaks both services. This is the shared database anti-pattern from Section 2 and is never acceptable between true microservices.

**Implementation coupling:** Service A depends on the internal implementation of Service B. For example, Service A knows that Service B stores user data in a specific JSON format and parses that format directly. When Service B changes its storage format, Service A breaks.

**Temporal coupling:** Service A can only work when Service B is available at the same moment. Synchronous request-response between services creates temporal coupling. If Service B is down, Service A fails. Asynchronous messaging reduces temporal coupling because Service A can publish a message and continue without waiting.

**Contract coupling (most acceptable):** Service A depends on the public API contract of Service B. This is the minimum necessary coupling. As long as Service B maintains backward compatibility in its API, Service A does not break. This is the coupling you should aim for.

**The goal is not zero coupling.** Services need to communicate. The goal is to push coupling from database and implementation coupling (tight, hidden, fragile) toward contract coupling (explicit, versioned, manageable).

> **Interview Relevance:** Being able to name different types of coupling and explain which types are acceptable versus harmful shows depth of understanding that most candidates lack.

---

## SLIDE 30: Cohesion and Coupling as a Decision Tool

Every service boundary decision comes down to one question: does this split increase cohesion and reduce coupling, or does it do the opposite?

#### When to keep things together (merge):

- Two components always change at the same time for the same business reason
- Splitting them would require constant synchronous communication between the two new services
- They share the same data and splitting would require complex data synchronization
- They are owned by the same team and there is no organizational reason to separate them

#### When to split things apart:

- Two components change for completely different business reasons at different frequencies
- They have different scaling requirements (one is read-heavy, the other is write-heavy)
- They are owned by different teams with different release cadences
- A failure in one should not affect the other
- They have different data storage needs (one needs a relational database, the other needs a document store)

**Common Mistake:** Splitting two components because they "feel" like separate things, even though they always change together. If the Pricing module and the Catalog module are always modified in the same sprint by the same team, they have high cohesion with each other and should probably be one service, even if "pricing" and "catalog" sound like different domains.

> **Interview Tip:** When an interviewer challenges your boundary choice ("why did you put pricing inside the catalog service?"), explain it in terms of cohesion and coupling: "Pricing and catalog always change together when we update product listings, and splitting them would create synchronous coupling for every product read. Keeping them together gives us higher cohesion with no added coupling."

> **Interview Relevance:** The cohesion/coupling reasoning framework is the most versatile tool for defending service boundary decisions in interviews.

---

## SLIDE 31: Ownership Boundaries

In microservices, every service must have one owning team. One team, one service. This is not just an organizational preference. It is an architectural requirement.

#### Why ownership matters:

If two teams co-own a service, every change requires cross-team coordination. Deployment schedules conflict. On-call responsibilities are unclear. Code review standards diverge. The service slowly becomes inconsistent because neither team feels fully responsible.

#### The ownership rules:

- **One team owns one service** - That team controls the code, the database, the deployment pipeline, the monitoring, and the on-call rotation
- **One team can own multiple services** - A team of 6 engineers might own 2-3 closely related services
- **No service has zero owners** - Every service must have a team in the service catalog. Orphaned services become unmaintained liabilities.
- **The owning team owns the API contract** - They decide what the public interface looks like, how it is versioned, and when breaking changes happen

**Real-world example:** In a platform like Uber, the Dispatch team owns the Dispatch Service. They decide how matching works, how the API is structured, and when to deploy changes. The Rider team and Driver team consume the Dispatch API but do not modify the Dispatch Service's code. If the Rider team needs a new dispatch feature, they file a request with the Dispatch team.

**The connection to Conway's Law from Section 1:** This is Conway's Law in practice. The team structure mirrors the service structure. When you draw service boundaries, you are also drawing team boundaries. If the boundaries do not make sense organizationally (one team would own a service but has no expertise in that domain), the boundaries are probably wrong.

> **Interview Relevance:** Mentioning team ownership when explaining your service boundaries shows the interviewer you think about microservices as an organizational architecture, not just a technical one.

---

## SLIDE 32: Service Granularity

One of the hardest decisions in microservices is how big or small each service should be. Both extremes cause problems.

**Too large (the "mini-monolith"):**

A single Order Service handles order creation, payment processing, inventory reservation, shipping calculation, and email notifications. It has 15 database tables and requires 8 engineers to maintain. It deploys twice a week because changes are risky and require extensive testing. This is a monolith with a microservice label. It has all the problems of a monolith (deployment coupling, team bottlenecks) plus the costs of being a distributed service (network overhead, operational complexity).

**Too small (the "nano-service"):**

Separate services for Order Creation, Order Status, Order History, and Order Cancellation. Each has one database table and 200 lines of code. A single "view order details" request requires calling all four services, adding network latency and failure points. Each service is too simple to justify its own deployment pipeline, monitoring, and on-call rotation. The team spends more time managing infrastructure than building features.

**The right granularity:**

A service should be large enough to represent a complete business capability and small enough to be owned by a single team (4-8 engineers). It should have enough internal complexity to justify its operational overhead. A good heuristic: if a service has fewer than 3 database tables, it is probably too small. If it has more than 15, it is probably too large.

> **Interview Relevance:** If an interviewer says "that service seems too big" or "that seems like a lot of services," being able to articulate the granularity trade-off shows you have thought about this beyond the textbook.

---

## SLIDE 32B: Interview Practice

### Question:

You are designing a social media platform like Instagram. A candidate in a mock interview proposes these services: User Profile Service, User Settings Service, User Avatar Service, User Follow Service, User Block Service, User Notification Preferences Service. The interviewer asks you to evaluate this design. What would you say?

*Take 60 seconds to evaluate before reading below.*

---

### Weak Answer:

> "This looks good. Each service is small and focused, which is what microservices should be."

**Why this is weak:** The candidate accepted the design at face value without analyzing cohesion, coupling, or granularity. Six services for user-related functionality is almost certainly too granular.

---

### Strong Answer:

> "This design is too granular. All six of these services relate to a single business capability: user profile management. They are all owned by the same team, change for similar business reasons, and a single user action like 'update my profile' would need to call multiple services.
>
> I would consolidate them into two services. First, a User Service that owns profiles, settings, avatar, and notification preferences. These all change when the user updates their account and they share the same consistency requirements. Second, a Social Graph Service that owns follows and blocks. This is a genuinely different domain with different data structures (a graph rather than a row), different scaling characteristics (fan-out queries for feeds), and potentially a different database technology (a graph database or wide-column store rather than relational).
>
> The User Service has high cohesion because everything inside it relates to a single user's account. The Social Graph Service has high cohesion because everything inside it relates to relationships between users. Coupling between them is minimal, limited to the User Service providing basic profile data when the Social Graph Service needs to display follower names."

**What makes this strong:** The candidate identifies the granularity problem, consolidates with clear reasoning (cohesion, shared ownership, shared data), and draws a principled boundary where a genuine domain difference exists (user account vs social graph).

---

## SLIDE 33: Decomposition by Business Capability vs Technical Layer

There are two fundamentally different approaches to splitting a system into services. One works. The other creates a distributed monolith.

#### Technical layer decomposition (wrong):

```
┌─────────────────────────────────────────────────────────────┐
│                    TECHNICAL LAYER SPLIT                   │
│                                                           │
│  ┌─────────────────────┐                                  │
│  │   API Service       │  ← All endpoints                  │
│  │   (REST endpoints)  │                                  │
│  └──────────┬──────────┘                                  │
│             │                                             │
│  ┌──────────▼──────────┐                                  │
│  │   Business Logic    │  ← All business rules             │
│  │   Service           │                                  │
│  └──────────┬──────────┘                                  │
│             │                                             │
│  ┌──────────▼──────────┐                                  │
│  │   Data Access       │  ← All database queries          │
│  │   Service           │                                  │
│  └─────────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
```

Every feature requires changes to all three services. Adding a "discount code" feature means updating the API Service (new endpoint), the Business Logic Service (discount calculation), and the Data Access Service (new query). These services cannot be developed, deployed, or scaled independently. You have rebuilt the monolith's layers as separate services with network calls in between.

#### Business capability decomposition (correct):

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Catalog       │  │   Order         │  │   Payment       │
│   Service       │  │   Service       │  │   Service       │
│                 │  │                 │  │                 │
│ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │
│ │ API         │ │  │ │ API         │ │  │ │ API         │ │
│ ├─────────────┤ │  │ ├─────────────┤ │  │ ├─────────────┤ │
│ │ Business    │ │  │ │ Business    │ │  │ │ Business    │ │
│ │ Logic       │ │  │ │ Logic       │ │  │ │ Logic       │ │
│ ├─────────────┤ │  │ ├─────────────┤ │  │ ├─────────────┤ │
│ │ Data Access │ │  │ │ Data Access │ │  │ │ Data Access │ │
│ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

Each service contains its own API layer, business logic, and data access for one business capability. Adding a "discount code" feature only changes the Order Service. The Catalog and Payment services do not need to change or redeploy.

**The definitive test:** When a product manager requests a new feature, how many services need to change? If the answer is consistently "only one," you have decomposed by business capability. If the answer is consistently "three or more," you have decomposed by technical layer.

> **Interview Relevance:** If an interviewer sees you drawing a "frontend service, backend service, database service" architecture, they will immediately recognize it as a technical layer split and push back.

---

## SLIDE 34: Decomposition by Workflow

Some service boundaries become clear only when you trace the workflow that a user action triggers from start to finish.

#### Example: E-commerce checkout workflow

When a customer clicks "Place Order," this workflow executes:

```
User Checkout Request
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Create Order     →  Order Service                       │
│ 2. Reserve Inventory →  Inventory Service                  │
│ 3. Authorize Payment →  Payment Service                    │
│ 4. Create Shipment   →  Shipping Service                   │
│ 5. Send Notification →  Notification Service               │
└─────────────────────────────────────────────────────────────┘
```

Each step in this workflow is a distinct responsibility with different data, different failure modes, and different teams. Order creation is a write-heavy transactional step. Inventory reservation needs real-time stock data. Payment authorization involves external payment providers with their own latency and failure characteristics. Shipment creation integrates with logistics partners. Notification sending is fire-and-forget and should never block the order.

#### How to use workflows to find boundaries:

1. Map out the major user workflows in your system (checkout, signup, content upload, search)
2. For each workflow, identify the distinct steps
3. Group steps that share data and change for the same business reason
4. Steps that have different scaling needs, different failure tolerance, or different team ownership become separate services

**When two workflow steps should stay together:** If step A and step B always happen in the same transaction and a failure in either means both must roll back, they probably belong in the same service. Splitting them would require a distributed transaction, which adds complexity for no benefit.

> **Interview Relevance:** Tracing a workflow on the whiteboard and then deriving service boundaries from it is one of the strongest interview techniques for demonstrating decomposition skill.

---

## SLIDE 35: Decomposition by Read and Write Patterns

Some service boundaries emerge from the observation that reads and writes have fundamentally different characteristics in many domains.

**The pattern:** In many systems, the write path and the read path differ in volume, latency requirements, data shape, and scaling needs. When this difference is large enough, separating them into different services (or at least different models) is justified.

#### Example: Product catalog in an e-commerce platform

- **Writes** are low volume. Product managers update listings a few hundred times per day. Each update involves complex validation, image processing, and category assignment. The write model is normalized and relational.
- **Reads** are extremely high volume. Millions of customers browse the catalog every hour. They need fast, denormalized responses with product name, price, main image, and average rating in a single query. The read model is denormalized and optimized for fast retrieval.

Trying to serve both patterns from the same service and the same database means compromising on both. The write model becomes denormalized to speed up reads, making writes more complex. The read model becomes normalized to simplify writes, making reads slower.

**The CQRS approach (preview):**

Command Query Responsibility Segregation separates the write model (commands) from the read model (queries). The write service processes updates and publishes events. The read service consumes those events and maintains an optimized read model. Section 5 (Data Ownership) will cover CQRS in full detail.

**When this decomposition applies:** When read volume is 10x-1000x higher than write volume, when read and write latency requirements differ significantly, or when the ideal data shape for reading differs from the ideal shape for writing.

> **Interview Relevance:** Mentioning read/write separation as a decomposition strategy shows the interviewer you think about data access patterns, not just business domains.

---

## SLIDE 36: Why One Service Per Database Table Fails

This is one of the most common mistakes in microservices design and it comes up frequently in interviews. It deserves its own slide because the reasoning behind why it fails illustrates the core principles of good decomposition.

**The mistake:** A team looks at their database schema, sees 12 tables, and creates 12 services. The User Service owns the users table. The Address Service owns the addresses table. The Phone Number Service owns the phone_numbers table.

#### Why it fails:

**Problem 1: No cohesion.** An Address Service with one table has no meaningful business logic. It is a CRUD wrapper around a database table. It adds network overhead, a deployment pipeline, and monitoring for something that could be a function call inside the User Service.

**Problem 2: Chatty communication.** Displaying a user profile requires calling the User Service, then the Address Service, then the Phone Number Service, then the Email Service. Four network calls for what was a single database join. Latency increases. Failure probability increases. Debugging becomes harder.

**Problem 3: Wrong consistency boundary.** A user updates their profile with a new name and new address. In a monolith, this is one transaction. With separate services, you now need to coordinate a distributed update across User Service and Address Service. If one succeeds and the other fails, the user has an inconsistent profile.

**The fix:** Group related tables into services based on business capability. Users, addresses, phone numbers, and emails all belong to the User Profile capability. They form one aggregate, owned by one service, changed in one transaction.

> **Interview Relevance:** If an interviewer asks "how many services would this system need?" and you count database tables, they will immediately know you have the wrong decomposition instinct.

---

## SLIDE 37: Service Boundary Anti-Patterns

Beyond the decomposition anti-patterns from Section 2, there are specific boundary-level mistakes that emerge during the design phase.

**Anti-pattern 1: Circular dependencies**

The Order Service calls the Payment Service to process payment. The Payment Service calls back to the Order Service to update order status. This creates a cycle where neither service can function without the other. The fix: the Payment Service publishes a "payment completed" event, and the Order Service consumes it. Data flows in one direction.

**Anti-pattern 2: The "just one more call" chain**

Service A calls B, which calls C, which calls D, which calls E. A single user request triggers a chain of 5 synchronous calls. Total latency is the sum of all 5 calls. If any one service is slow or down, the entire chain fails. The fix: evaluate whether A truly needs all that data, or if some calls can be replaced with cached data or asynchronous events.

**Anti-pattern 3: Anemic services**

A service that has no business logic and only forwards requests to another service. The "API Routing Service" that receives requests and passes them to the real services is not a useful boundary. It is an unnecessary network hop. The API Gateway handles this responsibility.

**Anti-pattern 4: Shared kernel coupling**

Two services import the same shared library that contains business logic. When the shared library is updated, both services must redeploy. Shared libraries for infrastructure concerns (logging, metrics, HTTP client) are fine. Shared libraries for business logic defeat the purpose of independent deployment.

> **Interview Relevance:** Being able to spot these anti-patterns in a whiteboard design, whether your own or one the interviewer presents, is a key skill for senior-level interviews.

---

## SLIDE 37B: Note It Down

### Service boundary anti-patterns to memorize:

1. **Circular dependencies** - If A calls B and B calls A, break the cycle with events
2. **Call chains** - If a request triggers 5+ synchronous hops, replace some with caching or events
3. **Anemic services** - If a service has no business logic and only proxies to another, remove it
4. **Shared business logic libraries** - If two services share a library with business rules, they are coupled at deployment time
5. **Database table = service** - If each table is a service, you have decomposed by schema, not capability
6. **God service** - If one service handles multiple unrelated business capabilities, split it

**The universal test for every boundary:**

Can this service be developed, deployed, and operated by one team without coordinating with other teams for most changes? If yes, the boundary is good. If no, the boundary needs to move.

> **Interview Relevance:** Memorizing these anti-patterns prepares you for the common interview question: "What could go wrong with this microservices design?"

---

## SLIDE 38: Boundary Discovery Techniques

How do you actually find the right boundaries in practice? These are the techniques used by real engineering teams.

**Technique 1: Event Storming**

Get domain experts, engineers, and product managers in a room. Use sticky notes on a wall. Write down every event that happens in the system ("Order Placed," "Payment Authorized," "Inventory Reserved," "Shipment Created"). Group related events. The groups that form naturally often correspond to service boundaries. Events that cross between groups become the inter-service communication.

**Technique 2: Start with the transaction boundaries**

Identify every operation that must be atomic (all-or-nothing). Everything that must succeed or fail together belongs in one service. If "create order" and "add line items" must be one transaction, they belong in the same service. If "create order" and "reserve inventory" can be eventually consistent, they can be separate services.

**Technique 3: Ask "what changes together?"**

Look at your version control history. Which files always change in the same commit? Which modules always appear in the same pull request? Code that changes together has high cohesion and should stay together.

**Technique 4: Follow the team structure**

If your organization already has a Payments team and an Orders team, those are your starting boundaries. Conway's Law will push the architecture in this direction regardless, so you might as well align proactively.

> **Interview Tip:** If an interviewer asks "how would you discover service boundaries for a system you are not familiar with?", mentioning event storming and transaction boundary analysis shows practical methodology, not guesswork.

> **Interview Relevance:** Interviewers do not just want to see your final service boundaries. They want to see the process you used to arrive at them.

---

## SLIDE 39: Real Example, E-Commerce Service Boundaries

**System:** An e-commerce platform like Amazon

```
┌─────────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE MICROSERVICES                        │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Catalog    │  │  Cart       │  │  Order      │              │
│  │  Service    │  │  Service    │  │  Service    │              │
│  │             │  │             │  │             │              │
│  │ • Products  │  │ • Cart      │  │ • Order     │              │
│  │ • Categories│  │ • Saved     │  │ • Status    │              │
│  │ • Images    │  │   Items     │  │ • History   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Payment    │  │  Inventory  │  │  Shipping   │              │
│  │  Service    │  │  Service    │  │  Service    │              │
│  │             │  │             │  │             │              │
│  │ • Auth      │  │ • Stock     │  │ • Shipments │              │
│  │ • Refunds   │  │ • Warehouse │  │ • Tracking  │              │
│  │ • Payment   │  │ • Restock   │  │ • Labels    │              │
│  │   Methods   │  │             │  │             │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

#### Why these boundaries work:

- **Catalog Service** - Manages product listings, descriptions, images, categories, search indexing. Changes when the merchandising team updates product content. Read-heavy, needs caching and search optimization.
- **Cart Service** - Manages shopping carts, saved items, cart persistence. Changes when the conversion team experiments with cart features. Needs session-level storage with fast reads and writes.
- **Order Service** - Manages order lifecycle from creation to completion. Changes when order workflow rules change. The core aggregate is the Order with its line items.
- **Payment Service** - Handles authorization, capture, refunds, payment methods. Changes when new payment providers are added. Integrates with external payment gateways.
- **Inventory Service** - Tracks stock levels, warehouse locations, reservations, restocking. Changes when warehousing logistics change. Needs real-time accuracy for stock counts.
- **Shipping Service** - Manages shipment creation, carrier integration, tracking, delivery estimates. Changes when logistics partnerships change.

Each service maps to a distinct business capability, is owned by a separate team, and has its own database.

> **Interview Relevance:** Having a well-reasoned e-commerce decomposition ready to present saves time in interviews and demonstrates experience with the most common system design problem domain.

---

## SLIDE 40: Real Example, Ride-Sharing Service Boundaries

**System:** A ride-sharing platform like Uber

| Service | Business Capability | Key Data | Why It Is Separate |
|---------|---------------------|----------|-------------------|
| **Rider Service** | Rider accounts, ride requests, ride history | Rider profiles, saved locations, ride preferences | Rider-facing features change independently from driver features |
| **Driver Service** | Driver onboarding, availability, earnings | Driver profiles, documents, vehicle info, earnings records | Driver lifecycle is a separate domain with different regulations |
| **Trip Service** | Active trip management, trip lifecycle | Trip state (requested, matched, in-progress, completed), route, timestamps | The core domain object. Trip lifecycle is the most complex workflow. |
| **Pricing Service** | Surge pricing, fare estimation, fare calculation | Pricing models, demand data, fare rules | Pricing algorithms change frequently and independently. Needs real-time demand data. |
| **Dispatch Service** | Driver-rider matching, route optimization | Real-time driver locations, matching algorithms, ETA calculations | Latency-critical (sub-second matching). Needs different infrastructure (in-memory, geospatial). |
| **Payment Service** | Rider charges, driver payouts, promotions | Payment methods, transaction records, promo codes | Financial data has strict compliance requirements and separate audit needs. |

**Key insight:** The Dispatch Service and the Pricing Service both need real-time location data, but they are separate services because they change for different reasons. Pricing changes when the economics team adjusts surge algorithms. Dispatch changes when the engineering team optimizes matching algorithms. They have different release cadences and different teams.

> **Interview Relevance:** Ride-sharing is one of the most frequently asked system design problems. Having a clear service decomposition with reasoning for each boundary prepared in advance is a significant advantage.

---

## SLIDE 41: Real Example, Streaming Platform and Banking

### System 1: A streaming platform like Netflix

- **User Service** - Profiles, preferences, parental controls, account settings
- **Catalog Service** - Movie and show metadata, genres, cast, availability by region
- **Recommendation Service** - Personalized suggestions based on viewing history and similar users. Compute-intensive, uses ML models, changes independently from catalog content.
- **Playback Service** - Stream initiation, quality adaptation, DRM, resume position. Latency-critical, closest to the CDN infrastructure, very different scaling profile from recommendations.
- **Billing Service** - Subscriptions, payment processing, plan management, invoicing. Changes when pricing tiers change. Strict financial compliance requirements.

**Key insight:** Recommendation and Playback are both part of "watching a movie," but they have completely different scaling profiles, technology stacks, and team expertise. Recommendations are batch-computed and ML-heavy. Playback is real-time and infrastructure-heavy. This is a textbook case where different scaling and technology needs drive separate services.

### System 2: A banking platform

- **Account Service** - Account creation, balance inquiries, account status. The core entity that everything references.
- **Ledger Service** - Transaction recording, double-entry bookkeeping, balance updates. Every financial movement is a ledger entry. This service must be append-only and auditable.
- **Transfer Service** - Initiating and executing transfers between accounts. Orchestrates the saga: debit source account, credit destination account, record in ledger.
- **Risk Service** - Fraud detection, transaction limits, suspicious activity monitoring. Evaluates every transfer in real-time. Changes when compliance regulations change.
- **Notification Service** - Alerts for transactions, low balance, suspicious activity. Fire-and-forget. Should never block a financial transaction.

**Key insight:** In banking, the Ledger Service is the most critical service because it is the system of record. Every other service eventually writes to or reads from the ledger. This is different from e-commerce where the Order Service is central.

> **Interview Relevance:** Having decomposition examples across multiple domains (e-commerce, ride-sharing, streaming, banking) shows the interviewer you can apply decomposition principles to any system, not just the one you memorized.

---

## SLIDE 41B: Interview Practice

### Question:

An interviewer asks you to decompose a messaging platform like WhatsApp into microservices. You have 2 minutes. Go.

*Take 60 seconds to think about your decomposition before reading below.*

---

### Weak Answer:

> "I would have a Message Service, a User Service, a Group Service, a Media Service, a Notification Service, and a Presence Service."

**Why this is weak:** The candidate listed services without explaining why these boundaries exist. More importantly, they did not discuss the unique characteristics of messaging that should drive the decomposition (real-time delivery, massive fan-out for groups, presence at scale, media storage).

---

### Strong Answer:

> "I would decompose based on the distinct business capabilities and their very different scaling and technical requirements.
>
> First, a **User Service** for account management, contacts, and profile information. This is standard CRUD with relatively low write volume.
>
> Second, a **Messaging Service** for the core message delivery pipeline. This is the heart of the system, handling one-to-one and group message routing, delivery receipts, and end-to-end encryption. This service has extreme throughput requirements. A platform like WhatsApp handles over 100 billion messages per day. It needs a specialized storage engine optimized for write-heavy sequential message storage.
>
> Third, a **Group Service** for group creation, membership management, and group metadata. I am separating this from messaging because group management (adding members, changing names, setting permissions) is a different business capability from message delivery, and group fan-out (sending one message to 256 members) has different scaling characteristics than one-to-one delivery.
>
> Fourth, a **Media Service** for image, video, and document storage and delivery. Media has completely different storage requirements (blob storage, CDN delivery) and different latency tolerance (uploading a video can take seconds, but the upload should not block text message delivery).
>
> Fifth, a **Presence Service** for online/offline status and last-seen timestamps. This is extremely high-frequency data (every app open/close is a presence update for 2 billion users) and needs a specialized in-memory data store rather than a relational database."

**What makes this strong:** The candidate explains the reasoning behind each boundary (different scaling profiles, different storage needs, different business capabilities) and includes real-world scale numbers to justify the separation.

---

## SLIDE 42: Decomposition Decision Framework

When you are unsure whether to split or merge two components, run through this checklist.

#### Split into separate services when:

- Different teams own them and have different release cadences
- They have significantly different scaling requirements (10x+ difference in load)
- They change for different business reasons at different frequencies
- A failure in one should not affect the other
- They need different data storage technologies
- They have different compliance or security requirements

#### Keep together in one service when:

- They always change together for the same business reason
- Splitting would require synchronous calls between them for most operations
- They share the same data and need transactional consistency
- They are owned by the same team with no foreseeable organizational split
- One cannot do anything useful without the other

#### The final validation:

After you draw your service boundaries, check each boundary against these three questions:

1. **Independence test** - Can each service deploy without coordinating with other services?
2. **Cohesion test** - Can you describe each service's purpose in one sentence without "and"?
3. **Coupling test** - Does a change in one service rarely require a change in another?

If all three answers are yes for every service, your boundaries are sound.

> **Interview Tip:** Running through this checklist out loud during an interview shows the interviewer you have a repeatable process for validating service boundaries, not just intuition.

> **Interview Relevance:** This framework is the most practical tool in this section. Memorize the split criteria, the merge criteria, and the three validation questions.

---

## SLIDE 43: Section Summary and Interview Tips

### Section 3 Summary:

| Concept | Key Takeaway |
|---------|--------------|
| **Business capability decomposition** | Services should map to business functions, not technical layers or database tables |
| **Bounded contexts** | The same word means different things in different domains. Each context gets its own model. |
| **Aggregates** | The consistency boundary within a service. Everything inside is one transaction. |
| **Cohesion** | Group things that change together for the same reason |
| **Coupling** | Minimize dependencies between services. Aim for contract coupling, avoid database coupling. |
| **Ownership** | One team, one service. Team boundaries should match service boundaries. |
| **Granularity** | 4-8 engineers per service, 3-15 database tables. Enough to be useful, small enough to be independent. |
| **Workflow decomposition** | Trace user workflows to discover where service boundaries naturally fall |
| **Read/write decomposition** | When read and write patterns differ dramatically, consider separating them |

### Common Interview Mistakes:

| Mistake | Why It Hurts You | Stronger Approach |
|---------|------------------|-------------------|
| Listing services without explaining the reasoning | Shows memorization, not design skill | "I chose these boundaries because each maps to a distinct business capability with different scaling needs and team ownership" |
| One service per database table | Creates chatty, anemic services with no cohesion | Group related tables under one business capability |
| Splitting by technical layer | Every feature touches every service, defeating independence | Split by business capability so each feature changes one service |
| Ignoring team structure | Architecture that does not match org structure will drift | Align service boundaries with team boundaries per Conway's Law |
| No validation of boundaries | Boundaries chosen by gut feel with no principled check | Apply the independence, cohesion, and coupling tests to each boundary |

### Key Numbers to Remember:

| Metric | Value |
|--------|-------|
| Ideal team size per service | 4-8 engineers |
| Database tables per service (heuristic) | 3-15 tables |
| Feature change test | A feature should change 1 service, not 3+ |
| WhatsApp daily messages | ~100 billion |
| Typical bounded contexts in e-commerce | 5-8 (catalog, cart, order, payment, inventory, shipping, customer, analytics) |

### How This Connects to Other Sections:

**Section 4 (Communication Patterns)** addresses the next question after boundaries: how do these services talk to each other? The synchronous vs asynchronous decision depends directly on the coupling level you are willing to accept between services.

**Section 5 (Data Ownership)** tackles the hardest consequence of good boundaries: when each service owns its data, how do you handle cross-service queries and data duplication?

**Section 6 (Sagas and Consistency)** builds directly on aggregates. Everything inside an aggregate is a local transaction. Everything across aggregates requires the saga pattern.

--

# Section 4: Inter-Service Communication

# Section 4: Inter-Service Communication

### **SLIDE 44: Section Intro**

**Section 4: Inter-Service Communication**

Once you have drawn service boundaries (Section 3), the next question is immediate: how do these services talk to each other? The answer to this question is not a technology choice. It is an architecture decision that determines coupling, latency, failure behavior, and consistency across your entire system.

Choosing REST vs gRPC vs message queues is not about which technology is “better.” It is about which communication pattern matches the coupling and consistency requirements of each specific interaction. A system typically uses multiple patterns for different interactions.

**What We Will Cover:**

1. Synchronous vs asynchronous communication and the trade-offs of each
2. REST, gRPC, and GraphQL for synchronous communication
3. Message queues and event streams for asynchronous communication
4. Commands vs events and why the distinction matters
5. Resilience patterns: timeouts, retries, circuit breakers, bulkheads, and backpressure
6. Contract versioning and schema evolution for independent deployment
7. Correlation IDs for tracing requests across services
8. A decision framework for choosing the right pattern

**Where This Appears in Real Systems:**

In a platform like Uber, the Rider App uses REST to request a ride. The Dispatch Service uses gRPC internally for low-latency driver matching. The Trip Service publishes events to a stream (like Kafka) that the Payment Service, Analytics Service, and Notification Service consume independently. A single ride request uses at least three different communication patterns. This section teaches you when and why to use each.

---

### **SLIDE 45: Synchronous vs Asynchronous Communication**

Every inter-service communication falls into one of two categories, and the choice between them is the most important communication decision you will make.

**Synchronous communication:**

Service A sends a request to Service B and waits for a response before continuing. The caller is blocked until the response arrives or a timeout occurs.

!image.png

**Asynchronous communication:**

Service A sends a message and continues immediately without waiting for a response. Service B processes the message whenever it is ready.

!image.png

**The fundamental trade-off:**

- **Synchronous** gives you immediate feedback (success/failure) but creates temporal coupling. If Service B is down, Service A fails too.
- **Asynchronous** removes temporal coupling (Service A does not depend on Service B being available right now) but you lose immediate feedback. You do not know if Service B processed the message successfully until later.

**A practical rule:** Use synchronous communication when the caller needs the response to continue its work. Use asynchronous communication when the caller does not need to wait for the result.

**Interview Relevance:** The first communication decision in any interview should be sync vs async for each interaction. Starting here shows you are thinking about coupling and failure modes, not just picking a technology.

---

### **SLIDE 46: REST**

REST (Representational State Transfer) is the most common communication protocol for microservices. It uses HTTP and is the default choice for public APIs and straightforward request-response interactions.

**When REST fits well:**

- Public-facing APIs where broad client compatibility matters (browsers, mobile apps, third-party integrations)
- CRUD-style operations (create, read, update, delete) on resources
- Interactions where simplicity and widespread tooling support matter more than raw performance

**A typical REST interaction:**

```
POST /orders HTTP/1.1
Host: order-service.internal
Content-Type: application/json

{
  "customer_id": "cust_12345",
  "items": [
    {"product_id": "prod_789", "quantity": 2}
  ]
}
```

```
HTTP/1.1 201 Created
Content-Type: application/json

{
  "order_id": "ord_98765",
  "status": "CREATED",
  "total": 59.98
}
```

**REST limitations in microservices:**

- **Over-fetching:** The client receives all fields even if it only needs two. A mobile app displaying an order summary gets the full order object with 30 fields.
- **Chattiness:** Displaying a product page might require calling the Catalog Service, Pricing Service, Inventory Service, and Reviews Service separately. Four HTTP round trips.
- **No streaming:** Standard REST is request-response. It does not support server-to-client streaming or bidirectional communication natively.
- **Text-based overhead:** JSON payloads are human-readable but larger than binary formats. For internal service-to-service calls at high throughput, this overhead adds up.

**Interview Relevance:** REST is the safe default for any external-facing API. For internal service-to-service calls, explain when you would use it and when you would choose gRPC instead.

---

### **SLIDE 47: gRPC**

gRPC is a high-performance RPC framework that uses Protocol Buffers (protobuf) for serialization and HTTP/2 for transport. It is designed for internal service-to-service communication where performance and type safety matter.

**When gRPC fits well:**

- Internal service-to-service calls where both ends are under your control
- High-throughput, low-latency communication (10x-100x more calls per second than REST)
- Strongly typed contracts where you want compile-time guarantees that the request and response structures are correct
- Streaming use cases (server streaming, client streaming, bidirectional streaming)

**How a gRPC contract is defined:**

```protobuf
service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc StreamOrderUpdates(GetOrderRequest) returns (stream OrderUpdate);
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
}
```

**gRPC vs REST:**

- Protobuf messages are binary, roughly 5-10x smaller than equivalent JSON
- HTTP/2 supports multiplexing (multiple requests over one connection) and header compression
- Code generation produces typed client and server stubs in any language. No hand-written HTTP parsing.
- The trade-off: gRPC is harder to debug (binary payloads are not human-readable), harder to call from browsers without a proxy, and requires both sides to share the proto definition file

**Interview Tip:** A strong pattern in interviews is “REST for external APIs, gRPC for internal service-to-service calls.” This shows you understand that the right protocol depends on who the consumer is, not which technology is newer.

**Interview Relevance:** Mentioning gRPC for internal communication shows the interviewer you think about performance and type safety at scale, not just developer convenience.

---

### **SLIDE 48: GraphQL**

GraphQL is a query language for APIs that lets the client specify exactly which fields it needs. It is not a replacement for REST or gRPC. It serves a different purpose.

**When GraphQL fits well:**

- Client-facing APIs where different clients (web, iOS, Android) need different subsets of the same data
- Aggregation layers that combine data from multiple backend services into one client response
- Situations where over-fetching and under-fetching from REST APIs are causing performance problems on the client side

**How GraphQL solves the over-fetching problem:**

A mobile app displaying an order summary only needs order ID, status, and total. With REST, it receives the full order object with 30 fields. With GraphQL, the client requests exactly what it needs:

```graphql
query {
  order(id: "ord_98765") {
    orderId
    status
    total
  }
}
```

The response contains only those three fields. No wasted bandwidth, no unnecessary data processing on the client.

**Where GraphQL fits in a microservices architecture:**

GraphQL works best as an edge layer, not as a service-to-service protocol. A GraphQL gateway sits between clients and backend microservices. It translates client queries into calls to the appropriate backend services (which use REST or gRPC internally) and assembles the response.

**GraphQL risks:**

- A single complex query can trigger expensive joins across multiple backend services. Without query complexity limits, clients can accidentally (or maliciously) overload the system.
- The GraphQL gateway can become a “god gateway” that accumulates business logic. It should only translate and aggregate, never contain business rules.
- Caching is harder than REST because every query can be unique.

**Interview Relevance:** Mentioning GraphQL as an edge aggregation layer (not as a service-to-service protocol) shows you understand where it adds value and where it creates risk.

---

### **SLIDE 49: REST vs gRPC vs GraphQL**

| Dimension | REST | gRPC | GraphQL |
| --- | --- | --- | --- |
| Best for | Public APIs, CRUD, broad compatibility | Internal service-to-service, high throughput | Client-facing aggregation, mobile apps |
| Payload format | JSON (text) | Protobuf (binary) | JSON (text) |
| Performance | Moderate | High (5-10x smaller payloads, HTTP/2) | Moderate (depends on query complexity) |
| Type safety | Weak (no compile-time contract) | Strong (generated from proto definitions) | Moderate (schema-defined) |
| Streaming | Not native | Built-in (server, client, bidirectional) | Subscriptions (separate protocol) |
| Browser support | Native | Requires proxy (gRPC-web) | Native |
| Debugging | Easy (human-readable JSON) | Harder (binary payloads) | Moderate |
| Caching | Straightforward (HTTP caching) | Harder (binary, custom caching) | Hard (dynamic queries) |
| Versioning risk | URI or header versioning | Proto field numbering, backward compatible | Schema evolution |

**The typical microservices pattern:**

- REST for public-facing APIs and third-party integrations
- gRPC for internal service-to-service calls where performance matters
- GraphQL as an optional edge layer for mobile/web clients that need flexible queries
- Many systems use REST and gRPC together, with no GraphQL layer at all

**Interview Relevance:** An interviewer does not want to hear “I would use REST.” They want to hear “I would use REST for the public API because it has broad client compatibility, and gRPC for internal calls between the Order Service and Payment Service because the latency budget is tight and we want typed contracts.”

---

### **SLIDE 49B: Interview Practice**

**Question:**

You are designing the communication between services in a food delivery platform. The interviewer asks: “How would the Rider App communicate with your backend, and how would your backend services communicate with each other?”

Take 60 seconds to think about your answer before reading below.

**Weak Answer:**

“Everything uses REST. It is simple, well-understood, and easy to implement.”

**Why this is weak:** Using REST for everything ignores the different requirements of external vs internal communication. Internal calls between the Dispatch Service and Driver Location Service happen thousands of times per second with a sub-10ms latency budget. REST with JSON serialization adds unnecessary overhead for these calls.

**Strong Answer:**

“I would use different protocols for different interactions based on their requirements.

For the Rider App to backend communication, I would use REST through an API Gateway. REST gives us broad compatibility across iOS, Android, and web clients, and the API Gateway handles authentication, rate limiting, and request routing.

For internal service-to-service calls, I would use gRPC. The Dispatch Service needs sub-10ms responses from the Driver Location Service when matching a rider with nearby drivers. gRPC’s binary serialization and HTTP/2 transport give us the performance we need, and the proto definitions give us compile-time type safety between teams.

For communication that does not need immediate responses, I would use asynchronous events. When a trip is completed, the Trip Service publishes a trip.completed event. The Payment Service consumes it to charge the rider. The Analytics Service consumes it to update dashboards. The Notification Service consumes it to send the receipt. None of these need to block the trip completion flow.”

**What makes this strong:** The candidate matches each communication pattern to a specific requirement (client compatibility, low latency, decoupling) and explains why the choice fits.

---

### **SLIDE 50: Message Queues**

A message queue is a buffer between a producer (the service sending a message) and a consumer (the service processing it). The producer puts a message on the queue and moves on. The consumer picks it up when ready.

!image.png

**Key characteristic:** Each message is delivered to exactly one consumer. If three workers are consuming from the same queue, each message goes to only one of them. This makes queues ideal for task distribution.

**When to use message queues:**

- **Background processing** - User uploads a video. The API returns immediately. A worker picks up the encoding task from the queue and processes it asynchronously.
- **Load leveling** - The Order Service receives 10,000 orders per minute during a flash sale. The Payment Service can only handle 2,000 per minute. The queue absorbs the burst, and the Payment Service processes at its own pace.
- **Decoupling** - The Order Service does not need to know which service processes payments. It puts a message on the queue. The Payment Service (or any replacement) consumes it.

**Real-world example:** In a platform like Amazon, when you click “Place Order,” the API returns immediately with your order confirmation. The actual payment processing, inventory reservation, and shipping creation happen asynchronously via messages. This is why your order status says “Processing” for a few seconds before updating.

**Interview Relevance:** Message queues are the right answer whenever the caller does not need an immediate response and the work can be processed in the background. Using synchronous calls for background work is a common interview mistake.

---

### **SLIDE 51: Event Streams**

An event stream is a durable, ordered log of events that multiple consumers can read independently. Unlike a message queue where each message goes to one consumer, an event stream allows every consumer to read every event.

!image.png

**Key characteristic:** Events are persisted in the stream. Consumers maintain their own read position (offset). Each consumer reads at its own pace. Adding a new consumer does not affect existing ones. You can even replay events from the beginning of the log.

**When to use event streams:**

- **Multiple consumers need the same event** - An “order placed” event is relevant to payments, inventory, shipping, analytics, and notifications. A queue would deliver it to only one of them.
- **Event history matters** - A new Analytics Service is deployed and needs to process all historical order events to build its initial dashboard. It reads the stream from the beginning.
- **Event-driven architecture** - Services react to events rather than being told what to do. The Payment Service does not receive a command “process this payment.” It observes an event “an order was placed” and decides to process the payment.

**Real-world example:** A platform like LinkedIn uses event streams (Kafka) to propagate profile updates. When you update your job title, that event is consumed by the search index (to update search results), the feed service (to notify your connections), the recommendation engine (to update job suggestions), and the analytics service (to track engagement). Each consumer processes the event independently.

**Interview Relevance:** Event streams are the right answer when multiple services need to react to the same event independently. This is one of the most common patterns in large-scale microservices.

---

### **SLIDE 52: Commands vs Events**

This distinction is subtle but critical. Getting it wrong creates tight coupling between services even when using asynchronous messaging.

**Commands: “Do this”**

A command is an instruction directed at a specific service. The sender knows who will process it and expects a specific outcome.

Example: “ProcessPayment(order_id=123, amount=59.98)” is sent to the Payment Service. The sender expects the payment to be processed.

**Events: “This happened”**

An event is a notification that something occurred. The publisher does not know or care who consumes it. It does not expect any specific outcome.

Example: “OrderPlaced(order_id=123, customer_id=456, items=[…])” is published to a stream. The publisher has no idea which services will react to it.

**Why this matters for coupling:**

- **Commands create coupling.** The Order Service must know that the Payment Service exists and what API it exposes. If the Payment Service is renamed, relocated, or replaced, the Order Service must be updated.
- **Events reduce coupling.** The Order Service publishes “OrderPlaced” without knowing that the Payment Service, Notification Service, and Analytics Service all consume it. New consumers can be added without changing the Order Service.

**When to use each:**

- Use commands when you need a guaranteed outcome from a specific service (e.g., “authorize this payment before confirming the order”)
- Use events when you want to notify the system that something happened and let interested services react independently (e.g., “an order was placed, whoever cares can act on it”)

**Interview Tip:** When describing your communication patterns in an interview, use the correct terminology. “The Order Service publishes an OrderPlaced event” sounds fundamentally different from “the Order Service sends a ProcessPayment command to the Payment Service.” The first is loosely coupled. The second is tightly coupled. Interviewers notice this distinction.

**Interview Relevance:** Using “commands” and “events” correctly in an interview signals that you understand coupling at the message level, not just at the service level.

---

### **SLIDE 53: Message Queues vs Event Streams**

| Dimension | Message Queue (RabbitMQ, SQS) | Event Stream (Kafka, Kinesis) |
| --- | --- | --- |
| Delivery model | Each message to one consumer | Each event to all consumers |
| Message persistence | Deleted after consumption | Retained for configurable duration (days, weeks, forever) |
| Consumer tracking | Queue tracks what has been delivered | Each consumer tracks its own position (offset) |
| Replay capability | No. Once consumed, the message is gone. | Yes. Consumers can re-read from any point in the log. |
| Ordering | Best-effort or per-queue FIFO | Ordered within a partition |
| Adding new consumers | Does not affect existing consumers | Does not affect existing consumers. New consumer can read history. |
| Best for | Task distribution, background jobs, load leveling | Event broadcasting, event history, event-driven architecture |
| Throughput | Moderate (thousands/sec) | Very high (millions/sec with partitioning) |

**Choosing between them:**

- If only one consumer should process each message (background jobs, work distribution), use a message queue.
- If multiple consumers need the same event (broadcasting to payment, analytics, notifications), use an event stream.
- If you need to replay historical events (rebuilding read models, onboarding new services), use an event stream.
- Many systems use both. Message queues for specific task routing. Event streams for system-wide event broadcasting.

**Interview Relevance:** Knowing the difference between a message queue and an event stream, and choosing the right one for each interaction, is a specific skill interviewers test for.

---

### **SLIDE 54: Request Choreography and the Fan-Out Problem**

When a single user action triggers calls to multiple downstream services, you face the fan-out problem. How you handle it determines your system’s latency and resilience.

**The fan-out problem:**

!image.png

A single “place order” request triggers 5 downstream calls. If each call takes 50ms, and they happen sequentially, total latency is 250ms. If the Fraud Service is slow (200ms), the entire order takes 400ms.

**Strategies to manage fan-out:**

**Parallelize independent calls.** Inventory check, pricing calculation, and fraud check can happen simultaneously. Only payment authorization must wait for the fraud check result. Parallel execution reduces latency from the sum of all calls to the duration of the longest parallel branch.

**Make non-critical calls asynchronous.** Notification does not need to complete before the order is confirmed. Publish an event and let the Notification Service process it later. This removes it from the critical path entirely.

**Cache frequently needed data.** If the Order Service calls the Pricing Service for every order and pricing data changes infrequently, cache pricing data locally. This eliminates a network call for most requests.

**Set per-hop timeout budgets.** If the total request budget is 500ms, allocate 100ms to inventory, 100ms to pricing, 200ms to payment (including fraud), and 100ms for overhead. If any hop exceeds its budget, fail fast rather than waiting.

**Interview Relevance:** When an interviewer sees a fan-out in your design, they will ask how you handle it. Having a strategy (parallelize, async non-critical, cache, timeout budgets) ready is essential.

---

### **SLIDE 54B: Interview Practice**

**Question:**

You are designing the checkout flow for an e-commerce platform. When a user clicks “Place Order,” your Order Service needs to: verify inventory, calculate the final price (with discounts and tax), authorize payment, create a shipment record, and send a confirmation email. The interviewer asks: “Walk me through the communication pattern for this flow.”

Take 60 seconds to design the flow before reading below.

**Weak Answer:**

“The Order Service calls each downstream service one at a time: first Inventory, then Pricing, then Payment, then Shipping, then Notification. Each call is REST.”

**Why this is weak:** Sequential synchronous calls create maximum latency (sum of all calls) and maximum fragility (any one failure blocks everything). Notification has no business being on the critical path, it does not affect whether the order succeeds.

**Strong Answer:**

“I would split these into critical-path synchronous calls and non-critical asynchronous events.

On the critical path, I would parallelize where possible. Inventory check and pricing calculation are independent, so they run in parallel. Both must succeed before we proceed. Then payment authorization happens synchronously because we need to know if payment succeeded before confirming the order.

Shipment creation is important but does not need to block the order confirmation. The Order Service publishes an OrderConfirmed event, and the Shipping Service consumes it to create the shipment asynchronously. If shipment creation fails, it retries from the event.

Notification is fully asynchronous. The Order Service publishes the same OrderConfirmed event, and the Notification Service consumes it to send the confirmation email. If the email fails, the order is still valid.

This gives us a critical path of: max(inventory_check, pricing_calculation) + payment_authorization. If inventory and pricing each take 50ms and payment takes 100ms, total checkout latency is about 150ms instead of 300ms for sequential calls.”

**What makes this strong:** The candidate separates critical from non-critical, parallelizes independent calls, uses async for fire-and-forget steps, and provides specific latency math.

---

### **SLIDE 55: Timeout Strategy**

Every network call in a microservices system must have a timeout. A call without a timeout can hang indefinitely, holding resources (threads, connections, memory) that other requests need.

**The problem without timeouts:**

The Payment Service becomes slow (responding in 30 seconds instead of 100ms). The Order Service waits 30 seconds per request. Order Service threads pile up waiting. New orders cannot be processed because all threads are blocked. The Order Service effectively goes down because the Payment Service is slow. This is a cascading failure.

**Per-hop timeout budgets:**

If the total request budget from client to response is 2 seconds, each downstream call gets a fraction:

- Inventory check: 200ms timeout
- Pricing calculation: 200ms timeout
- Payment authorization: 500ms timeout (external provider, slower)
- Total overhead (serialization, network, processing): 300ms
- Remaining buffer: 800ms

If the Inventory Service does not respond within 200ms, fail immediately and return an error rather than consuming 1.8 seconds of remaining budget waiting.

**Timeout guidelines:**

- P99 latency of the downstream service is a good starting point for the timeout value
- Set timeouts on every outgoing call, including database calls and cache calls
- Log when timeouts are hit so you can detect degraded services
- Never use “no timeout” or “infinite timeout” as a default

**Interview Relevance:** Mentioning timeouts proactively when describing service communication shows the interviewer you understand production realities, not just happy-path design.

---

### **SLIDE 56: Retry Strategy and Idempotency**

When a call fails or times out, the natural instinct is to retry. But retrying incorrectly can make a bad situation catastrophically worse.

**When retrying is safe:**

A retry is safe only when the operation is idempotent, meaning executing it twice produces the same result as executing it once. Reading data is always idempotent. Creating a resource with a client-generated ID is idempotent (creating order “ord_123” twice results in one order). Incrementing a counter is not idempotent (incrementing twice gives a different result than incrementing once).

**Idempotency keys:**

For operations that are not naturally idempotent, attach a unique idempotency key to each request. The server stores the key and the result. If the same key arrives again, the server returns the stored result without re-executing.

```python
def authorize_payment(idempotency_key, order_id, amount):
    existing_result = db.get_by_idempotency_key(idempotency_key)
    if existing_result:
        return existing_result

    result = payment_gateway.charge(order_id, amount)
    db.save_with_idempotency_key(idempotency_key, result)
    return result
```

**Retry with exponential backoff and jitter:**

If all callers retry after exactly 1 second, the failing service gets hit with a synchronized wave of retries. Add exponential backoff (wait 1s, 2s, 4s, 8s) and random jitter (add a random 0-500ms) so retries are spread out.

**Retry limits:** Set a maximum number of retries (typically 2-3). After that, fail and let the caller handle the error. Unlimited retries can overload a recovering service.

**Interview Relevance:** Explaining idempotency keys and exponential backoff with jitter demonstrates practical distributed systems knowledge that goes beyond textbook concepts.

---

### **SLIDE 57: Circuit Breakers**

A circuit breaker prevents a service from repeatedly calling a downstream dependency that is failing. It is named after electrical circuit breakers that cut power to prevent damage.

**The three states:**

!image.png

**How it works:**

- **Closed (normal):** Requests pass through to the downstream service. The circuit breaker counts failures. When failures exceed a threshold (e.g., 5 failures in 10 seconds), the breaker trips to Open.
- **Open (protecting):** All requests fail immediately without calling the downstream service. This gives the failing service time to recover and prevents the caller from wasting resources on calls that will fail. After a timeout (e.g., 30 seconds), the breaker moves to Half-Open.
- **Half-Open (testing):** One test request is sent to the downstream service. If it succeeds, the breaker returns to Closed. If it fails, the breaker returns to Open for another timeout period.

**Why this matters:**

Without a circuit breaker, the Order Service keeps sending requests to a failing Payment Service, waiting for timeouts on each one. With a circuit breaker, after 5 failures the Order Service stops calling the Payment Service immediately (no timeout wait), and returns a graceful error to the user: “Payment is temporarily unavailable, please try again shortly.”

**Interview Relevance:** Circuit breakers are one of the most frequently asked resilience patterns in system design interviews. Know the three states, the failure threshold, and the recovery mechanism.

---

### **SLIDE 58: Bulkheads and Backpressure**

These two patterns prevent failures from spreading across the system.

**Bulkheads:**

Named after the watertight compartments in a ship’s hull. If one compartment floods, the others stay dry. In microservices, bulkheads isolate failure domains so that a problem with one dependency does not exhaust resources needed for other operations.

**Example:** The Order Service calls both the Payment Service and the Recommendation Service. Without bulkheads, both calls share the same thread pool (100 threads). If the Recommendation Service becomes slow and 90 threads are stuck waiting on it, only 10 threads remain for payment processing. Payments slow down because of recommendations.

With bulkheads, you assign separate thread pools: 70 threads for payments (critical) and 30 for recommendations (non-critical). If recommendations become slow, only the recommendation thread pool is exhausted. Payments continue unaffected.

**Backpressure:**

When a service receives more requests than it can handle, it must signal the caller to slow down rather than accepting everything and degrading for all users.

**Backpressure mechanisms:**

- Return HTTP 429 (Too Many Requests) when the service is at capacity
- Use bounded queues that reject messages when full instead of growing unbounded until memory runs out
- Implement rate limiting per caller so one noisy client cannot starve others

**Without backpressure:** A service accepts 50,000 requests per second when it can only handle 10,000. Response times go from 50ms to 5 seconds. All 50,000 callers get a degraded experience. **With backpressure:** The service accepts 10,000 and rejects 40,000 with a clear signal to retry later. The 10,000 accepted requests get normal performance.

**Interview Relevance:** Mentioning bulkheads and backpressure in an interview shows you think about failure isolation at a granular level, beyond just circuit breakers.

---

### **SLIDE 59: Resilience Patterns Summary**

| Pattern | What It Prevents | How It Works | When To Use |
| --- | --- | --- | --- |
| Timeouts | Indefinite waiting, thread exhaustion | Set a maximum wait time for every network call | Every outgoing call, no exceptions |
| Retries with backoff | Transient failures causing permanent errors | Retry with increasing delay and random jitter | Idempotent operations only |
| Circuit breakers | Repeated calls to a failing service | Stop calling after failure threshold, test periodically | Every critical downstream dependency |
| Bulkheads | One slow dependency exhausting all resources | Isolate thread pools or connection pools per dependency | When multiple dependencies share resources |
| Backpressure | Overloaded services degrading for everyone | Reject excess requests with a clear signal to retry | Every service that can receive unbounded traffic |
| Idempotency keys | Duplicate processing from retries | Deduplicate using a unique key per request | Any non-idempotent write operation |

**How they work together:**

A well-designed service uses all of these patterns simultaneously. A request goes out with a timeout. If it fails, a retry is attempted with backoff (only if idempotent). If multiple retries fail, the circuit breaker trips. The dependency is isolated behind a bulkhead so other operations continue. The service applies backpressure if its own capacity is threatened.

**Interview Relevance:** Being able to explain how these patterns compose together, not just define each one in isolation, is what distinguishes a senior-level answer from a mid-level answer.

---

### **SLIDE 59B: Note It Down**

**Six resilience patterns to memorize:**

1. **Timeouts** - Every call, no exceptions. Use P99 latency as starting point.
2. **Retries** - Only for idempotent operations. Exponential backoff + random jitter. Max 2-3 retries.
3. **Circuit breakers** - Three states: Closed, Open, Half-Open. Trips after failure threshold.
4. **Bulkheads** - Separate resource pools per dependency. Critical paths get more resources.
5. **Backpressure** - Reject excess load with HTTP 429. Bounded queues. Rate limiting.
6. **Idempotency keys** - Client-generated unique key. Server deduplicates. Required for safe retries on writes.

**The sentence to remember:**

“Every outgoing call has a timeout, retries only if idempotent with backoff and jitter, is protected by a circuit breaker, isolated by a bulkhead, and the service applies backpressure when overloaded.”

**Interview Relevance:** If you can state this sentence naturally during an interview and then explain any pattern the interviewer asks about, you have demonstrated production-grade distributed systems knowledge.

---

### **SLIDE 60: Contract Versioning and Backward Compatibility**

In microservices, services deploy independently. This means version 2 of the Payment Service must work with version 1 of the Order Service, and vice versa. If a new deployment breaks existing consumers, you do not have independent deployment. You have a release train.

**The backward compatibility rule:**

You can add new fields. You can add new endpoints. You cannot remove fields that consumers depend on. You cannot change the type of existing fields. You cannot rename existing endpoints.

**REST versioning strategies:**

- **URI versioning:** `/api/v1/orders` and `/api/v2/orders`. Simple but requires routing logic and maintaining multiple versions.
- **Header versioning:** `Accept: application/vnd.company.v2+json`. Cleaner URLs but less visible.
- **Backward-compatible evolution (preferred):** Add new fields without removing old ones. Consumers that do not know about the new fields simply ignore them. No version number needed.

**Example of backward-compatible change:**

Original response: `{"order_id": "123", "status": "CONFIRMED", "total": 59.98}`

Updated response: `{"order_id": "123", "status": "CONFIRMED", "total": 59.98, "currency": "USD", "estimated_delivery": "2024-03-15"}`

Old consumers continue working because they ignore `currency` and `estimated_delivery`. New consumers can use the new fields. No version bump required.

**Interview Relevance:** When you say “services deploy independently” in an interview, the interviewer may ask “how do you ensure one service’s deployment does not break another?” Backward-compatible API evolution is the answer.

---

### **SLIDE 61: Schema Evolution**

For services communicating via gRPC (protobuf) or event streams (Avro, JSON Schema), the schema defines the contract. Schema evolution rules determine which changes are safe to make without breaking consumers.

**Protobuf evolution rules:**

- **Safe:** Add a new optional field (assign a new field number). Existing consumers ignore the new field.
- **Safe:** Deprecate a field (stop populating it, but do not reuse the field number). Existing consumers that still read it get the default value.
- **Unsafe:** Remove a field and reuse its number. A consumer compiled against the old schema interprets the new field’s data as the old field’s type.
- **Unsafe:** Change a field’s type (e.g., int32 to string). Binary decoding breaks.

**Avro evolution (commonly used with Kafka):**

Avro uses a schema registry. Producers register their schema. Consumers fetch the schema and can handle both old and new formats. Avro supports three compatibility modes:

- **Backward compatible:** New schema can read data written with the old schema. You can add fields with defaults.
- **Forward compatible:** Old schema can read data written with the new schema. You can remove fields with defaults.
- **Full compatible:** Both backward and forward. The safest but most restrictive.

**The practical rule:** Always add fields with default values. Never remove fields in the same release that consumers depend on. Deprecate first, then remove after all consumers have migrated.

**Interview Tip:** If you propose gRPC or Kafka in an interview, the interviewer may ask “how do you handle schema changes?” Knowing the evolution rules for protobuf or Avro shows you have thought about the long-term maintainability of the system, not just the initial design.

**Interview Relevance:** Schema evolution is the mechanism that makes independent deployment possible when services share contracts. Understanding it separates theory from practice.

---

### **SLIDE 62: Consumer-Driven Contract Testing**

Contract testing verifies that a service’s API continues to work for its consumers after a change. It catches breaking changes before deployment, not after.

**The problem it solves:**

The Payment Service team adds a new field and accidentally renames `transaction_id` to `txn_id`. Unit tests pass because they test the Payment Service in isolation. Integration tests in staging catch the break, but only after deployment. The Order Service, which depends on `transaction_id`, is now broken in staging.

**How consumer-driven contract testing works:**

1. Each consumer (Order Service, Refund Service) writes a contract describing what it expects from the Payment Service API: “I send a POST to /payments with these fields, and I expect a response containing transaction_id and status.”
2. These contracts are shared with the Payment Service team.
3. The Payment Service runs all consumer contracts as part of its CI pipeline.
4. If any contract fails, the build fails, and the team knows which consumers would break before deploying.

**The key insight:** The consumers define the contract, not the provider. This ensures that the provider never accidentally breaks a field or endpoint that a consumer depends on. Changes that no consumer cares about (adding a new field, adding a new endpoint) pass all contracts automatically.

**Tools:** Pact is the most widely used consumer-driven contract testing framework. It supports REST, gRPC, and messaging contracts.

**Interview Relevance:** Mentioning contract testing in an interview shows you think about the lifecycle of service communication, not just the initial implementation.

---

### **SLIDE 63: Correlation IDs and Request Tracing**

In a monolith, debugging is straightforward. One request, one process, one log file, one stack trace. In microservices, a single user request can touch 5-10 services. Without a way to connect the logs from all those services, debugging is nearly impossible.

**How correlation IDs work:**

!image.png

The API Gateway generates a unique correlation ID (e.g., a UUID) for every incoming request. This ID is passed to every downstream service in the request headers. Every service includes the correlation ID in all of its log entries for that request.

**What this enables:**

When a user reports “my order failed,” you search your log aggregation system for that correlation ID. You instantly see every log entry from every service that was involved in that request, in chronological order. You can trace exactly where the failure occurred.

**Implementation requirements:**

- The API Gateway generates the correlation ID (or accepts one from the client for mobile debugging)
- Every service reads the correlation ID from incoming headers and includes it in all outgoing calls
- Every log entry includes the correlation ID as a structured field
- A log aggregation system (like the ELK stack or Datadog) lets you search by correlation ID

**Interview Relevance:** When an interviewer asks “how would you debug a problem in this microservices system?”, correlation IDs and distributed tracing are the expected answer. Without them, you are guessing.

---

### **SLIDE 63B: Interview Practice**

**Question:**

An interviewer presents this scenario: “We have 8 microservices. A user reports that their order was placed but they never received a confirmation email. The order exists in the database. How would you debug this?”

Take 60 seconds to think about your debugging approach before reading below.

**Weak Answer:**

“I would check the logs of the Notification Service to see if it tried to send the email.”

**Why this is weak:** It jumps to one service without understanding the full request path. Maybe the Order Service never published the order confirmation event. Maybe the event was published but the message queue lost it. Maybe the Notification Service consumed the event but the email provider was down. Checking one service’s logs in isolation does not tell you where in the chain the failure occurred.

**Strong Answer:**

“First, I would get the order ID from the user’s report and look up the correlation ID for that request in our logs. Using the correlation ID, I would trace the request through every service it touched.

I would check: did the Order Service successfully create the order? The user says yes and the order exists, so this step succeeded. Did the Order Service publish an OrderConfirmed event? I would check the Order Service logs for the correlation ID and look for an event publication log entry. If the event was published, I would check the event stream or message queue to verify it was persisted. Then I would check the Notification Service consumer logs to see if it received the event. If it did, I would check whether it called the email provider and what response it got.

This trace tells me exactly where the chain broke: was it the event not being published, the event not being delivered, the Notification Service failing to process it, or the email provider rejecting the send. Each scenario has a different fix.”

**What makes this strong:** The candidate uses correlation IDs and traces the full request path systematically, checking each link in the chain rather than guessing which service failed.

---

### **SLIDE 64: Communication Pattern Decision Framework**

Use this framework to choose the right communication pattern for each interaction in your system.

**Step 1: Does the caller need the response to continue?**

- Yes, the response is required for the next step (e.g., “I need to know if payment was authorized before confirming the order”) → Synchronous
- No, the work can happen independently (e.g., “send a confirmation email whenever you get to it”) → Asynchronous

**Step 2: If synchronous, which protocol?**

- External clients (browsers, mobile, third parties) → REST
- Internal service-to-service with tight latency budget → gRPC
- Client needs flexible field selection across multiple backend services → GraphQL at the edge

**Step 3: If asynchronous, queue or stream?**

- Only one consumer should process each message (task distribution, background jobs) → Message queue
- Multiple consumers need the same event (broadcasting, event-driven architecture) → Event stream
- You need event replay capability (rebuilding state, onboarding new consumers) → Event stream

**Step 4: What resilience patterns does this call need?**

- Every synchronous call: timeout + circuit breaker + bulkhead
- Every retryable call: exponential backoff + jitter + idempotency key
- Every service: backpressure when overloaded

**Interview Tip:** Walking through this decision framework on the whiteboard, choosing different patterns for different interactions and explaining why, is one of the most effective ways to demonstrate communication design skill in an interview.

**Interview Relevance:** The framework itself is less important than the reasoning behind each choice. Interviewers want to see that you evaluate coupling, latency, failure modes, and consistency for each interaction, not apply one pattern everywhere.

---

### **SLIDE 65: Section Summary and Interview Tips**

**Section 4 Summary:**

| Concept | Key Takeaway |
| --- | --- |
| Synchronous vs asynchronous | Sync when you need the response to continue. Async when you do not. |
| REST | Default for public APIs. Simple, widely compatible. |
| gRPC | Internal calls with tight latency budgets. Binary, typed, fast. |
| GraphQL | Edge aggregation layer for clients needing flexible queries. Not for service-to-service. |
| Message queues | One message to one consumer. Task distribution and load leveling. |
| Event streams | One event to many consumers. Event history and replay. |
| Commands vs events | Commands couple sender to receiver. Events decouple them. |
| Timeouts | Every call, no exceptions. |
| Retries | Only idempotent operations. Backoff + jitter. Max 2-3 attempts. |
| Circuit breakers | Three states: Closed, Open, Half-Open. Prevent cascading failure. |
| Bulkheads | Isolate resource pools per dependency. |
| Backpressure | Reject excess load. Bounded queues. Rate limiting. |
| Contract versioning | Add fields, never remove. Backward compatibility enables independent deployment. |
| Correlation IDs | Trace one request across all services using a single unique identifier. |

**Common Interview Mistakes:**

| Mistake | Why It Hurts You | Stronger Approach |
| --- | --- | --- |
| “Everything uses REST” | Ignores different requirements for external vs internal communication | “REST for public APIs, gRPC for internal low-latency calls, events for async workflows” |
| No timeouts mentioned | Implies no production experience with distributed systems | Proactively mention timeout budgets for every synchronous call |
| Retrying non-idempotent operations | Causes duplicate payments, duplicate orders, data corruption | “Retries only on idempotent operations, and we use idempotency keys for writes” |
| Synchronous calls for non-critical work | Creates unnecessary coupling and latency on the critical path | “Notifications are async. The order confirmation does not wait for the email to send.” |
| No debugging strategy | Cannot explain how to troubleshoot failures across services | “Every request gets a correlation ID, and we trace through all services using log aggregation” |

**Key Numbers to Remember:**

| Metric | Value |
| --- | --- |
| REST JSON vs gRPC protobuf payload size | Protobuf is 5-10x smaller |
| Typical timeout starting point | P99 latency of downstream service |
| Retry attempts (max) | 2-3 with exponential backoff |
| Circuit breaker failure threshold | Typically 5-10 failures in a 10-second window |
| Circuit breaker recovery timeout | Typically 15-60 seconds |
| Kafka throughput | Millions of events/second with partitioning |

**How This Connects to Other Sections:**

Section 5 (Data Ownership) builds on the event-driven patterns from this section. When each service owns its data, cross-service data access often happens through event consumption and materialized views, not synchronous API calls. Section 6 (Sagas and Consistency) uses both commands and events to coordinate multi-step workflows across services. The orchestration pattern uses commands. The choreography pattern uses events. The choice between them maps directly to the coupling trade-offs covered in this section. Section 8 (Reliability and Evolution) expands the resilience patterns from this section into system-level failure isolation and observability.

---

# Section 5: Data Ownership and Read Models

### **SLIDE 66: Section Intro**

**Section 5: Data Ownership and Read Models**

This is where microservices get genuinely hard. Most engineers can split a system into services and pick communication protocols. Far fewer can answer the question that follows: when each service owns its own database, how do you handle the data that multiple services need?

In a monolith, you join tables. In microservices, those tables live in different databases owned by different services. You cannot join across service boundaries. Every pattern in this section exists to solve that fundamental problem.

**What We Will Cover:**

1. The database-per-service principle and why it is non-negotiable
2. The shared database anti-pattern and exactly how it breaks independence
3. Cross-service reads: API composition and materialized views
4. Cross-service writes: events and workflows
5. Data duplication as a deliberate trade-off
6. Eventual consistency and the read-your-writes problem
7. CQRS: separating write models from read models
8. Reference data ownership, search indexing, cache ownership, and reporting
9. Schema migration strategies across independently deployed services

**Where This Appears in Real Systems:**

In a platform like Amazon, when you view a product page, the data comes from at least five different services: catalog (product details), inventory (stock status), pricing (current price), reviews (ratings), and recommendations (similar products). None of these services share a database. The product page is assembled from data owned by independent services, each using a different strategy to make its data available.

---

### **SLIDE 67: Database per Service**

Every microservice must own its own database. This is the most important data rule in microservices and the one that creates the most practical challenges.

!image.png

**What “owns its own database” means:**

- The Order Service is the only service that can read from or write to the Order database
- No other service has credentials to connect to the Order database
- No other service knows the Order database schema (table names, column types, indexes)
- If the Payment Service needs order data, it calls the Order Service API. It does not run SQL against the Order database.

**Why this rule exists:**

Without database-per-service, you cannot deploy services independently. If the Order Service and Payment Service both query the `orders` table, a schema migration (renaming a column, changing a type, adding a constraint) requires coordinating both services. That coordination is a deployment dependency. Deployment dependencies are what microservices exist to eliminate.

**The different database options:**

Each service can choose the database technology that best fits its needs. The Order Service might use PostgreSQL for relational data with ACID transactions. The Catalog Service might use Elasticsearch for full-text search. The Recommendation Service might use a graph database. The Session Service might use Redis. This technology independence is a direct benefit of database-per-service.

**Interview Relevance:** Drawing separate databases for each service on your whiteboard diagram is the first signal to an interviewer that you understand microservices data ownership. Drawing multiple services pointing to one database immediately raises concerns.

---

### **SLIDE 68: The Shared Database Anti-Pattern**

The shared database anti-pattern is the single most common architectural mistake in microservices. It occurs when multiple services read from or write to the same database, even if they access different tables.

**Why teams fall into this pattern:**

- “It is easier to just query the data directly than to build an API”
- “We will separate the databases later when we have time”
- “We only read from their tables, we do not write, so it is fine”
- “It is the same PostgreSQL instance, but different schemas, so it is separated”

Every one of these justifications leads to the same outcome: coupled services that cannot deploy independently.

**The three levels of shared database coupling:**

1. **Same table, same schema** - Multiple services read and write the same tables. This is the worst case. Any schema change is a cross-team coordination nightmare.
2. **Same database, different tables** - Services use their own tables but in the same database instance. A database outage or resource contention affects all services simultaneously. Schema migrations lock the database for all services.
3. **Same instance, different schemas** - Better isolation, but still shares compute resources, connection pools, and availability. A runaway query from one service can starve another service’s connections.

**Interview Relevance:** If an interviewer describes a system where services share a database and asks “what would you change?”, this is the first thing to fix. Separating databases is the foundation that every other microservices benefit depends on.

---

### **SLIDE 68B: Shared Database, How It Breaks Independent Deployment**

**Scenario:** An e-commerce platform has an Order Service and a Reporting Service. Both query the `orders` table in the same PostgreSQL database.

**The breaking change:**

The Order Service team needs to split the `shipping_address` column (a single text field) into separate columns: `street`, `city`, `state`, `zip_code`, `country`. This improves address validation and shipping cost calculation.

**What happens with a shared database:**

1. The Order Service team writes a migration to add the new columns and populate them from the old `shipping_address` field
2. They cannot drop the old `shipping_address` column because the Reporting Service reads it
3. They ask the Reporting Service team to update their queries first
4. The Reporting Service team is in the middle of a different project and cannot prioritize this for 3 weeks
5. The Order Service team waits 3 weeks to ship their feature, or maintains both the old and new columns indefinitely

**What happens with database-per-service:**

1. The Order Service team changes their internal schema however they want
2. The Order Service API response still includes `shipping_address` for backward compatibility (Section 4, contract versioning)
3. The Reporting Service continues consuming the API. It is not affected.
4. The Order Service team ships their feature the same day

**The lesson:** Database-per-service does not just prevent technical coupling. It prevents organizational coupling. Teams can move at their own pace without waiting for other teams to adapt to internal changes.

**Interview Relevance:** This specific scenario (internal schema change blocked by another service’s direct database access) is a powerful concrete example to use in interviews when explaining why database-per-service matters.

---

### **SLIDE 69: Private Schema Ownership**

Once each service has its own database, the next question is: what data goes where?

**The ownership rule:** A service owns the data that it is the system of record for. The service that creates and modifies a piece of data owns it. Other services can have copies, but only one service is the authoritative source of truth.

**Practical examples:**

- The **User Service** owns user profiles (name, email, account status). It is where user profiles are created and modified. Other services may cache the user’s name, but the User Service is the authority.
- The **Order Service** owns orders. It is where orders are created, updated, and cancelled. The Reporting Service may have a copy of order data for analytics, but the Order Service is the source of truth.
- The **Inventory Service** owns stock levels. It is where inventory is reserved and updated. The Catalog Service may display “in stock” on the product page, but it gets that information from the Inventory Service.

**What the owning service must provide:**

- An API for other services to query the data they need (synchronous)
- Events published when the data changes, so other services can maintain their own copies (asynchronous)
- A contract that is versioned and backward compatible so consumers are not broken by internal changes

**What the owning service must protect:**

- No direct database access from other services. The database credentials are known only to the owning service.
- No exposing internal schema details in the API. The API response structure can differ from the database schema.
- No allowing other services to bypass the API with “just a quick query.” There are no exceptions.

**Interview Relevance:** When you define service boundaries in an interview, explicitly stating which service is the “system of record” for each entity shows you have thought about data ownership, not just service naming.

---

### **SLIDE 70: Cross-Service Reads with API Composition**

The most straightforward way to get data from another service is to call its API. When a single user request needs data from multiple services, you compose the response by calling each service and combining the results.

**Example: Product page in an e-commerce platform**

!image.png

The API Gateway (or a Backend-for-Frontend layer) calls four services in parallel and assembles the product page response from their individual responses.

**When API composition works well:**

- The data is needed in real-time and must be fresh (e.g., current stock level, current price)
- The number of services to call is small (2-4 calls)
- The downstream services are fast and reliable
- The call pattern is simple (parallel fan-out, not sequential chains)

**When API composition breaks down:**

- **Too many calls.** Assembling a dashboard that needs data from 10 services adds latency and fragility. One slow service delays the entire response.
- **Data needs joining.** Displaying “top 10 products by revenue” requires joining order data with product data. Neither service has both datasets.
- **High volume.** If the product page gets 100,000 requests per second and each request triggers 4 downstream calls, the downstream services receive 400,000 requests per second. The fan-out multiplies load.

When API composition breaks down, materialized views (next slide) are the alternative.

**Interview Relevance:** API composition is the simplest cross-service read pattern. Start with it in interviews and move to materialized views only when you can articulate why composition is not sufficient.

---

### **SLIDE 71: Cross-Service Reads with Materialized Views**

When API composition is too slow, too fragile, or requires joining data across services, a materialized view provides an alternative. The consuming service builds and maintains its own pre-computed copy of the data it needs.

!image.png

**How it works:**

1. The Catalog Service, Pricing Service, and Inventory Service each publish events when their data changes
2. The Product Page Service consumes all three event streams
3. It builds a denormalized read model that contains product name, current price, and stock status in a single database row
4. When a client requests the product page, the Product Page Service queries its own local database. Zero cross-service calls at read time.

**The trade-off:**

- **Reads are fast.** One local database query instead of 4 network calls.
- **Reads are resilient.** If the Catalog Service is down, the Product Page Service still serves data from its local copy.
- **Data is eventually consistent.** When the Pricing Service updates a price, there is a delay (typically milliseconds to seconds) before the Product Page Service’s read model reflects the change.
- **Storage is duplicated.** Product data exists in the Catalog Service database and in the Product Page Service’s read model.

**When to use materialized views:**

- Read volume is much higher than write volume (product pages are viewed millions of times, but prices change thousands of times)
- You need data from 3+ services combined in a single query
- Latency requirements are strict and fan-out adds too much delay
- Slight staleness (seconds) is acceptable

**Interview Relevance:** Proposing a materialized view in an interview shows you understand that microservices trade data freshness for read performance and resilience. This is a senior-level pattern.

---

### **SLIDE 71B: Interview Practice**

**Question:**

You are building a search page for an e-commerce platform. The search results need to show: product name (from Catalog Service), current price (from Pricing Service), average rating (from Reviews Service), and stock status (from Inventory Service). The search page receives 50,000 requests per second. How would you design the data access for this page?

Take 60 seconds to design your approach before reading below.

**Weak Answer:**

“The search page calls all four services for each request. We can parallelize the calls so it is fast.”

**Why this is weak:** At 50,000 requests per second, each downstream service receives 50,000 calls just from search. The Pricing Service, Inventory Service, and Reviews Service also serve their own direct traffic. The combined load is unsustainable. One slow service makes all search results slow. One failing service makes search unavailable.

**Strong Answer:**

“At 50,000 requests per second, API composition would overwhelm the downstream services and add unacceptable latency. I would use a materialized view approach.

I would build a Search Service with its own denormalized data store, likely Elasticsearch for full-text search capability. The Search Service consumes events from all four upstream services: ProductUpdated from Catalog, PriceChanged from Pricing, RatingUpdated from Reviews, and StockUpdated from Inventory. It maintains a search index where each document contains the product name, price, rating, and stock status together.

When a user searches, the Search Service queries its own local Elasticsearch index. Zero cross-service calls at query time. The search index updates within seconds of any upstream change, which is acceptable for a search page where slight staleness does not affect the user experience.

The trade-off is data duplication and eventual consistency. Product data exists in five places: Catalog DB, Pricing DB, Reviews DB, Inventory DB, and the Search index. But at 50,000 QPS, this is the only pattern that provides the performance and resilience we need.”

**What makes this strong:** The candidate explains why API composition fails at this scale, proposes a concrete alternative with the right technology (Elasticsearch), explains the event-driven update mechanism, and acknowledges the trade-offs explicitly.

---

### **SLIDE 72: Cross-Service Writes**

Reading data across services is solved by API composition or materialized views. Writing data across services is fundamentally harder because you need changes in multiple databases to be coordinated.

**The problem:**

When a customer places an order, three things must happen: the Order Service creates the order, the Inventory Service reserves the stock, and the Payment Service authorizes the charge. In a monolith, this is one database transaction. In microservices, these are three separate databases with no shared transaction.

What happens if the order is created, the inventory is reserved, but the payment authorization fails? You now have an order in the database with reserved inventory but no payment. The data is inconsistent across services.

**The approaches to cross-service writes:**

1. **Synchronous orchestration** - A central coordinator calls each service in sequence and handles failures by calling compensating actions (refund payment, release inventory, cancel order). This is the saga pattern, covered in depth in Section 6.
2. **Event-driven choreography** - Each service publishes an event when it completes its step. The next service reacts to that event. No central coordinator. Also covered in Section 6.
3. **Avoid cross-service writes entirely** - Redesign the boundaries so that the data that must change together lives in the same service. Sometimes the right answer is not a clever distributed write pattern but a better service boundary.

**The key insight from Section 3:** The aggregate boundary (Slide 27) defines what must change in one transaction. Everything inside an aggregate is a local write within one service. Everything that crosses an aggregate boundary is a cross-service write that requires sagas or events. Good aggregate design minimizes the number of cross-service writes you need.

**Interview Relevance:** When you describe a write operation that spans multiple services, the interviewer expects you to immediately address consistency. Saying “the Order Service creates the order and calls the Payment Service” without addressing failure scenarios is a gap that interviewers will probe.

---

### **SLIDE 73: Data Duplication as a Trade-Off**

In microservices, data duplication is not a mistake. It is a deliberate architectural trade-off that enables service independence. This is one of the hardest mindset shifts for engineers who come from a relational database background where normalization is a core principle.

**Why duplication is necessary:**

In a monolith, you normalize data to avoid redundancy. The product name exists in one row of the `products` table. Every query that needs the product name joins to that table. This works because everything is in one database.

In microservices, the `products` table is in the Catalog Service database. The Order Service cannot join to it. When the Order Service creates an order, it needs to store the product name and price at the time of purchase. It copies that data into its own `order_items` table. The product name now exists in two databases.

**When duplication is correct:**

- The Order Service stores “product name at time of purchase” because the product name might change later. The order record should reflect what the customer bought, not the current product name.
- The Search Service stores a denormalized copy of product data for fast querying. The source of truth remains the Catalog Service.
- The Notification Service stores the customer’s email address so it can send emails without calling the User Service for every notification.

**When duplication is a problem:**

- If two services both consider themselves the source of truth for the same data and both accept writes, you have a conflict resolution problem, not a duplication strategy.
- If duplicated data is never refreshed and becomes stale in ways that affect business correctness (e.g., the price in the order does not match what the customer was shown).

**The rule:** One service is the source of truth. Other services hold copies that are either snapshotted at a point in time (order capturing price at purchase) or kept updated through events (search index consuming price change events).

**Interview Relevance:** Saying “I would duplicate the product name in the order record because the order should capture the state at purchase time” shows the interviewer you understand why duplication exists in microservices, not just that it happens.

---

### **SLIDE 74: Eventual Consistency in Practice**

In a monolith with one database, every read sees the latest write. In microservices with separate databases and event-driven updates, there is a delay between when data changes in one service and when other services reflect that change. This is eventual consistency.

**How eventual consistency manifests:**

1. A seller updates a product price from $29.99 to $24.99 in the Catalog Service
2. The Catalog Service publishes a PriceChanged event
3. The Search Service consumes the event and updates its index
4. For a few hundred milliseconds (or a few seconds under load), the product page shows $24.99 but the search results still show $29.99

**Is this acceptable?** Usually, yes. Most users never notice a sub-second inconsistency between two different pages. The few who do will see the correct price when they refresh. This is a drastically different situation from a banking system where an account balance must be immediately consistent.

**When eventual consistency is not acceptable:**

- Financial transactions where the user’s balance must reflect a debit immediately
- Inventory checks where selling an out-of-stock item is costly (overselling)
- Security-sensitive operations where permission changes must take effect immediately

For these cases, use synchronous calls to the source-of-truth service rather than relying on an eventually consistent copy. This trades performance for correctness.

**Typical consistency delays:**

- Event stream consumption (Kafka): 10-500ms under normal load
- Materialized view update: 100ms-2 seconds depending on processing
- Search index update (Elasticsearch): 1-5 seconds (near real-time refresh)
- Cache TTL expiry: depends on configuration, commonly 30 seconds to 5 minutes

**Interview Tip:** When you propose eventual consistency in an interview, always specify which interactions can tolerate it and which cannot. “Search results can be eventually consistent, but the checkout price must come from the source of truth in real-time” shows you understand the nuances.

**Interview Relevance:** Every microservices interview eventually reaches the consistency question. Having a clear framework for when eventual consistency is acceptable versus when strong consistency is required is essential.

---

### **SLIDE 75: The Read-Your-Writes Problem**

This is the most common consistency problem that users actually notice. A user makes a change and immediately sees stale data because their read hits a replica or a cached copy that has not been updated yet.

**The classic scenario:**

1. A user updates their profile name from “John” to “Jonathan” in the User Service
2. The User Service writes to the primary database
3. The user immediately navigates to their profile page
4. The profile page reads from a replica or a cached copy that still shows “John”
5. The user thinks the update failed

**Solution: Read-your-writes consistency**

After a user writes data, route their subsequent reads to the source of truth (primary database or owning service) for a short window. Other users can continue reading from replicas or caches with no issue.

```python
def update_profile(user_id, new_name):
    primary_db.update_user(user_id, name=new_name)
    cache.set(f"recent_write:{user_id}", True, ttl=5)

def get_profile(user_id, requesting_user_id):
    if requesting_user_id == user_id and cache.get(f"recent_write:{user_id}"):
        return primary_db.get_user(user_id)
    return replica_db.get_user(user_id)
```

After a user updates their profile, a flag is set in the cache for 5 seconds. During those 5 seconds, that specific user’s profile reads are routed to the primary database. Every other user reads from the replica as usual. After 5 seconds, the replica has caught up, and the flag expires.

**The key insight:** You only need strong consistency for the user who just wrote. Every other user can tolerate eventual consistency and never notice. This gives you the performance benefits of replicas and caches for 99.9% of reads while guaranteeing that the writing user sees their own changes immediately.

**Interview Relevance:** Read-your-writes consistency is a specific, named pattern that interviewers expect senior candidates to know. It demonstrates understanding of consistency beyond just “strong” or “eventual.”

---

### **SLIDE 76: CQRS**

CQRS (Command Query Responsibility Segregation) separates the write model (commands) from the read model (queries) into different data stores optimized for their respective access patterns.

!image.png

**Why separate read and write models?**

In many systems, the ideal data structure for writing is different from the ideal structure for reading. The write side needs normalization, constraints, and ACID transactions to maintain correctness. The read side needs denormalization, pre-computed aggregations, and fast lookups to serve queries quickly.

**Example: Order history page**

The write model stores orders in normalized tables: `orders`, `order_items`, `shipping_addresses`, `payment_records`. Writes enforce referential integrity and business rules.

The read model stores a denormalized order summary: order ID, customer name, total amount, item count, order status, and last updated timestamp in a single document. Reads serve the order history page with one query instead of joining four tables.

**When CQRS is justified:**

- Read and write volumes differ by 10x or more (far more reads than writes)
- The read model needs a different database technology than the write model (e.g., writes to PostgreSQL, reads from Elasticsearch)
- The read model needs different data shapes for different consumers (mobile app needs a summary, admin dashboard needs full details)

**When CQRS is overkill:**

- Read and write patterns are similar (basic CRUD with similar volume)
- The system is small enough that one database handles both efficiently
- The added complexity of maintaining two models and event synchronization is not justified by the performance gain

**Interview Relevance:** CQRS is a pattern that interviewers expect you to know but also expect you to justify. Proposing CQRS for a simple CRUD service signals over-engineering. Proposing it for a high-read system with complex query needs signals strong design sense.

---

### **SLIDE 76B: CQRS, Worked Example**

**Scenario:** A food delivery platform like DoorDash needs to display a restaurant’s menu page and also allow restaurant owners to update their menus.

**Write Side (Restaurant Management Service):**

Restaurant owners update menus through an admin panel. The write model is normalized and relational:

```sql
CREATE TABLE restaurants (
    restaurant_id UUID PRIMARY KEY,
    name VARCHAR(255),
    cuisine_type VARCHAR(100),
    address TEXT
);

CREATE TABLE menu_items (
    item_id UUID PRIMARY KEY,
    restaurant_id UUID REFERENCES restaurants(restaurant_id),
    name VARCHAR(255),
    description TEXT,
    price DECIMAL(10,2),
    category VARCHAR(100),
    available BOOLEAN DEFAULT TRUE
);
```

Write volume: a few hundred menu updates per day across all restaurants. Writes need validation (price must be positive, category must be valid) and transactional consistency (adding an item and updating the restaurant’s item count must be atomic).

**Read Side (Menu Display Service):**

Customers browse menus on the app. The read model is denormalized and optimized for fast retrieval:

```json
{
  "restaurant_id": "rest_123",
  "name": "Mario's Pizza",
  "cuisine": "Italian",
  "menu": {
    "Appetizers": [
      { "name": "Bruschetta", "price": 8.99, "available": true },
      { "name": "Calamari", "price": 12.99, "available": false }
    ],
    "Mains": [{ "name": "Margherita Pizza", "price": 14.99, "available": true }]
  },
  "total_available_items": 15,
  "average_price": 16.5
}
```

Read volume: millions of menu views per day. Each read returns the full menu with pre-computed aggregations (total items, average price) in one query. No joins needed.

**The sync mechanism:** When a restaurant owner updates a menu item, the write service publishes a MenuItemUpdated event. The read service consumes the event and rebuilds the denormalized menu document. Typical lag: under 1 second.

**Interview Relevance:** Walking through a concrete CQRS example with actual schemas and data shapes demonstrates that you can implement the pattern, not just name it.

---

### **SLIDE 76C: Interview Practice**

**Question:**

An interviewer asks: “Your e-commerce platform has a product catalog with 10 million products. The catalog page gets 200,000 reads per second. Product managers update about 5,000 products per day. How would you design the data layer?”

Take 60 seconds to design your approach before reading below.

**Weak Answer:**

“I would use a PostgreSQL database with read replicas to handle the read volume. The replicas can serve the 200,000 reads per second.”

**Why this is weak:** PostgreSQL replicas can scale reads, but each read still requires joining product, pricing, and inventory tables. At 200,000 QPS with multi-table joins, even replicas will struggle. The answer also does not address the fact that the read model (what customers see) is very different from the write model (what product managers edit).

**Strong Answer:**

“The read-to-write ratio here is extreme: 200,000 reads per second versus about 0.06 writes per second (5,000 per day). This is a textbook case for CQRS.

For the write side, I would use PostgreSQL with a normalized schema. Product managers update products through an admin API. Writes enforce business rules (valid categories, positive prices, required images) and are transactionally consistent. At 5,000 writes per day, a single PostgreSQL instance handles this easily.

For the read side, I would use a denormalized data store optimized for fast reads. Each product document contains everything needed to render the catalog page: product name, description, main image URL, current price, stock status, and average rating. At 200,000 QPS, I would use a combination of Elasticsearch for search queries and Redis for individual product lookups, both populated from events published by the write side.

When a product manager updates a product, the write service publishes a ProductUpdated event. The read services consume it and update their denormalized stores. Typical lag is under 1 second, which is completely acceptable for catalog browsing.

This gives us fast reads with no joins, a clean write model with proper validation, and independent scaling of the read and write sides.”

**What makes this strong:** The candidate calculates the read-to-write ratio, justifies CQRS with specific numbers, chooses appropriate technologies for each side, and explains the synchronization mechanism.

---

### **SLIDE 77: Materialized Views and Denormalization**

Materialized views are pre-computed query results stored in a read-optimized format. In microservices, they are the primary tool for making cross-service data available without runtime API calls.

**How materialized views work in microservices:**

The source services (Catalog, Pricing, Inventory) publish events when their data changes. A consuming service listens to all relevant event streams and builds a combined, denormalized view in its own database. The view is “materialized” because it is physically stored, not computed on-the-fly.

**Denormalization across services:**

In a monolith, the order history query joins the `orders`, `products`, `users`, and `payments` tables. In microservices, each of those tables belongs to a different service. The Order History view denormalizes all of this into a single document per order:

- Order ID, status, created date (from Order Service events)
- Product names and prices at time of purchase (snapshotted when the order was created)
- Customer name and email (from User Service events)
- Payment status and method (from Payment Service events)

One query returns everything needed to display the order history. No cross-service calls.

**Keeping materialized views fresh:**

- The view rebuilds automatically as events arrive
- If an event is missed or processing fails, the consumer replays from the last known offset in the event stream
- Periodic full rebuilds can be scheduled to catch any drift

**The cost of materialized views:**

- Storage duplication (the same data exists in the source service and the view)
- Processing overhead (consuming events, updating the view)
- Eventual consistency (the view lags behind the source by milliseconds to seconds)
- Increased system complexity (event schemas must be maintained, consumers must handle out-of-order events)

**Interview Relevance:** Materialized views are the go-to solution for cross-service reads at scale. Knowing when to use them and how they are kept fresh is expected in senior-level interviews.

---

### **SLIDE 78: Ownership of Reference Data**

Reference data is data that many services need but only one service should own. Product names, user profiles, currency codes, country lists, and pricing tiers are all reference data. Handling it correctly prevents the most common data ownership conflicts.

**Pattern 1: Snapshot at event time**

The Order Service needs the product name and price when creating an order. It captures a snapshot of this data at the moment of purchase and stores it in its own database. The snapshot never changes, even if the product name or price changes later. The Order Service does not consume ongoing product events.

This is appropriate for data that must reflect a point in time (what the customer actually bought and paid).

**Pattern 2: Event-driven local cache**

The Notification Service needs the customer’s email address to send emails. It consumes UserUpdated events from the User Service and maintains a local lookup table of user IDs to email addresses. When it needs to send an email, it queries its local table instead of calling the User Service.

This is appropriate for reference data that should stay current but does not need real-time freshness.

**Pattern 3: Synchronous API call**

The Checkout Service needs the customer’s current default payment method right now, at checkout time. It calls the Payment Service API synchronously because this data must be fresh (the customer may have just changed their payment method) and stale data would cause a payment failure.

This is appropriate for data where staleness causes business errors.

**Choosing between patterns:**

- Does the data need to reflect a point in time? → Snapshot
- Does the data need to be roughly current but can tolerate seconds of staleness? → Event-driven local cache
- Does staleness cause business failures? → Synchronous API call

**Interview Relevance:** Reference data ownership is a practical problem that comes up in every real microservices system. Having three named patterns with clear selection criteria makes your interview answers specific and actionable.

---

### **SLIDE 79: Search Indexing as a Derived Read Model**

Search is almost always a cross-service concern. A search query like “red shoes under $50 in stock near me” touches data from the Catalog Service (product attributes), Pricing Service (price), Inventory Service (stock levels), and Location Service (warehouse proximity). No single service owns all of this data.

!image.png

**The pattern:**

A Search Service owns a search index (typically Elasticsearch or OpenSearch). An indexer process consumes events from all relevant upstream services and maintains the search index. Each document in the index is a denormalized product record containing all searchable attributes.

**This is a materialized view specifically optimized for search.** It supports full-text search, faceted filtering, relevance ranking, and geospatial queries that no individual service’s database can handle efficiently.

**Who owns the Search Service?**

The Search Service owns the index, the indexer, and the search API. It does not own the source data. The Catalog Service is still the authority for product names. The Pricing Service is still the authority for prices. The Search Service is a derived read model that assembles and indexes data from multiple sources.

**Consistency considerations:**

A price change takes 1-5 seconds to appear in search results. This is acceptable because search results show approximate information. The actual price is always confirmed from the source of truth (Pricing Service) at checkout time.

**Interview Relevance:** Search is one of the most common features in system design interviews. Explaining how the search index is populated from multiple services via events demonstrates a deep understanding of cross-service data patterns.

---

### **SLIDE 80: Cache Ownership and Invalidation**

Caching in microservices raises a specific ownership question: which service is responsible for caching data, and which service is responsible for invalidating the cache when the data changes?

**The ownership rule:** The service that owns the data owns the cache invalidation logic. Other services that cache a copy of the data should follow the owning service’s signals (events, TTLs) to invalidate their caches.

**Pattern 1: Source-owned cache**

The Catalog Service caches its own product data in Redis. When a product is updated, the Catalog Service invalidates or updates the cache entry. Consumers of the Catalog API automatically get fresh data because the source manages the cache.

This is the simplest pattern and should be the default.

**Pattern 2: Consumer-owned cache with TTL**

The Order Service caches product names locally to avoid calling the Catalog Service for every order display. The cache has a 60-second TTL. After 60 seconds, the next request fetches fresh data from the Catalog API.

This is appropriate when slight staleness is acceptable and the consuming service wants to reduce its dependency on the source service.

**Pattern 3: Consumer-owned cache with event invalidation**

The Search Service caches product data and consumes ProductUpdated events from the Catalog Service. When an event arrives, the Search Service invalidates or updates the relevant cache entry immediately.

This provides fresher data than TTL-based caching at the cost of consuming and processing the event stream.

**The cardinal sin of cache ownership:** Service A caches data from Service B, and Service B has no idea. When Service B’s data changes, Service A’s cache remains stale with no invalidation mechanism. This causes bugs that are extremely difficult to diagnose because the stale data looks correct until you compare it with the source.

**Interview Tip:** When you describe caching in an interview, always state who owns the invalidation. “The Catalog Service caches product data and invalidates on write” is a complete answer. “We cache product data” leaves the interviewer wondering who invalidates it.

**Interview Relevance:** Cache invalidation is famously one of the hardest problems in computer science. In microservices, it is even harder because the cache and the data source live in different services. Showing you have a clear ownership model for invalidation is a strong signal.

---

### **SLIDE 81: Reporting and Analytics**

Operational databases (the databases each microservice owns) are optimized for transactional workloads: fast reads and writes for individual records. Analytical workloads (aggregations, trends, cross-domain reports) have very different requirements and should not run against operational databases.

**The problem:**

A business analyst wants to run: “Show me total revenue by product category for the last 90 days.” This query touches order data (Order Service), product categories (Catalog Service), and payment amounts (Payment Service). Running this aggregation across three microservice APIs would be slow, resource-intensive, and could degrade the operational services for real users.

**The solution: Operational database vs analytical store**

Each microservice publishes domain events to an event stream. An ETL (Extract, Transform, Load) pipeline or stream processor consumes these events and loads them into a centralized analytical store: a data warehouse (like BigQuery, Snowflake, or Redshift) or a data lake.

The analytical store contains copies of data from all services in a format optimized for aggregation, joining, and trend analysis. Business analysts query the analytical store directly. They never touch the operational databases.

**How this preserves service independence:**

- Services only publish events. They do not need to support reporting queries.
- The reporting schema can differ from any service’s internal schema.
- Adding new reporting dimensions does not require changing any service.
- Heavy analytical queries do not affect operational service performance.

**Interview Relevance:** When an interviewer asks about reporting or analytics in a microservices system, the expected answer is a separate analytical store fed by events. Running analytical queries against operational microservice databases is a design mistake.

---

### **SLIDE 81B: Note It Down**

**Seven data ownership patterns to remember:**

1. **Database per service** - Non-negotiable. Each service owns its data store. No shared databases.
2. **API composition** - Call multiple services and combine results. Best for real-time data from 2-4 services.
3. **Materialized views** - Pre-compute cross-service data from events. Best for high-read, cross-service queries.
4. **CQRS** - Separate write model from read model. Best when read/write patterns differ dramatically.
5. **Event-driven local cache** - Consume events to maintain a local copy. Best for reference data that should stay current.
6. **Snapshot at event time** - Capture data at the moment of use. Best for point-in-time records (order capturing price at purchase).
7. **Analytical store** - Centralized warehouse fed by events. Best for reporting and cross-domain analytics.

**The selection rule:** Start with the simplest pattern (API composition). Move to more complex patterns (materialized views, CQRS) only when you can articulate a specific problem that the simpler pattern does not solve (too many calls, too much latency, read/write ratio too skewed).

**Interview Relevance:** Being able to name these patterns and explain when each is appropriate puts you ahead of most candidates, who only know “each service has its own database” without knowing how to handle the cross-service data problems that creates.

---

### **SLIDE 82: Schema Migration Across Services**

In a monolith, a schema migration is one coordinated change. In microservices, schema changes in one service must not break other services that consume its API or events.

**The expand-contract pattern:**

This is the standard approach for making schema changes safely across independently deployed services.

**Phase 1: Expand**

Add the new field alongside the old field. Populate both. The API and events include both the old and new fields. Existing consumers continue using the old field. New consumers can start using the new field.

**Phase 2: Migrate**

All consumers switch to using the new field. This happens service by service, each deploying at their own pace. No coordination required.

**Phase 3: Contract**

Once all consumers have migrated, remove the old field from the API and events. Stop populating it. Deploy the owning service.

**Example: Renaming “shipping_address” to structured address fields**

Phase 1: The Order Service API returns both `shipping_address` (old string field) and `street`, `city`, `state`, `zip`, `country` (new structured fields). Phase 2: The Reporting Service, the Shipping Service, and the Analytics pipeline each update to use the new structured fields, deploying independently. Phase 3: After confirming all consumers have migrated (contract tests pass without the old field), the Order Service removes `shipping_address` from the API.

**Timeline:** This process typically takes 2-4 weeks, not because the technical work is complex, but because each consuming team deploys on their own schedule.

**Interview Relevance:** Describing the expand-contract pattern shows the interviewer you understand how schema changes work in a system where services deploy independently. It connects directly to the backward compatibility principles from Section 4.

---

### **SLIDE 82B: Interview Practice**

**Question:**

You have designed a microservices system for an e-commerce platform. The interviewer asks: “The product detail page needs data from five different services. How do you avoid making five API calls for every page view? And what happens to the product page if one of those services goes down?”

Take 60 seconds to think about your answer before reading below.

**Weak Answer:**

“I would call all five services in parallel. Parallel calls are fast. If one service is down, I would show a degraded page without that data.”

**Why this is weak:** Parallel calls help latency but not reliability. At 100,000 page views per second, each service receives 100,000 calls from the product page alone. The answer also does not address how the “degraded page” works in practice.

**Strong Answer:**

“I would not make five API calls per page view. Instead, I would build a Product Page Service that maintains a materialized view combining data from all five upstream services.

The Product Page Service consumes events from the Catalog Service (product details), Pricing Service (current price), Inventory Service (stock status), Reviews Service (ratings), and Promotion Service (active discounts). It stores a denormalized product document in its own database with all five data points in a single record.

When a client requests a product page, the Product Page Service queries its own local database. One query, zero cross-service calls. At 100,000 QPS, this is the only pattern that scales.

For resilience: if the Catalog Service goes down, the Product Page Service is unaffected because it reads from its own database. The only impact is that product updates stop flowing until the Catalog Service recovers. But existing products continue to be served with the last known data, which is perfectly fine for a product page.

The trade-off is eventual consistency. A price change takes 1-2 seconds to appear on the product page. This is acceptable because the actual price is always confirmed from the Pricing Service at checkout time, not from the product page cache.”

**What makes this strong:** The candidate proposes a concrete architecture (materialized view), explains the event-driven update mechanism, addresses both performance and resilience, and acknowledges the consistency trade-off with a mitigation strategy (confirming price at checkout).

---

### **SLIDE 83: Data Patterns Decision Framework**

When you need data from another service, use this decision tree.

**Question 1: Is this a read or a write?**

- Read → Continue to Question 2
- Write across services → Saga pattern (Section 6)

**Question 2: How many services are involved?**

- 1-2 services, low volume → API composition (call their APIs directly)
- 3+ services, or high volume → Materialized view or CQRS

**Question 3: Does the data need to be real-time fresh?**

- Yes, staleness causes business errors (e.g., current balance, inventory at checkout) → Synchronous API call to the source of truth
- No, seconds of staleness is acceptable (e.g., search results, product catalog, dashboards) → Materialized view or event-driven cache

**Question 4: Are read and write patterns dramatically different?**

- Yes (reads 100x-1000x more than writes, or reads need a different data shape) → CQRS
- No (similar read/write volume and shape) → Single model is sufficient

**Question 5: Is this for reporting or analytics?**

- Yes → Analytical store (data warehouse) fed by events. Never query operational databases for analytics.

**Interview Relevance:** Walking through this decision tree in an interview, arriving at a pattern through reasoning rather than stating it from memory, demonstrates the kind of systematic thinking interviewers want to see.

---

### **SLIDE 84: Section Summary and Interview Tips**

**Section 5 Summary:**

| Concept | Key Takeaway |
| --- | --- |
| Database per service | Non-negotiable. Each service owns its data store exclusively. |
| Shared database | Anti-pattern. Prevents independent deployment and schema evolution. |
| API composition | Call multiple services and combine. Simple but does not scale to high volume or many services. |
| Materialized views | Pre-compute cross-service data from events. High read performance. Eventually consistent. |
| Data duplication | Deliberate trade-off for independence. One source of truth, multiple copies. |
| Eventual consistency | Default in microservices. Acceptable for most reads. Not acceptable for financial transactions. |
| Read-your-writes | Route the writing user’s reads to the primary for a short window after their write. |
| CQRS | Separate write and read models. Justified when read/write patterns differ dramatically. |
| Reference data | Snapshot at event time, event-driven cache, or synchronous API call depending on freshness needs. |
| Search indexing | Derived read model fed by events from multiple services. |
| Cache ownership | The service that owns the data owns the invalidation logic. |
| Reporting | Separate analytical store fed by events. Never run analytics against operational databases. |
| Schema migration | Expand-contract pattern: add new, migrate consumers, remove old. |

**Common Interview Mistakes:**

| Mistake | Why It Hurts You | Stronger Approach |
| --- | --- | --- |
| Multiple services sharing a database | The most common and most damaging microservices anti-pattern | “Each service has its own database. Cross-service data access goes through APIs or events.” |
| Calling 5+ APIs per user request | Creates latency and fragility at scale | “I would use a materialized view so the product page queries one local database instead of five services.” |
| Proposing CQRS for simple CRUD | Over-engineering that adds complexity without benefit | “CQRS makes sense here because read volume is 1000x write volume and the read model needs Elasticsearch.” |
| Ignoring consistency trade-offs | Makes the design sound naive | “Search results are eventually consistent (acceptable), but checkout price comes from the source of truth (required).” |
| No cache invalidation strategy | Leaves a gap that interviewers will probe | “The Catalog Service owns cache invalidation and publishes events when products change.” |

**Key Numbers to Remember:**

| Metric | Value |
| --- | --- |
| Event stream lag (typical) | 10-500ms under normal load |
| Search index refresh | 1-5 seconds (near real-time) |
| Cache TTL (common range) | 30 seconds to 5 minutes |
| Read-your-writes window | 3-10 seconds after the write |
| CQRS justification threshold | Read volume 10x-1000x greater than write volume |
| API composition limit | 2-4 services before materialized views become better |

**How This Connects to Other Sections:**

Section 6 (Sagas and Consistency) addresses the cross-service write problem that this section identified but deferred. When two services both need to update their data as part of one business operation, sagas coordinate the workflow with compensating transactions for failures. Section 7 (API Gateway) builds on the API composition pattern. The gateway or BFF layer is often where API composition happens, aggregating responses from multiple backend services before returning to the client. Section 8 (Reliability) extends the eventual consistency discussion into system-level observability, showing how to monitor consistency lag and detect when materialized views fall behind.


--
--
# Section 6: Distributed Transactions, Sagas, and Consistency

## SLIDE 85: Section Intro

### Section 6: Distributed Transactions, Sagas, and Consistency

Section 5 established that each service owns its own database. This section confronts the hardest consequence of that rule: when a business operation spans multiple services, how do you keep data consistent across them without a shared transaction?

In a monolith, you wrap everything in a database transaction. If payment fails, the order is rolled back, inventory is released, and the database is consistent. In microservices, there is no shared transaction. Payment lives in one database, inventory in another, orders in a third. When payment fails after inventory has already been reserved, you need a strategy that goes far beyond `BEGIN` and `ROLLBACK`.

This section teaches that strategy. It is the highest-signal material in this course for senior-level interviews.

#### What We Will Cover:

- Why local ACID transactions do not work across service boundaries
- Two-phase commit (2PC) and why it is usually avoided
- The saga pattern: orchestration and choreography
- Compensating transactions for undoing completed steps
- The transactional outbox and inbox patterns for reliable messaging
- Idempotency, at-least-once delivery, and the exactly-once myth
- Dead-letter queues, retry strategies, and partial failure handling
- State machines for managing service lifecycles
- A complete worked example: e-commerce checkout saga with failure cases
- Human intervention for unrecoverable failures

#### Where This Appears in Real Systems:

Every time you place an order on a platform like Amazon, a saga coordinates order creation, inventory reservation, payment authorization, and shipment scheduling across independent services. When payment fails, compensating transactions release inventory. When shipping fails after payment, the system must decide between retrying, refunding, or escalating to a human operator. This is the reality of distributed consistency.

---

## SLIDE 86: Why Local ACID Is Not Enough

In a monolith, a single database transaction guarantees four properties (ACID): **Atomicity** (all or nothing), **Consistency** (valid state after every transaction), **Isolation** (concurrent transactions do not interfere), and **Durability** (committed data survives crashes).

#### The monolith checkout in one transaction:

```sql
BEGIN TRANSACTION;
  INSERT INTO orders (order_id, customer_id, total) VALUES ('ord_123', 'cust_456', 59.98);
  UPDATE inventory SET reserved = reserved + 1 WHERE product_id = 'prod_789';
  INSERT INTO payments (payment_id, order_id, amount, status) VALUES ('pay_001', 'ord_123', 59.98, 'AUTHORIZED');
COMMIT;
```

If any statement fails, the entire transaction rolls back. The database is never in a state where an order exists without a payment, or where inventory is reserved without an order.

#### Why this breaks in microservices:

The orders table is in the Order Service database. The inventory table is in the Inventory Service database. The payments table is in the Payment Service database. There is no single `BEGIN TRANSACTION` that spans three separate databases on three separate servers.

You cannot roll back a payment that was authorized on Stripe's servers by rolling back your local database transaction. You cannot release inventory in another service's database by issuing a `ROLLBACK` in yours.

**The core challenge:** In microservices, each service can guarantee ACID within its own database. But no mechanism guarantees ACID across multiple services' databases. Everything in this section exists to handle that gap.

> **Interview Relevance:** When you propose writing data across multiple services in an interview, the interviewer expects you to immediately address consistency. If you do not, they will ask, and it signals you have not thought about the hardest part of the design.

---

## SLIDE 87: Two-Phase Commit (2PC)

Two-phase commit is a protocol designed to achieve atomic transactions across multiple databases. It is important to understand it because interviewers ask about it, even though it is rarely used in modern microservices.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TWO-PHASE COMMIT (2PC)                          │
│                                                                   │
│   Phase 1: Prepare                                                │
│   ┌──────────────┐                                                │
│   │ Coordinator  │─────── Prepare ──────► Order DB               │
│   │              │─────── Prepare ──────► Inventory DB            │
│   │              │─────── Prepare ──────► Payment DB              │
│   │              │                                               │
│   │              │◄────── Ready ──────── Order DB                │
│   │              │◄────── Ready ──────── Inventory DB             │
│   │              │◄────── Ready ──────── Payment DB               │
│   └──────────────┘                                               │
│                                                                   │
│   Phase 2: Commit                                                 │
│   ┌──────────────┐                                                │
│   │ Coordinator  │─────── Commit ──────► Order DB                │
│   │              │─────── Commit ──────► Inventory DB             │
│   │              │─────── Commit ──────► Payment DB               │
│   └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────────┘
```

**Phase 1 (Prepare):** The coordinator asks each participant: "Can you commit this transaction?" Each participant acquires locks, validates the data, and responds "Ready" or "Abort."

**Phase 2 (Commit):** If all participants said "Ready," the coordinator sends "Commit" to all of them. If any participant said "Abort," the coordinator sends "Rollback" to all of them.

**What it guarantees:** Either all participants commit or all participants roll back. The transaction is atomic across all databases.

> **Interview Relevance:** Understanding 2PC is necessary because interviewers test whether you know why it exists and why it is avoided. The next slide explains why.

---

## SLIDE 88: Why 2PC Is Usually Avoided

2PC guarantees atomicity, but the guarantees come at a cost that makes it impractical for most microservices architectures.

#### Problem 1: Blocking and locks

During Phase 1, each participant holds locks on the rows involved in the transaction. These locks are held until Phase 2 completes. If the coordinator crashes between Phase 1 and Phase 2, participants hold their locks indefinitely, blocking all other transactions on those rows. In a high-throughput system processing thousands of orders per second, this lock contention is a severe bottleneck.

#### Problem 2: Coordinator is a single point of failure

If the coordinator crashes after sending "Prepare" but before sending "Commit," participants are stuck in an uncertain state. They have said "Ready" and are holding locks, but they do not know whether to commit or roll back. Recovery requires a new coordinator to read the transaction log and resume, which adds complexity and delay.

#### Problem 3: Latency

2PC requires two network round trips to every participant. In a microservices system where participants might be in different data centers, each round trip adds latency. A checkout that takes 50ms in a monolith might take 300-500ms with 2PC.

#### Problem 4: Reduced availability

If any single participant is unavailable, the entire transaction cannot proceed. With 4 participants each at 99.9% availability, the combined availability of the transaction is 99.6%. 2PC trades availability for consistency, which conflicts with the microservices goal of independent availability.

**The practical reality:** 2PC works for tightly controlled environments (two databases in the same data center, managed by the same team). It does not work well for independently deployed microservices with separate databases, separate teams, and separate availability requirements.

> **Interview Tip:** In an interview, explain 2PC briefly, then explain why you would use sagas instead. "2PC guarantees atomicity but creates a blocking, latency-heavy, availability-reducing protocol that conflicts with microservices independence. Sagas achieve eventual consistency without distributed locks." This shows depth.

> **Interview Relevance:** The ability to explain why 2PC is avoided, not just that it is avoided, separates candidates who understand distributed systems from those who memorized a pattern name.

---

## SLIDE 89: The Distributed Transaction Problem

Before learning the solution (sagas), you need to fully understand the problem. Here is a concrete scenario that illustrates why distributed consistency is hard.

#### Scenario: E-commerce checkout

A customer places an order. Three services must update their data:

- **Order Service:** Create the order record (status: PENDING)
- **Inventory Service:** Reserve 2 units of product XYZ
- **Payment Service:** Authorize $59.98 on the customer's credit card

The happy path works fine. All three services succeed. The order is confirmed.

#### Now consider the failure cases:

| Case | What Happens | Problem |
|------|--------------|---------|
| **Case 1** | Order created, inventory reserved, payment fails (insufficient funds) | The order exists. Inventory is reserved. But there is no payment. The customer is not charged, but the product is held and unavailable to other customers. |
| **Case 2** | Order created, inventory reservation fails (out of stock) | The order exists with no inventory backing it. If we proceed to payment, we charge the customer for something we cannot ship. |
| **Case 3** | Order created, inventory reserved, payment authorized, then the Order Service crashes before updating the order status to CONFIRMED | The payment was charged, inventory is reserved, but the order shows PENDING. The customer is charged but sees no confirmation. |

Each failure case requires a different recovery action. There is no single "rollback" that fixes everything. You need compensating transactions: cancel the order, release the inventory, refund the payment. And each compensating action can itself fail.

**This is the problem sagas solve.** Not with a single distributed transaction, but with a sequence of local transactions and compensating actions.

> **Interview Relevance:** Describing these specific failure cases in an interview before proposing your solution shows the interviewer you understand the problem deeply, not just the pattern name.

---

## SLIDE 90: The Saga Pattern

A saga is a sequence of local transactions, where each transaction updates one service's database, and each transaction has a corresponding compensating transaction that can undo its effect if a later step fails.

**The key insight:** A saga does not guarantee atomicity in the ACID sense. It guarantees that the system will eventually reach a consistent state, either by completing all steps (success) or by running compensating transactions for all completed steps (failure recovery).

#### How a saga differs from a transaction:

- A database transaction is **all-or-nothing**. Either everything commits or everything rolls back.
- A saga is a sequence of **committed** local transactions. Each step commits immediately in its own database. If step 3 fails, steps 1 and 2 have already committed. You cannot roll them back. You must **compensate** them.

#### Compensation is not the same as rollback:

- A database rollback **undoes** uncommitted changes as if they never happened.
- A compensating transaction creates a **new transaction** that semantically reverses the effect of a committed transaction.
- Cancelling an order is not rolling back the `INSERT`. It is creating a new state change: `UPDATE order SET status = 'CANCELLED'`.
- Refunding a payment is not undoing the charge. It is creating a new refund transaction on the customer's credit card.

#### The two implementation approaches:

1. **Orchestration:** A central coordinator service tells each participant what to do and handles failures. Covered in Slide 92.
2. **Choreography:** Each service reacts to events from the previous step. No central coordinator. Covered in Slide 93.

> **Interview Relevance:** When you mention sagas in an interview, immediately clarify that a saga is not a distributed transaction with rollback. It is a sequence of committed local transactions with compensating actions for failure recovery. This distinction matters.

---

## SLIDE 90B: Saga Pattern, Step-by-Step

#### How a saga executes:

**Forward flow (success):**

```
Step 1: Create Order     ──► Order DB: ORDER_CREATED
Step 2: Reserve Inventory ──► Inventory DB: INVENTORY_RESERVED
Step 3: Authorize Payment ──► Payment DB: PAYMENT_AUTHORIZED
Step 4: Confirm Order     ──► Order DB: ORDER_CONFIRMED
```

Each step executes and commits in its own service database. Step 1 commits in the Order DB. Step 2 commits in the Inventory DB. Step 3 commits in the Payment DB. Step 4 updates the Order DB to CONFIRMED.

**Backward flow (failure at Step 3):**

```
Step 3: Authorize Payment ──► FAILS (insufficient funds)
         │
         ▼
Compensate Step 2: Release Inventory ──► Inventory DB
         │
         ▼
Compensate Step 1: Cancel Order ──► Order DB
```

Payment authorization fails. Steps 1 and 2 have already committed (the order exists, inventory is reserved). The saga runs compensating transactions in reverse order: release the reserved inventory (compensating step 2), then cancel the order (compensating step 1).

#### What each step looks like:

| Step | Action | Compensating Action |
|------|--------|-------------------|
| **1. Create Order** | Insert order with status PENDING | Update order status to CANCELLED |
| **2. Reserve Inventory** | Decrement available stock, increment reserved stock | Increment available stock, decrement reserved stock |
| **3. Authorize Payment** | Call payment gateway to authorize the charge | Call payment gateway to void the authorization |
| **4. Confirm Order** | Update order status to CONFIRMED | No compensating action needed (this is the final step) |

> **Interview Relevance:** Being able to list each saga step with its corresponding compensating action for a concrete scenario is the level of detail interviewers expect at the senior level.

---

## SLIDE 90C: Interview Practice

### Question:

An interviewer asks: "How is a saga different from a distributed transaction? And what happens if a compensating transaction itself fails?"

*Take 60 seconds to think about both parts before reading below.*

---

### Weak Answer:

> "A saga is basically a distributed transaction but using events instead of 2PC. If a compensating transaction fails, we retry it."

**Why this is weak:** Saying a saga is "basically a distributed transaction" is incorrect. They have fundamentally different guarantees. And "retry it" does not address what happens if retries keep failing.

---

### Strong Answer:

> "A saga and a distributed transaction are fundamentally different. A distributed transaction (2PC) holds locks across all participants and guarantees atomicity, meaning either everything commits or nothing does. The data is never in an intermediate state visible to other transactions.
>
> A saga commits each step immediately in its own database. After step 2 commits, that data is visible to the rest of the system. If step 3 fails, steps 1 and 2 are already committed and visible. You cannot roll them back. You must run compensating transactions to semantically reverse their effects. During the compensation window, the system is in a temporarily inconsistent state. This is the trade-off: sagas give up atomicity for availability and independence.
>
> For the second part: if a compensating transaction fails, you retry it with exponential backoff. Compensating transactions must be designed to be idempotent, so retrying is always safe. If retries keep failing (the Inventory Service is down for an extended period), the failed compensation goes to a dead-letter queue. An alert fires. A human operator or an automated recovery process handles it when the downstream service recovers. In practice, this is rare, but the system must have a path for it. You never silently drop a failed compensation."

**What makes this strong:** The candidate explains the fundamental difference (committed vs uncommitted intermediate states), acknowledges the temporary inconsistency trade-off, and addresses the failure-of-compensation scenario with a practical answer (retry, DLQ, human intervention).

---

## SLIDE 91: Compensating Transactions

A compensating transaction is a new transaction that semantically reverses the effect of a previously committed transaction. It does not undo the original transaction. It creates a new action that counteracts it.

#### Why "undo" is the wrong mental model:

In a database, `ROLLBACK` discards uncommitted changes. The data reverts to its previous state as if the transaction never happened. Compensating transactions cannot do this because the original transaction has already been committed. Other services may have already read the committed data and acted on it.

#### Examples of compensating transactions:

| Original Action | Compensating Action |
|-----------------|-------------------|
| Create order (status: PENDING) | Update order status to CANCELLED. The order record still exists in the database. It is not deleted. |
| Reserve 2 units of inventory | Release 2 units back to available stock. The reservation record may be kept for auditing. |
| Authorize $59.98 payment | Void the authorization (if not yet captured) or refund the charge (if already captured). The payment record shows both the charge and the reversal. |
| Send confirmation email | **There is none.** You cannot unsend an email. This is why notification should be the last step in a saga, or sent only after the saga is fully confirmed. |

#### Design rules for compensating transactions:

1. **Every saga step must have a defined compensating action** before you implement the saga. If you cannot define how to compensate a step, reconsider whether it belongs in the saga.

2. **Compensating transactions must be idempotent.** Running "release inventory for order ord_123" twice should produce the same result as running it once.

3. **Some actions are not compensable.** Sending an email, calling a third-party API with side effects, or shipping a physical product cannot be reversed. Design the saga so non-compensable steps happen last.

> **Interview Relevance:** Explaining that compensation is not rollback, and listing which steps are compensable versus which are not, demonstrates a practical understanding of saga design that most candidates lack.

---

## SLIDE 92: Orchestration

In the orchestration approach, a central coordinator (the orchestrator) manages the saga by telling each service what to do and handling the results.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATION APPROACH                          │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    ORCHESTRATOR                             │  │
│  │                                                             │  │
│  │  1. Send "Create Order"   ──► Order Service                │  │
│  │  2. Receive response                                       │  │
│  │  3. Send "Reserve Stock"  ──► Inventory Service            │  │
│  │  4. Receive response                                       │  │
│  │  5. Send "Authorize Payment" ──► Payment Service           │  │
│  │  6. Receive response                                       │  │
│  │  7. Send "Confirm Order"  ──► Order Service                │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│    Order Service    Inventory Service    Payment Service          │
└─────────────────────────────────────────────────────────────────────┘
```

#### How the orchestrator works:

The orchestrator holds the saga's state machine. It knows which step to execute next, what to do if a step succeeds, and what compensating actions to trigger if a step fails. It sends commands to each service ("create this order," "reserve this inventory") and waits for responses.

#### Advantages of orchestration:

- The entire workflow is visible in one place (the orchestrator's code)
- Easy to understand the flow by reading the orchestrator logic
- Easy to add new steps or change the order
- Centralized error handling and compensation logic

#### Disadvantages of orchestration:

- The orchestrator can become a single point of failure
- Risk of the orchestrator accumulating too much business logic (it should coordinate, not compute)
- Tighter coupling: the orchestrator must know about every service in the saga

**When to use orchestration:** Complex sagas with many steps (5+), conditional branching, or intricate failure handling. The visibility of a central coordinator outweighs the coupling cost.

> **Interview Relevance:** When you describe a saga in an interview, the interviewer will ask whether you would use orchestration or choreography. Having a clear opinion with reasoning is expected.

---

## SLIDE 93: Choreography

In the choreography approach, there is no central coordinator. Each service listens for events and reacts by executing its step and publishing the next event.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CHOREOGRAPHY APPROACH                           │
│                                                                   │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐        │
│  │   Order     │────►│  Inventory  │────►│   Payment   │        │
│  │   Service   │     │   Service   │     │   Service   │        │
│  │             │     │             │     │             │        │
│  │ Publish:    │     │ Publish:    │     │ Publish:    │        │
│  │ OrderCreated│     │ Inventory  │     │ Payment     │        │
│  │             │     │ Reserved    │     │ Authorized  │        │
│  └─────────────┘     └─────────────┘     └─────────────┘        │
│         │                   │                   │                │
│         └───────────────────┼───────────────────┘                │
│                             │                                    │
│                      ┌──────▼──────┐                             │
│                      │  Event Bus  │                             │
│                      │  (Kafka)    │                             │
│                      └─────────────┘                             │
└─────────────────────────────────────────────────────────────────────┘
```

#### How choreography works:

No service tells another service what to do. Each service publishes an event describing what happened. Other services subscribe to events they care about and decide independently how to respond. The Order Service publishes "OrderCreated." The Inventory Service hears it and reserves inventory. It publishes "InventoryReserved." The Payment Service hears it and authorizes payment.

#### Advantages of choreography:

- No single point of failure (no orchestrator to crash)
- Loose coupling: services only know about events, not about each other
- Easy to add new consumers without changing existing services
- Each service is autonomous

#### Disadvantages of choreography:

- The overall workflow is spread across multiple services. No single place shows the full flow.
- Debugging is harder. You must trace events across services to understand what happened.
- Complex compensation logic is distributed. Each service must know when and how to compensate.
- Cyclic event chains can be hard to detect and prevent.

**When to use choreography:** Simple sagas with 2-3 steps where the flow is straightforward and failure handling is simple. The loose coupling and autonomy outweigh the reduced visibility.

> **Interview Relevance:** Choosing between orchestration and choreography is a design decision that interviewers test. The next slide provides the comparison framework.

---

## SLIDE 94: Orchestration vs Choreography

| Dimension | Orchestration | Choreography |
|-----------|--------------|--------------|
| **Control flow** | Central coordinator manages the sequence | Each service reacts to events independently |
| **Visibility** | Entire saga visible in orchestrator code | Workflow spread across multiple services |
| **Coupling** | Orchestrator knows all participating services | Services only know about events |
| **Single point of failure** | Orchestrator can fail | No single coordinator to fail |
| **Debugging** | Read the orchestrator logic | Trace events across services |
| **Adding new steps** | Modify the orchestrator | Add a new event consumer |
| **Compensation handling** | Centralized in orchestrator | Distributed across services |
| **Complexity ceiling** | Handles complex workflows well | Becomes confusing beyond 3-4 steps |
| **Best for** | Complex sagas, many steps, conditional logic | Simple sagas, 2-3 steps, straightforward flow |

#### The practical reality:

Most production systems use orchestration for their most critical workflows (checkout, payment, fulfillment) and choreography for simpler, less critical flows (analytics updates, notification triggers, cache invalidation).

Many teams start with choreography because it feels simpler, then migrate to orchestration when the workflow grows to 5+ steps and debugging event chains becomes painful.

> **Interview Tip:** A strong interview answer is: "I would use orchestration for the checkout saga because it has 5 steps with complex failure handling, and I need the full workflow visible in one place. For downstream processes like sending notifications and updating analytics, I would use choreography because those are simple, independent reactions to the OrderConfirmed event."

> **Interview Relevance:** Choosing the right approach for different workflows within the same system, rather than picking one approach for everything, shows mature architectural judgment.

---

## SLIDE 95: The Transactional Outbox Pattern

The transactional outbox solves a critical reliability problem: how do you update your database and publish an event atomically? Without it, you can lose events or publish events for changes that were rolled back.

#### The problem:

```
1. Order Service inserts order into database
2. Order Service publishes OrderCreated event to message broker
```

What if the database insert succeeds (step 1) but the event publish fails (step 2)? The order exists but no other service knows about it. Inventory is not reserved. Payment is not authorized. The order is stuck.

What if the event publishes (step 2) but the database insert fails (step 1)? Other services start processing an order that does not exist.

#### The solution: Write the event to the database, not the message broker.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TRANSACTIONAL OUTBOX PATTERN                    │
│                                                                   │
│  ┌─────────────────────────────┐                                  │
│  │   Order Service             │                                  │
│  │                             │                                  │
│  │   BEGIN TRANSACTION;        │                                  │
│  │     INSERT INTO orders...   │                                  │
│  │     INSERT INTO outbox...   │  ◄─── Same transaction          │
│  │   COMMIT;                   │                                  │
│  └──────────────┬──────────────┘                                  │
│                 │                                                 │
│                 ▼                                                 │
│  ┌─────────────────────────────┐                                  │
│  │   Message Relay             │                                  │
│  │   (separate process)        │                                  │
│  │                             │                                  │
│  │  1. Poll outbox table       │                                  │
│  │  2. Publish to broker       │                                  │
│  │  3. Mark as published       │                                  │
│  └─────────────────────────────┘                                  │
└─────────────────────────────────────────────────────────────────────┘
```

Instead of publishing directly to the message broker, the Order Service writes the event to an outbox table in the same database as the order, within the same transaction.

A separate process (the message relay or CDC connector) reads the outbox table and publishes events to the message broker. After successful publication, the relay marks the outbox entry as sent.

**Why this works:** The order insert and the outbox insert are in the same database transaction. They either both commit or both roll back. You never have an order without an outbox entry, and you never have an outbox entry without an order.

> **Interview Relevance:** The transactional outbox is a specific pattern that interviewers expect senior candidates to know. It shows you understand that publishing an event and writing to a database are two separate operations that can fail independently.

---

## SLIDE 95B: Transactional Outbox, Implementation

#### The outbox table:

```sql
CREATE TABLE outbox (
    event_id UUID PRIMARY KEY,
    aggregate_type VARCHAR(100),
    aggregate_id VARCHAR(100),
    event_type VARCHAR(100),
    payload JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    published BOOLEAN DEFAULT FALSE
);
```

#### Writing to the outbox within the same transaction as the business data:

```python
def create_order(customer_id, items):
    order_id = generate_uuid()
    event_id = generate_uuid()

    db.begin_transaction()
    db.execute(
        "INSERT INTO orders (order_id, customer_id, status) VALUES (?, ?, ?)",
        order_id, customer_id, "PENDING"
    )
    db.execute(
        "INSERT INTO outbox (event_id, aggregate_type, aggregate_id, event_type, payload) VALUES (?, ?, ?, ?, ?)",
        event_id, "Order", order_id, "OrderCreated",
        json.dumps({"order_id": order_id, "customer_id": customer_id, "items": items})
    )
    db.commit_transaction()
```

The order insert and the outbox insert succeed or fail together. There is no window where one exists without the other.

#### The message relay:

A separate process polls the outbox table for unpublished events, publishes them to the message broker, and marks them as published. Alternatively, Change Data Capture (CDC) tools like Debezium can tail the database transaction log and publish outbox entries automatically without polling.

#### What about duplicate events?

The relay might publish an event and crash before marking it as published. On restart, it publishes the same event again. This means consumers must be idempotent, which is a requirement we will address in the next slides.

> **Interview Relevance:** Showing the actual table schema and transaction code in an interview demonstrates implementation-level understanding, not just conceptual awareness.

---

## SLIDE 95C: Interview Practice

### Question:

An interviewer asks: "Your Order Service creates an order and needs to notify the Inventory Service to reserve stock. How do you guarantee that the event is always published when the order is created?"

*Take 60 seconds to think about the reliability problem before reading below.*

---

### Weak Answer:

> "After inserting the order into the database, I publish an event to Kafka. If the publish fails, I retry."

**Why this is weak:** There is a fundamental gap between the database commit and the Kafka publish. If the application crashes after the database commit but before the Kafka publish, the event is lost permanently. Retrying only works if the application is still running. The answer does not address the atomicity problem.

---

### Strong Answer:

> "I would use the transactional outbox pattern. Instead of publishing directly to Kafka, I write the event to an outbox table in the same database as the order, within the same transaction. Both the order row and the outbox row either commit together or roll back together.
>
> A separate message relay process reads the outbox table and publishes events to Kafka. After successful publication, it marks the outbox entry as sent. If the relay crashes before marking it, it republishes on restart, which means consumers might receive duplicate events. So consumers must be idempotent, processing the same event twice produces the same result as processing it once.
>
> Alternatively, I could use Debezium with Change Data Capture to tail the PostgreSQL write-ahead log and publish outbox entries automatically. This avoids polling latency and is the approach many production systems use."

**What makes this strong:** The candidate identifies the atomicity problem (database commit and event publish are separate operations), proposes a concrete solution (outbox), explains the duplicate event consequence, and mentions a production-grade implementation (Debezium CDC).

---

## SLIDE 96: The Inbox Pattern and Message Deduplication

The outbox pattern guarantees that events are published **at least once**. But "at least once" means the same event might be delivered to a consumer more than once. The inbox pattern handles this on the consumer side.

#### Why duplicates happen:

- The message relay publishes an event to Kafka but crashes before marking it as sent. On restart, it publishes again.
- Kafka delivers a message to a consumer, but the consumer crashes before committing its offset. Kafka redelivers.
- Network issues cause a retry that results in the broker receiving the same message twice.

#### The inbox pattern:

Each consumer maintains an inbox table that records the ID of every event it has processed.

```python
def handle_order_created(event):
    if db.exists("SELECT 1 FROM inbox WHERE event_id = ?", event.event_id):
        return

    db.begin_transaction()
    db.execute("INSERT INTO inbox (event_id, processed_at) VALUES (?, NOW())", event.event_id)
    db.execute(
        "UPDATE inventory SET reserved = reserved + ? WHERE product_id = ?",
        event.quantity, event.product_id
    )
    db.commit_transaction()
```

Before processing an event, the consumer checks if the event ID is already in the inbox. If it is, the event has already been processed and is skipped. If it is not, the consumer inserts the event ID into the inbox and processes the event in the same transaction.

#### Inbox table maintenance:

Over time, the inbox table grows. Periodically purge entries older than the message broker's retention period (e.g., if Kafka retains messages for 7 days, purge inbox entries older than 7 days).

> **Interview Relevance:** Mentioning the inbox pattern alongside the outbox pattern shows the interviewer you understand the full reliability chain: reliable publishing (outbox) and reliable consumption (inbox).

---

## SLIDE 97: Idempotency Keys

Idempotency is the property that executing an operation multiple times produces the same result as executing it once. In microservices with at-least-once delivery, idempotency is not optional. It is a requirement for correctness.

#### Operations that are naturally idempotent:

- **Setting a value:** "Set user name to Jonathan" produces the same result whether run once or five times.
- **Deleting a record by ID:** "Delete order ord_123" succeeds once and is a no-op afterward.
- **Reading data:** Always idempotent.

#### Operations that are not naturally idempotent:

- **Incrementing a value:** "Add $10 to account balance" gives a different result each time.
- **Creating a record with auto-generated ID:** Each execution creates a new record.
- **Sending an email:** Each execution sends another email.

#### Making non-idempotent operations safe with idempotency keys:

The caller generates a unique key for each logical operation and sends it with the request. The server checks if that key has been seen before. If yes, it returns the stored result without re-executing. This was covered briefly in Section 4 (Slide 56), but in the context of sagas it becomes critical.

#### Why idempotency is critical for sagas:

In a saga, the orchestrator sends a command to the Payment Service: "Authorize $59.98 for order ord_123." If the Payment Service processes it but the response is lost, the orchestrator retries. Without idempotency, the customer is charged twice. With an idempotency key, the Payment Service recognizes the duplicate and returns the original result.

> **Interview Relevance:** Idempotency is one of the most practical and frequently tested concepts in distributed systems interviews. If you propose retries anywhere in your design, the interviewer will ask how you handle duplicates.

---

## SLIDE 98: At-Least-Once Delivery and the Exactly-Once Myth

Most message brokers guarantee **at-least-once delivery**. This means every message will be delivered, but some messages might be delivered more than once. Understanding why exactly-once delivery is effectively impossible helps you design systems correctly.

#### Why exactly-once is a myth in distributed systems:

Imagine the message broker delivers a message to the consumer. The consumer processes it and sends an acknowledgment. But the acknowledgment is lost due to a network failure. The broker thinks the message was not delivered and redelivers it. The consumer has already processed it. From the broker's perspective, it delivered twice. From the consumer's perspective, it received twice. Neither side did anything wrong.

#### The three delivery guarantees:

| Guarantee | Description | When to Use |
|-----------|-------------|-------------|
| **At-most-once** | The message is delivered zero or one times. If delivery fails, the message is lost. Fast but unreliable. | Acceptable for metrics and analytics where losing a data point is not critical. |
| **At-least-once** | The message is delivered one or more times. No messages are lost, but duplicates are possible. | **This is the default for most systems** (Kafka, RabbitMQ, SQS). |
| **Effectively-once** | At-least-once delivery combined with idempotent processing on the consumer side. Each message is delivered at least once, but the consumer's idempotency logic ensures it is only processed once. | **This is what production systems actually achieve.** |

**The practical approach:** Design for at-least-once delivery (the default) and make every consumer idempotent (using inbox tables or idempotency keys). This gives you "effectively-once" processing, which is the strongest guarantee achievable in practice.

> **Interview Relevance:** If an interviewer asks about "exactly-once delivery," the correct answer is that it is not achievable in a distributed system, but you can achieve effectively-once semantics through idempotent consumers. This shows real understanding of distributed systems theory.

---

## SLIDE 99: Dead-Letter Queues

A dead-letter queue (DLQ) is a holding area for messages that cannot be processed after multiple retry attempts. Instead of retrying forever or silently dropping the message, failed messages are moved to the DLQ for investigation.

#### When a message goes to the DLQ:

- The consumer has retried processing 3-5 times and keeps failing
- The message payload is malformed and cannot be deserialized
- A bug in the consumer causes a crash every time it encounters this specific message type
- A downstream dependency is permanently unavailable for this specific request

#### What happens to messages in the DLQ:

1. An alert fires notifying the on-call engineer that messages are accumulating in the DLQ
2. The engineer investigates: is this a bug, a bad message, or a temporary downstream outage?
3. If the issue is fixed (bug patched, downstream recovered), messages are replayed from the DLQ back to the main queue
4. If the message is genuinely unprocessable (corrupted data, invalid state), it is logged and archived

#### Why DLQs are essential for sagas:

In a saga, a compensating transaction that cannot be processed is a serious consistency problem. If "release inventory for order ord_123" fails and goes to the DLQ, inventory remains reserved for an order that has been cancelled. The DLQ ensures this failure is visible and eventually handled, rather than silently ignored.

> **Interview Relevance:** Mentioning dead-letter queues when discussing message-driven architectures shows the interviewer you think about what happens when processing fails, not just when it succeeds.

---

## SLIDE 100: Retry and Backoff

Retrying failed operations is essential in distributed systems, but retrying incorrectly can turn a minor outage into a catastrophic one.

#### The retry storm problem:

The Inventory Service goes down for 10 seconds. During those 10 seconds, 5,000 messages accumulate. All 5,000 messages retry simultaneously. The Inventory Service comes back up and immediately receives 5,000 requests. It cannot handle the burst and goes down again. This cycle repeats, and the service never recovers.

#### Exponential backoff:

Instead of retrying immediately, wait an increasing amount of time between retries: 1 second, 2 seconds, 4 seconds, 8 seconds. This spreads retries over time and gives the recovering service breathing room.

#### Jitter:

Even with exponential backoff, if 5,000 callers all start with a 1-second delay, they all retry at the same time. Adding random jitter (a random delay of 0 to backoff_interval) desynchronizes the retries so the recovering service receives a steady trickle instead of a burst.

#### Retry budget:

Set a maximum number of retries (typically 2-3 for synchronous calls, 5-10 for asynchronous messages). After exhausting the retry budget, fail the operation and move the message to a dead-letter queue.

#### The formula:

```
delay = min(base_delay * 2^attempt + random(0, base_delay), max_delay)
```

With `base_delay = 1 second` and `max_delay = 30 seconds`:
- Attempt 1 waits 1-2s
- Attempt 2 waits 2-3s
- Attempt 3 waits 4-5s
- Attempt 4 waits 8-9s, capped at 30 seconds

> **Interview Relevance:** Mentioning "exponential backoff with jitter" is the expected answer whenever retries are discussed. If you just say "retry after 1 second," the interviewer will ask about retry storms.

---

## SLIDE 100B: Note It Down

### Messaging reliability patterns to memorize:

| Pattern | Description |
|---------|-------------|
| **Transactional outbox** | Write events to the database, not the broker. Same transaction as the business data. Relay publishes later. |
| **Inbox pattern** | Consumer deduplicates by tracking processed event IDs in a local table. |
| **Idempotency keys** | Caller-generated unique key per operation. Server returns stored result on duplicate. |
| **At-least-once delivery** | Default for Kafka, RabbitMQ, SQS. Messages are never lost but may be duplicated. |
| **Effectively-once** | At-least-once delivery + idempotent consumers. The strongest practical guarantee. |
| **Dead-letter queue** | Failed messages go to a DLQ after retry budget is exhausted. Alert and investigate. |
| **Exponential backoff with jitter** | Prevent retry storms. Delay increases exponentially. Jitter desynchronizes retries. |

**The chain of reliability:**

1. **Outbox** ensures events are published
2. **At-least-once delivery** ensures events reach consumers
3. **Inbox/idempotency** ensures events are processed exactly once
4. **DLQ** ensures failures are visible
5. **Backoff** ensures retries do not cause cascading failure

> **Interview Relevance:** Being able to describe this full reliability chain in an interview demonstrates production-grade distributed systems knowledge. Most candidates know one or two of these patterns. Knowing all seven and how they compose is a differentiator.

---

## SLIDE 101: Partial Failure Handling

Partial failure is the defining challenge of distributed systems. Unlike a monolith where an operation either fully succeeds or fully fails, microservices can leave you in a state where some steps succeeded and others did not.

#### The three response strategies for partial failure:

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| **Forward recovery (retry and complete)** | If step 3 of 4 fails, retry step 3. If the failure is transient (network timeout, temporary unavailability), retrying may succeed and the saga completes normally. | This is the first strategy to try and handles the majority of failures in practice. |
| **Backward recovery (compensate and cancel)** | If step 3 fails permanently (insufficient funds, invalid data), run compensating transactions for steps 1 and 2. The saga moves backward, undoing completed steps and leaving the system in a clean cancelled state. | Use for permanent business errors that cannot be retried. |
| **Best-effort with human intervention** | If compensation itself fails (the Inventory Service is down and cannot release reserved stock), the system logs the failure, alerts the operations team, and the saga enters a COMPENSATION_FAILED state. | Use for persistent infrastructure errors that cannot be resolved automatically. |

#### Choosing between strategies:

| Error Type | Strategy |
|------------|----------|
| Transient error (timeout, 503) | Retry with backoff (forward recovery) |
| Business error (insufficient funds, out of stock) | Compensate (backward recovery) |
| Persistent infrastructure error | DLQ and human intervention |

> **Interview Tip:** In an interview, explicitly mention all three strategies. "First, I retry because most failures are transient. If the failure is permanent, I run compensating transactions. If compensation fails, the saga enters a stuck state and the operations team is alerted." This shows you think about the full failure spectrum.

> **Interview Relevance:** The ability to describe what happens when things go wrong at multiple levels, not just the happy path, is the single biggest differentiator between senior and mid-level interview answers.

---

## SLIDE 102: State Machines for Service Lifecycles

A state machine defines every possible state a business entity can be in and every valid transition between states. In microservices with sagas, state machines prevent invalid state transitions and make the system's behavior predictable and debuggable.

#### Example: Order lifecycle state machine

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ORDER LIFECYCLE STATE MACHINE                   │
│                                                                   │
│                        ┌─────────────┐                            │
│                        │   CREATED   │                            │
│                        └──────┬──────┘                            │
│                               │                                   │
│                 ┌─────────────┼─────────────┐                     │
│                 │             │             │                     │
│                 ▼             ▼             ▼                     │
│          ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│          │ INVENTORY  │ │  PAYMENT   │ │ CANCELLED  │           │
│          │ RESERVED   │ │ AUTHORIZED │ │            │           │
│          └─────┬──────┘ └──────┬─────┘ └────────────┘           │
│                │               │                                  │
│                └───────┬───────┘                                  │
│                        │                                          │
│                        ▼                                          │
│                 ┌────────────┐                                    │
│                 │ CONFIRMED  │                                    │
│                 └──────┬─────┘                                    │
│                        │                                          │
│                        ▼                                          │
│                 ┌────────────┐                                    │
│                 │  SHIPPED   │                                    │
│                 └──────┬─────┘                                    │
│                        │                                          │
│                        ▼                                          │
│                 ┌────────────┐                                    │
│                 │ DELIVERED  │                                    │
│                 └────────────┘                                    │
└─────────────────────────────────────────────────────────────────────┘
```

#### Why state machines matter for sagas:

Without a state machine, an order can end up in an undefined state. What happens if the Payment Service sends a "PaymentAuthorized" event for an order that is already CANCELLED? Without a state machine, the Order Service might process it and move the order to PAYMENT_AUTHORIZED, overwriting the cancellation. With a state machine, the transition from CANCELLED to PAYMENT_AUTHORIZED is not defined, so it is rejected.

#### Implementation:

```python
VALID_TRANSITIONS = {
    "CREATED": ["INVENTORY_RESERVED", "CANCELLED"],
    "INVENTORY_RESERVED": ["PAYMENT_AUTHORIZED", "CANCELLED"],
    "PAYMENT_AUTHORIZED": ["CONFIRMED", "CANCELLED"],
    "CONFIRMED": ["SHIPPED", "CANCELLED"],
    "SHIPPED": ["DELIVERED"],
    "CANCELLED": [],
    "DELIVERED": [],
}

def transition_order(order, new_status):
    if new_status not in VALID_TRANSITIONS.get(order.status, []):
        raise InvalidStateTransitionError(order.status, new_status)
    order.status = new_status
    db.save(order)
```

Every state change goes through this validation. Invalid transitions are rejected and logged, making bugs immediately visible instead of silently corrupting data.

> **Interview Relevance:** Drawing a state machine diagram on the whiteboard when designing a saga shows the interviewer you think about every possible state, including error states. This is a hallmark of production-experienced engineers.

---

## SLIDE 103: Worked Example, Checkout Saga (Happy Path)

**Scenario:** A customer places an order for 2 items totaling $59.98 on an e-commerce platform like Amazon. The checkout saga uses orchestration.

#### Saga steps (happy path):

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Create  │    │ Reserve │    │Authorize│    │ Confirm │    │ Create  │    │ Send    │
│ Order   │───►│Inventory│───►│Payment  │───►│ Order   │───►│Shipment │───►│Email    │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
    │              │              │              │              │              │
    ▼              ▼              ▼              ▼              ▼              ▼
 Order DB      Inventory DB    Payment DB      Order DB      Shipping DB    Async
(created)      (reserved)      (authorized)   (confirmed)   (created)      (email)
```

#### Each step and its data:

| Step | Service | Action | Idempotency Key | State After |
|------|---------|--------|-----------------|-------------|
| 1 | Order Service | Create order record | `order_id: ord_123` | CREATED |
| 2 | Inventory Service | Reserve 2 units of prod_789 | `reservation_id: res_001` | INVENTORY_RESERVED |
| 3 | Payment Service | Authorize $59.98 | `idempotency_key: ik_abc` | PAYMENT_AUTHORIZED |
| 4 | Order Service | Update status to CONFIRMED | `order_id: ord_123` | CONFIRMED |
| 5 | Shipping Service | Create shipment | `shipment_id: ship_789` | SHIPPED (pending) |
| 6 | Notification Service | Send confirmation email | Async, fire-and-forget | No state change |

Notice: Every mutating step has an idempotency key. The notification is the last step and is asynchronous, because you cannot unsend an email, and email failure should never block an order.

> **Interview Relevance:** Being able to walk through a complete checkout saga with specific IDs, idempotency keys, and state transitions demonstrates production-level understanding of distributed workflows.

---

## SLIDE 104: Worked Example, Checkout Saga (Failure Cases)

Using the same checkout saga from the previous slide, here is what happens when each step fails.

#### Failure Case 1: Inventory reservation fails (out of stock)

| Saga Step | What Happens |
|-----------|--------------|
| Step 1: Create order | Succeeds. Order exists with status CREATED. |
| Step 2: Reserve inventory | **Fails.** Product is out of stock. |
| **Compensation: Cancel order** | Order status updated to CANCELLED. |
| **Result** | Customer sees "Sorry, this item is out of stock." |

#### Failure Case 2: Payment authorization fails (insufficient funds)

| Saga Step | What Happens |
|-----------|--------------|
| Step 1: Create order | Succeeds. |
| Step 2: Reserve inventory | Succeeds. 2 units reserved. |
| Step 3: Authorize payment | **Fails.** Card declined. |
| **Compensation: Release inventory** | 2 units returned to available stock. |
| **Compensation: Cancel order** | Order status updated to CANCELLED. |
| **Result** | Customer sees "Payment failed. Please try a different payment method." |

#### Failure Case 3: Shipping creation fails after payment

| Saga Step | What Happens |
|-----------|--------------|
| Steps 1-4 | All succeed. Order is CONFIRMED. Payment is authorized. |
| Step 5: Create shipment | **Fails.** Shipping provider API is down. |
| **Decision** | Do NOT cancel the order or refund payment. Retry shipping. |
| **Action** | Retry with exponential backoff. After 3 retries, move to DLQ. Alert operations. |
| **Result** | Customer sees order as CONFIRMED. Shipment is created when provider recovers. |

**Key insight:** Not every failure triggers full compensation. Shipping failure after payment does not require refunding the customer. The correct response is to retry shipping, because the customer wants their order, not a refund. Full compensation (refund + cancel) is only appropriate when the core operation cannot proceed (no stock, no payment).

> **Interview Relevance:** Walking through these specific failure cases and explaining why the response differs for each one is the highest-signal demonstration of saga understanding in an interview.

---

## SLIDE 104B: Interview Practice

### Question:

An interviewer asks: "In your checkout saga, the payment authorization succeeds, but then the Inventory Service goes down and you cannot release inventory during compensation. The customer has been charged but inventory is stuck in reserved state. What do you do?"

*Take 60 seconds to think about this edge case before reading below.*

---

### Weak Answer:

> "I would retry the inventory release until it succeeds."

**Why this is weak:** The Inventory Service might be down for hours. The customer has been charged. Retrying indefinitely without any fallback leaves the customer in a bad state.

---

### Strong Answer:

> "This is a compensation failure, which is the hardest case in saga design. Here is how I would handle it.
>
> First, the orchestrator retries the inventory release with exponential backoff and jitter, with a retry budget of 5 attempts over about 2 minutes. Most Inventory Service outages are transient and resolve within that window.
>
> If retries are exhausted, the orchestrator marks the saga as COMPENSATION_FAILED and the failed compensation message goes to a dead-letter queue. An alert fires to the operations team.
>
> For the customer, I would void the payment authorization immediately. Payment authorization is a hold on funds, not a charge. Voiding it costs nothing and releases the hold on the customer's card. I do not need the Inventory Service to be available to void the payment.
>
> For the stuck inventory reservation, when the Inventory Service recovers, a reconciliation job processes all DLQ messages and releases the orphaned reservations. This can also be done manually by the operations team if urgency requires it.
>
> The saga's final state is: order CANCELLED, payment voided, inventory released (eventually). The customer is never charged for a cancelled order."

**What makes this strong:** The candidate addresses the immediate customer impact (void the payment hold), the eventual consistency resolution (DLQ and reconciliation), and the monitoring path (alerts to operations). They demonstrate that compensation failures are expected and planned for, not surprising edge cases.

---

## SLIDE 105: Human Intervention and Manual Recovery

No matter how well you design your sagas, some failures cannot be resolved automatically. Acknowledging this and designing for human intervention is a sign of production maturity, not a sign of failure.

#### When human intervention is needed:

- A compensating transaction has failed after all retries and the DLQ contains messages that cannot be replayed because the downstream service has a data corruption issue
- A third-party payment provider charged the customer but returned an error response, so your system thinks the payment failed but the customer sees a charge on their card
- An inventory discrepancy exists where the physical warehouse count does not match the digital count, and automated reconciliation cannot resolve it
- A regulatory or compliance issue requires manual review before proceeding (fraud suspicion, sanctions screening)

#### Designing for human intervention:

- **Saga status dashboard:** A UI that shows all sagas, their current state, and any sagas in COMPENSATION_FAILED state. Operations teams can view the saga history, see which step failed, and trigger manual resolution.
- **Manual compensation actions:** Admin endpoints that allow operators to manually release inventory, void payments, or update order status. These are protected by authentication and audit logging.
- **Runbooks:** Step-by-step instructions for each type of compensation failure. "If the Payment Service charged the customer but the saga shows PAYMENT_FAILED, verify with the payment provider, then manually update the saga state."

#### The honest reality:

In a system processing millions of orders per year, a small fraction (0.001-0.01%) will require human intervention. At 10 million orders per year, that is 100-1,000 cases annually. Building the tooling to handle these efficiently is part of production-grade microservices design.

> **Interview Relevance:** Mentioning human intervention as a deliberate part of your saga design shows the interviewer you have production experience. Systems that claim to handle every failure automatically are either lying or have not encountered enough failure modes yet.

---

## SLIDE 106: Section Summary and Interview Tips

### Section 6 Summary:

| Concept | Key Takeaway |
|---------|--------------|
| **Why ACID is not enough** | A single transaction cannot span multiple service databases |
| **Two-phase commit** | Guarantees atomicity but creates blocking, latency, and availability problems |
| **Saga pattern** | Sequence of local transactions with compensating actions for failure recovery |
| **Compensating transactions** | New transactions that semantically reverse committed steps. Not rollback. |
| **Orchestration** | Central coordinator manages the saga. Best for complex workflows. |
| **Choreography** | Services react to events. Best for simple, loosely coupled flows. |
| **Transactional outbox** | Write events to database in same transaction as business data. Relay publishes later. |
| **Inbox pattern** | Consumer deduplicates by tracking processed event IDs |
| **Idempotency keys** | Unique key per operation. Server returns stored result on duplicate. |
| **At-least-once delivery** | Default guarantee. Combine with idempotent consumers for effectively-once. |
| **Dead-letter queues** | Holding area for messages that fail after all retries |
| **Exponential backoff with jitter** | Prevent retry storms. Increasing delay with randomization. |
| **Partial failure** | Retry for transient errors. Compensate for permanent errors. Escalate if compensation fails. |
| **State machines** | Define every valid state and transition. Reject invalid transitions. |
| **Human intervention** | Some failures require manual resolution. Design tooling for it. |

### Common Interview Mistakes:

| Mistake | Why It Hurts You | Stronger Approach |
|---------|------------------|-------------------|
| "A saga is like a distributed transaction" | They are fundamentally different. Sagas commit each step; transactions hold locks across all. | "A saga is a sequence of committed local transactions with compensating actions, not a distributed transaction with rollback." |
| No compensating transactions defined | Shows you have not thought about failure recovery | Define a compensating action for every saga step before implementing the saga |
| "We use exactly-once delivery" | This is not achievable in distributed systems | "We use at-least-once delivery with idempotent consumers for effectively-once semantics." |
| Only describing the happy path | The happy path is the easy part. Interviewers care about failures. | Walk through at least two failure cases with specific compensating actions. |
| Ignoring compensation failures | What if the compensating transaction itself fails? | "If compensation fails after retries, the message goes to a DLQ, an alert fires, and a human operator resolves it." |

### Key Numbers to Remember:

| Metric | Value |
|--------|-------|
| Retry budget (synchronous) | 2-3 attempts |
| Retry budget (asynchronous messages) | 5-10 attempts |
| Backoff formula | `min(base_delay * 2^attempt + jitter, max_delay)` |
| Typical max backoff | 30-60 seconds |
| DLQ alert threshold | Any message in DLQ triggers investigation |
| Human intervention rate | 0.001-0.01% of transactions in mature systems |
| Outbox relay latency | 100ms-1 second (polling) or near-real-time (CDC) |

### How This Connects to Other Sections:

**Section 3 (Service Decomposition)** defined aggregates as the consistency boundary within a service. Everything inside an aggregate is a local ACID transaction. Everything that crosses an aggregate boundary is a saga, which this section taught you how to design.

**Section 4 (Communication Patterns)** provided the messaging infrastructure (events, queues, streams) that sagas rely on. The outbox pattern from this section ensures reliable event publishing over the messaging infrastructure from Section 4.

**Section 5 (Data Ownership)** identified cross-service writes as the hardest data challenge. This section provided the solution: sagas with compensating transactions, orchestrated or choreographed, with the full reliability chain of outbox, inbox, idempotency, and DLQ.

**Section 7 (API Gateway)** handles the edge where checkout requests arrive, and **Section 8 (Reliability)** covers the observability needed to monitor saga health in production.
