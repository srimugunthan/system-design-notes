Here is a summary of the AWS Full Course video, broken down into 15-minute chronological chunks:

### [[00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=0)] - [[15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=900)] Course Overview & The 7-Step AWS Road Map

* **Introduction to AWS:** The video highlights AWS's position as a cloud market leader (holding nearly a third of the global cloud market). It provides an overview of how renting compute resources beats managing hardware manually [[01:03](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=63)].
* **Building Blocks:** The instructor sets a 7-step road map for 2025/2026 to become an AWS Cloud Engineer, beginning with fundamental skills like Linux commands, networking basics, Python scripting, and virtualization [[03:40](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=220)].
* **Cloud Architecture Basics:** Explains cloud concepts such as public, private, and hybrid models alongside the three deployment flavors: Infrastructure as a Service (IaaS), Platform as a Service (PaaS), and Software as a Service (SaaS) [[05:08](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=308)].
* **Core Domains:** Introduces the four foundational pillars of AWS infrastructure—Compute, Storage, Networking, and Security [[06:10](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=370)]. It previews other intermediate and advanced steps including serverless architectures, DevOps methodologies (Infrastructure as Code via CloudFormation), and CI/CD pipelines [[09:19](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=559)].

### [[15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=900)] - [[30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=1800)] Core Service Domains & AWS Free Tier Account Setup

* **Deep Dive into the Pillars:** Explains the core categories of AWS services [[18:24](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=1104)]. It details Compute (the brain, like EC2 and Lambda), Storage (the memory, like S3, EBS, and EFS), Networking (the roots, like VPC and CloudFront), and Security (the guard, like IAM and KMS) [[18:56](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=1136)].
* **Account Creation Steps:** Walks through setting up an AWS Free Tier account from the registration console [[25:01](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=1501)].
* **Verification and Security:** Shows step-by-step instructions on verifying emails, setting strong root account passwords, and inserting necessary personal/billing profiles [[26:27](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=1587)].

### [[30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=1800)] - [[45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2700)] AWS Management Console, Latency Optimization, & Budgets

* **Navigating the Dashboard:** Takes a look at the AWS Management Console layout and walks through service directories categorized by application needs (such as Analytics, Developer Tools, and Machine Learning) [[31:21](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=1881)].
* **Region Selection & Latency:** Emphasizes how critical it is to choose the correct region (e.g., Mumbai or Northern Virginia) close to your user base to minimize latency [[35:35](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2135)]. The trainer warns against creating forgotten active resources in unmonitored regions to prevent unwanted costs [[36:21](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2181)].
* **Setting up a Zero Budget Alert:** Steps through the Billing and Cost Management console to create a proactive tracking alert [[38:29](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2309)]. Setting a customizable cost budget threshold of $1 helps notify you via email when Free Tier limits are exceeded [[40:05](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2405)].

### [[45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2700)] - [[01:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=3600)] IAM Identity & Access Management Best Practices

* **Authentication vs. Authorization:** Explains AWS Security through IAM (Identity and Access Management), defining its core mission as checking *who you are* and *what you are authorized to do* [[42:10](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2530)].
* **Corporate Analogy:** Relates users to employees, user groups to company departments, policies to job descriptions (e.g., JSON documents allowing or denying access), and roles to temporary permissions [[43:29](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2609)].
* **Default Deny Rule:** The trainer shares an essential interview question: IAM explicitly denies everything by default unless permissions are granted [[01:01:11](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=3671)].
* **Hands-on IAM Exercises:** Demonstrates creating a new "developer" IAM user, spinning up a "developers" user group, and attaching precise policy permissions (like VPC Full Access and EC2 Full Access) [[48:35](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=2915)]. It covers generating temporary AWS role policies and setting up multi-factor authentication (MFA) to lock down security profiles [[56:18](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=3378)].

### [[01:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=3600)] - [[01:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=4500)] Compute Frameworks & Launching an EC2 Virtual Linux Machine

* **Sizing up AWS Compute Options:** Transitions back to Compute, comparing Elastic Compute Cloud (EC2) instances against highly managed options like AWS Lightsail, Serverless Lambda architectures, and containerized platforms (ECS/EKS) [[01:02:09](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=3729)].
* **Deploying your First Server:** Demonstrates launching an EC2 virtual server from scratch [[01:06:02](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=3962)]. Steps look at filtering for Free Tier eligible Amazon Machine Images (AMIs), selecting compliant instance sizes (T2.micro), and creating critical authentication key-pair files (`.pem`) [[01:06:30](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=3990)].
* **Network & Security Groups:** Configures firewall rules inside the Network tab by allowing incoming web traffic via standard HTTP and HTTPS ports worldwide [[01:09:44](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=4184)].
* **Connecting over Console:** Connects directly into the running instance using the browser-based EC2 Instance Connect utility [[01:13:56](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=4436)].

### [[01:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=4500)] - [[01:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=5400)] Hosting a Live Web Server and S3 Static Storage Basics

* **Web Server Setup (Apache):** Inside the virtual terminal, uses Linux commands (`sudo yum update` and `sudo yum install httpd`) to mount an Apache web server [[01:14:16](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=4456)].
* **Deploying the Page:** Echoes a basic custom HTML greeting into the `index.html` root directory, starts up the system link, and verifies successful live deployment by loading the public IP address inside a browser tab [[01:15:55](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=4555)].
* **Clean-up Habit:** Emphasizes that you should immediately terminate the test instance so you do not use up accidental bills [[01:20:40](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=4840)].
* **Amazon Simple Storage Service (S3):** Introduces block object data structures (S3) for file backups and static asset hosting [[01:21:28](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=4888)]. It details setting public block configurations, managing bucket version histories, and handling default server-side encryptions during bucket configuration [[01:25:04](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=5104)].

### [[01:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=5400)] - [[01:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=6300)] Deploying a Static Site on S3 & RDS Database Provisioning

* **S3 Static Website Hosting:** Uploads an assortment of static asset folders (HTML, CSS, JavaScript, and images) into a newly uniquely-named S3 bucket [[01:30:03](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=5403)].
* **Bucket Policies:** Edits properties to enable static website hosting, updates the bucket permissions using an authorized public read access JSON policy, and loads the active root endpoint to show off a live web application [[01:31:25](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=5485)].
* **Relational Database Service (RDS):** Shifts from file objects to structured relational models, comparing manual database installation on single EC2 instances vs. fully managed RDS systems [[01:37:37](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=5857)].
* **Provisioning MySQL:** Configures a Free-Tier eligible MySQL database engine instance, assigns credential user profiles, allocates minimum storage quotas (8 GB), and connects it securely to the active EC2 compute environment [[01:40:02](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=6002)].

### [[01:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=6300)] - [[02:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=7200)] Global Infrastructure Nuances & Linux Connectivity Methods

* **Availability Zones & High Availability:** A new instructor deepens structural knowledge around global cloud foundations, explaining how regions map to distinct data centers called Availability Zones (AZs) [[01:47:02](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=6422)]. Spreading workloads across isolated AZs shields architectures from systemic single-point-of-failure downtimes [[01:48:34](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=6514)].
* **Instance Types Classification:** Breaks down EC2 structures into five optimized tracks: general purpose, memory-optimized, storage-optimized, accelerated computing, and compute-optimized types [[01:50:36](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=6636)].
* **Public vs. Elastic IPs:** Differentiates between standard temporary public IPs and static Elastic IP configurations linked to permanent account directories [[01:56:01](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=6961)].
* **Hands-on Virtualization Part 1:** Walks through launching a second virtual machine using an Ubuntu AMI on a T2.micro tier, building corresponding security key structures, and initializing a live workspace connection over the terminal console [[02:00:14](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=7214)].

### [[02:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=7200)] - [[02:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=8100)] Remote Workstations over Putty & Bootstrapping Windows Servers

* **Managing Third-Party Connectivity:** Explains how to link a local workstation to an active cloud server using external terminal clients like PuTTY [[02:10:53](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=7853)].
* **Key Conversions:** Uses the `PuTTYgen` system utility to load standard private encryption files (`.pem`) and convert them into PuTTY-compliant formats (`.ppk`) [[02:11:13](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=7873)]. It captures the instance's public IP vector and successfully logs into the cloud-hosted Ubuntu environment [[02:13:01](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=7981)].
* **Windows Instance Deployment:** Switches tracks to configure a non-Linux machine type by picking a Microsoft Windows Server AMI [[02:14:48](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=8088)]. It maps compatible hardware specs and downloads an active Remote Desktop file configuration tool [[02:18:48](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=8328)].

### [[02:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=8100)] - [[02:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=9000)] RDP Windows Connections & AMI Regional Replication

* **Accessing the Windows Cloud Desktop:** Opens the Remote Desktop Protocol (RDP) file payload, uploads local private encryption credentials to decrypt the host administrator password, and successfully runs a fully scalable virtualized Windows screen inside the desktop client browser [[02:19:19](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=8359)].
* **Replicating Server Architectures:** Teaches the logic behind copying an Amazon Machine Image (AMI) to preserve and replicate operational systems across distant boundaries [[02:24:27](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=8667)].
* **Cross-Region Migration:** Right-clicks on the active Ubuntu configuration, runs a snapshot backup creation query to generate a customized boot system, and initiates a cross-region copy protocol to clone that infrastructure straight into a secondary server hub (Oregon region) [[02:26:13](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=8773)].

### [[02:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=9000)] - [[02:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=9900)] Elastic Block Store (EBS) Performance and Storage Formatting

* **EBS Frameworks:** Defines Amazon Elastic Block Store (EBS) properties as fast, raw, unformatted network storage drives tied directly to individual instances [[02:30:56](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=9056)].
* **Throughput vs. IOPS:** Breaks down performance vectors by contrasting random small-tier data access tasks (optimized by high IOPS on solid-state drives) against massive high-speed streaming workloads (optimized by file throughput performance on standard mechanical hard drives) [[02:34:14](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=9254)].
* **Volume Creation and Attachment:** Creates a separate 5 GB baseline EBS data volume inside the designated AZ network and performs an active system mount attaching it to the active Linux instance framework [[02:43:55](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=9835)].
* **Linux Disk Allocation:** Enters the virtual terminal interface to trace disk properties (`lsblk`) and uses partition commands (`sudo mkfs -t ext4`) to structure the raw disk partition layout [[02:46:38](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=9998)].

### [[02:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=9900)] - [[03:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=10800)] Advanced Storage Mounting & Expanding Windows File Volumes

* **Mounting the File Directory:** Maps out a distinct structural folder destination inside the operational Linux system array (`sudo mkdir /ebs_volume`) and runs system instructions to finish binding the raw partition into a mounted working space [[02:50:41](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=10241)].
* **Unmounting the Volumes:** Covers clean device teardowns by executing a system detachment loop (`sudo umount`) [[02:53:50](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=10430)].
* **Windows Volume Expansion:** Replicates the network drive process for Windows users by appending a separate 10 GB storage device volume to a running Windows instance [[02:54:25](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=10465)].
* **Disk Management Tools:** Accesses internal administrative settings inside the Windows system screen, triggers the Disk Management configuration application, turns the new storage space online, format-maps it to an NTFS data tree framework, and designates a new functional local hard disk boundary (`D:` drive) [[02:57:42](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=10662)].

### [[03:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=10800)] - [[03:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11700)] EBS Snapshot Lifecycle Routines & Elastic Load Balancing (ELB)

* **Snapshot Backups:** Runs custom volume modifications to expand live storage partitions on the fly without system resets, demonstrating that cloud-mounted volumes can scale up but never decrease in total data size [[03:04:58](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11098)].
* **Incremental Backups via S3:** Creates data snapshot backups inside protected S3 data vaults to build automated system protection recovery loops [[03:05:55](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11155)]. It tests this by forcing a disk attachment wipe and successfully restoring files from the safe snapshot state [[03:08:11](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11291)].
* **Elastic Load Balancing Architecture:** Introduces automated web traffic routing, defining the Elastic Load Balancer (ELB) as a secure single-point-of-contact interface that monitors incoming request flows and evenly distributes requests across available, healthy server instances [[03:10:21](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11421)].
* **ELB Archetypes:** Compares Application Load Balancers (Layer 7 routing for regular web pages) against ultra-high performance Network Load Balancers (Layer 4 transport protocol handling for massive million-request traffic spikes) [[03:11:48](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11508)].

### [[03:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11700)] - [[03:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=12600)] Configuring High Availability Application Load Balancers

* **Target Group Topologies:** Walks through building an Application Load Balancer from scratch [[03:18:04](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11884)]. It provisions two distinct Ubuntu web nodes inside a default VPC network, assigning one as a primary Apache platform and the second to handle Nginx configurations [[03:18:12](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=11892)].
* **Health Check Routines:** Builds a custom Target Group array, sets baseline system tracking check protocols via regular HTTP loops, and hooks both independent web nodes into the monitoring frame [[03:20:58](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=12058)].
* **Simulating Alternate Routing Paths:** Finalizes the Application Load Balancer setup, attaches tracking networks across available subnets, and pulls the primary system DNS endpoint address [[03:24:08](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=12248)].
* **Round-Robin Verification:** Pastes the live endpoint inside a web browser window and runs consecutive browser updates to prove that it routes incoming traffic evenly between Apache and Nginx web servers [[03:29:28](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=12568)].

### [[03:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=12600)] - [[03:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13500)] Network Load Balancers (NLB) & Multi-AZ Infrastructure Concepts

* **Deploying Layer 4 Traffic Loops:** Configures an ultra-high performance Network Load Balancer (NLB) to handle high-throughput workloads [[03:30:23](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=12623)].
* **TCP Layer Pipelines:** Constructs a new high-speed target classification frame using explicit raw transport protocol layers (TCP port 80 loops) and hooks both active server nodes into the layout [[03:31:48](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=12708)].
* **Static Access Checking:** Extracts the raw NLB system connection string network path and runs browser verification loops to confirm successful connections [[03:35:32](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=12932)].
* **Infrastructure Design Nuances:** Re-introduces global environment structures, using visual chart frames to show how virtual workspaces draw physical resources from actual regional data centers [[03:40:32](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13232)].
* **Architecting for Latency:** Walks through selecting global node distribution networks based on target user distances to minimize cross-border latency and maximize site response speeds [[03:44:39](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13479)].

### [[03:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13500)] - [[04:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=14400)] Server Lifecycle States & Dynamic EBS Network Attachments

* **Stopped vs. Terminated State:** Explains how server charges change during lifecycle state transitions [[03:46:46](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13606)]. Terminating an instance deletes it completely and stops all related charges [[03:46:53](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13613)]. Stopping an instance freezes processing costs, but continues to incur minimal storage fees since the space remains reserved for you (similar to paying rent on an empty apartment) [[03:48:18](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13698)].
* **Mapping Storage Links:** Sets up custom storage mappings to attach separate EBS data links to virtual machine systems [[03:49:22](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13762)].
* **Internal vs. External Volumes:** Contrasts internal root volumes (which are deleted automatically when a server is terminated) against external network-attached volumes that keep your data safe even if the server crashes [[03:50:45](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=13845)].
* **Mounting Storage Links:** Demonstrates finding isolated volume blocks inside the instance dashboard directory tree [[04:01:25](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=14485)] and attaching them to working servers [[04:01:37](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=14497)].

### [[04:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=14400)] - [[04:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15300)] IAM Granular Control Frameworks & Group Permissions

* **Investigating Active Server Blocks:** Reviews volume allocation bounds, looking at maximum volume sizes (16 TB limits) [[03:56:11](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=14171)] and verifying cross-AZ mounting constraints (EBS volumes can only bind to servers running within their exact AZ) [[04:01:45](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=14505)].
* **Granular IAM User Switching:** Switches to the IAM console to test granular permissions, logging in with a restricted test profile ("John User") [[04:05:49](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=14749)].
* **Least Privilege Access Policies:** Demonstrates how permissions apply in real time by removing active S3 policy entries from the test profile, resulting in immediate access-denied errors [[04:08:46](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=14926)].
* **Scalable Team Group Management:** Introduces IAM Groups to streamline administration for large teams [[04:12:09](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15129)]. Instead of updating 20 developers individually, you add them to a single group and manage their access policies in one place [[04:12:51](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15171)].

### [[04:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15300)] - [[04:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=16200)] MFA Device Assignments & Cross-Service IAM Roles

* **Enforcing Two-Factor Security:** Sets up Multi-Factor Authentication (MFA) to secure access for IAM users [[04:16:04](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15364)]. It shows how to scan custom account QR codes using smartphone authenticator apps and verify consecutive security keys [[04:17:21](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15441)].
* **Cross-Service Communication Limitations:** Explains the core security rule for cross-service communication: by default, separate AWS services cannot interact with each other [[04:19:52](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15592)]. For instance, an EC2 server has no native permission to access files inside an S3 storage bucket.
* **Deploying IAM Roles:** Demonstrates creating an IAM Role to bridge this gap [[04:21:10](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15670)]. Unlike users (created for people), roles grant temporary, keyless permissions to AWS services so they can safely interact with one another [[04:21:34](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=15694)].

### [[04:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=16200)] - [[04:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=17100)] Resolving API Authorization Overlap & JSON Condition Trees

* **Debugging Console Errors:** Fixes permission issues when an IAM user attempts to attach a cross-service role to an EC2 instance but lacks authorization to view the account's roles [[04:30:23](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=16223)].
* **Inline Policy Customization:** Builds a custom Inline Policy to resolve the issue safely [[04:32:10](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=16330)]. Instead of granting risky broad administrative access, it follows the principle of least privilege by adding just one specific API permission (`listInstanceProfile`) using the visual editor [[04:32:42](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=16362)].
* **Resolving PassRole Constraints:** Adds the `PassRole` permission to authorize the service link, allowing the EC2 terminal to successfully list files from the target S3 bucket [[04:34:42](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=16482)].
* **Advanced JSON Custom Formatting:** Constructs a tailored S3 bucket policy with dual statement blocks [[04:37:19](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=16639)]. This allows full general access across the entire storage directory while explicitly denying delete actions on critical production buckets [[04:40:46](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=16846)].
* **Deny Override Rule:** Highlights an essential security rule: explicit deny rules always take precedence over allow rules, regardless of whether they are set at the user or group level [[04:43:34](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=17014)].

### [[04:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=17100)] - [[05:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18000)] CloudTrail Auditing & Auto Scaling Blueprint Models

* **Account-Wide Security Audits:** Introduces AWS CloudTrail as an unalterable activity log that automatically tracks and records all user and API actions across your account for up to 90 days [[04:47:36](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=17256)].
* **Auto Scaling Group Frameworks:** Explains the mechanics of Auto Scaling Groups (ASG), which automatically launch new instances during traffic surges and terminate extra servers when demand drops [[04:49:43](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=17383)].
* **Constructing Launch Templates:** Builds a custom Launch Template to act as a gold-standard configuration blueprint for the scaling group [[04:51:40](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=17500)]. It defines core parameters like the standard operating system (AMI), instance sizing, network connections, and custom bootstrap user data scripts [[04:52:31](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=17551)].
* **Multi-AZ Availability Configurations:** Combines the launch template with an active scaling group across multiple availability zones to ensure high availability and balanced server distribution [[04:57:28](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=17848)].

### [[05:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18000)] - [[05:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18900)] Scale Allocation Bounds & Virtual Private Cloud (VPC) Foundations

* **Managing Failures with Scaling Strategies:** Configures scaling behaviors, setting the system to route deployments to alternative healthy AZs if a launch fails in one zone [[05:00:40](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18040)].
* **Defining Scale Boundaries:** Explains how to set capacity boundaries to protect your infrastructure [[05:09:05](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18545)]. You configure a Minimum Capacity to maintain a baseline of running servers during low traffic, and a Maximum Capacity to cap instance creation and prevent runaway costs from DDoS traffic spikes [[05:09:11](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18551)].
* **Auto Scaling Group Teardowns:** Explains how resource dependencies behave during teardowns: deleting an Auto Scaling Group automatically terminates all the individual EC2 servers it created [[05:15:04](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18904)].
* **Virtual Private Cloud (VPC) Logic:** Explains Virtual Private Clouds (VPC) as isolated, private network boundaries that secure multi-tenant cloud environments [[05:16:30](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18990)]. This ensure your servers and data remain private and separated from other accounts, even when running on the same physical hardware [[05:18:34](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=19114)].

### [[05:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=18900)] - [[05:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=19800)] VPC Architecture Overviews & Classless Inter-Domain Routing (CIDR)

* **Custom VPC Strategies:** Explains that while AWS provides a default VPC in every region, you should build custom VPC architectures for real-world production environments to ensure tight security control [[05:22:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=19320)].
* **VPC Component Mapping:** Breaks down a standard VPC architecture into its key parts, including subnets, route tables, internet gateways, and routers [[05:24:25](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=19465)].
* **Subnet Allocations:** Covers subnets as logical subdivisions within a VPC network, split between Public Subnets (front-facing web apps) and Private Subnets (isolated databases) [[05:25:20](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=19520)].
* **CIDR IP Network Sizing:** Teaches Classless Inter-Domain Routing (CIDR) mapping to allocate IP address ranges [[05:40:08](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=20408)]. It explains how adjusting the prefix suffix changes your available IP host count (e.g., a `/16` prefix provides 65,536 addresses, while a `/24` prefix narrows the scope to 256 addresses) [[05:41:15](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=20475)].

### [[05:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=19800)] - [[05:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=20700)] Provisioning Custom VPC Networks & Subnet Designations

* **Building a Custom VPC:** Uses the AWS Console to provision a new custom VPC network [[05:48:36](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=20916)]. It maps out a baseline block size allocation (`10.0.0.0/16`) and sets the resource tenancy to default shared hardware [[05:49:24](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=20964)].
* **Subnet Sizing Calculations:** Uses an online CIDR calculator to divide the larger VPC address range into clean, non-overlapping subnet blocks [[05:59:34](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=21574)].
* **Creating Subnet Blocks:** Demonstrates creating explicit public and private subnet partitions within the targeted VPC, binding each to specific Availability Zones [[06:03:37](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=21817)].

### [[05:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=20700)] - [[06:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=21600)] Public Conversion Toggles & Reserved AWS Network Addresses

* **Converting Subnets to Public:** Converts a standard private subnet into a public subnet by enabling the auto-assign public IPv4 address setting [[06:10:16](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=22216)].
* **Verifying IP Deployments:** Validates the network layout by launching two test instances. The public subnet node receives both a public and private IP address, while the private subnet node receives only an internal private IP [[06:12:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=22320)].
* **Reserved IP Constraints:** Explains an essential networking rule: AWS automatically reserves 5 IP addresses within every single subnet for internal infrastructure tasks (including the network address, VPC router link, DNS server, future use space, and broadcast block) [[06:15:28](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=22528)].

### [[06:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=21600)] - [[06:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=22500)] Deploying Internet Gateways & Route Table Mapping Rules

* **Routing Traffic through Internet Gateways:** Introduces Internet Gateways (IGW) as the essential dual-direction link that allows custom VPC networks to connect to the public internet [[06:22:35](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=22955)].
* **Binding Gateways to VPCs:** Demonstrates creating a new Internet Gateway entity and binding it to your custom VPC network [[06:23:31](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23011)]. It highlights a key structural limitation: you can only attach one Internet Gateway to a VPC at a time [[06:24:45](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23085)].
* **Updating Route Tables:** Adds a default route (`0.0.0.0/0`) pointing directly to your Internet Gateway [[06:26:46](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23206)]. This provides instances with a clear network path to connect to the internet through the newly attached doorway [[06:28:26](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23306)].

### [[06:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=22500)] - [[06:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23400)] Multi-Tier Application Topologies & Stateless Network ACLs (NACL)

* **Structuring Multi-Tier Architectures:** Uses a banking application as an example to explain classic 3-tier architecture design [[06:30:33](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23433)]. Front-facing web servers sit in public subnets to accept user requests, while core backend applications and sensitive databases remain safely isolated inside private subnets [[06:32:05](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23525)].
* **Two-Layer Firewall Defense:** Compares the two types of network security firewalls in AWS: Security Groups and Network Access Control Lists (NACL) [[06:35:28](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23728)].
* **NACL vs. Security Groups:** Details how firewalls handle traffic [[06:36:31](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23791)]. Security Groups act as a firewall at the instance level, while NACLs act as a firewall at the subnet level. This dual-layer approach provides a secure, layered defense-in-depth model [[06:37:34](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23854)].
* **Explicit Deny Configuration Rules:** Explains that while Security Groups can only allow traffic, NACLs give you the flexibility to configure explicit allow and deny rules for specific IP addresses [[06:40:08](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=24008)].

### [[06:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=23400)] - [[06:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=24300)] Stateful Security Operations & Custom Customer VPC Bounds

* **Stateful vs. Stateless Firewalls:** Compares firewall behaviors [[06:48:40](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=24520)]. Security Groups are stateful firewalls, meaning they automatically allow return traffic for any approved incoming request without needing an explicit outbound rule [[06:49:12](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=24552)]. Conversely, NACLs are stateless firewalls, meaning they do not track connection states and require you to configure explicit rules for both incoming and outgoing traffic [[06:49:37](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=24577)].
* **VPC Suffix Allocation Rules:** Outlines AWS sizing rules for custom VPC networks, which restrict network allocations to prefix scales between a minimum of `/28` and a maximum of `/16` [[06:51:39](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=24699)].
* **Building a Custom VPC:** Uses the VPC Wizard to configure a custom VPC environment [[06:53:56](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=24836)], defining its prefix boundary range (`10.0.0.0/16`) and selecting a shared network tenancy layout [[07:02:26](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=25346)].

### [[06:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=24300)] - [[07:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=25200)] Departmental Data Splitting & Subnet Boundary Logic

* **Network Segmentation Analogy:** Uses a corporate office layout analogy to explain network segmentation, showing how grouping desks by department keeps operations organized and prevents cross-department traffic chaos [[07:08:20](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=25700)].
* **Preventing Broadcast Congestion:** Explains how subdividing large, unmanageable networks into smaller subnet pools prevents broadcast noise and keeps data traffic flows clean and isolated [[07:14:53](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=26093)].
* **Mapping Subnets to AZs:** Outlines subnet scoping rules: while a VPC spans an entire AWS region, individual subnets are scoped to a single Availability Zone [[07:19:47](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=26387)].

### [[07:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=25200)] - [[07:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=26100)] Multi-Tier Isolation Frameworks & Decoupling Database Stacks

* **Deconstructing Multi-Tier Layouts:** Explains the risks of running web servers, applications, and databases on a single host machine, as a public breach on the web layer exposes your entire database stack [[07:30:33](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27033)].
* **Decoupling Application Stacks:** Demonstrates how decoupling components into distinct layers—web presentation, business application logic, and backend database storage—improves security by restricting cross-layer communication to specific ports [[07:34:50](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27290)].
* **Securing Sensitive Financial Data:** Explains how hosting databases in private subnets protects sensitive financial data from public internet exposure by routing traffic securely through intermediate application layers [[07:36:20](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27380)].

### [[07:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=26100)] - [[07:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27000)] Architecting High Availability Multi-Account Networks

* **Designing Multi-Account Environments:** Explains that modern cloud designs isolate environments (like development and production) across separate AWS accounts, rather than just using basic VPC filtering within a single account [[07:44:50](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27890)].
* **High Availability Subnet Topologies:** Maps out a high-availability subnet layout across two distinct Availability Zones, replicating web, application, and database subnets in both zones to ensure operational continuity [[07:46:32](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27992)].
* **Subnet Provisioning in Action:** Provisions custom subnet blocks in the AWS Console, mapping explicit CIDR blocks to targeted zones and showing how internal network allocation rules reserve five IP addresses from the pool [[07:24:38](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=26678)].

### [[07:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27000)] - [[07:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27900)] Calculating IP Sizing Requirements & Local VPC Routing Paths

* **Forecasting Long-Term Growth:** Walks through calculating network sizing requirements for real-world projects, taking into account initial instance footprints, estimated growth over a 5-year window, and safety padding buffers [[07:53:33](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=28413)].
* **Mapping Custom Suffix Scales:** Explains how to select the right subnet prefixes (such as `/27` for 32 IPs or `/26` for 64 IPs) to fit your forecasted growth targets [[07:57:47](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=28667)].
* **Default Local Routing Logic:** Introduces default route tables, explaining how the built-in local route entry (`10.0.0.0/16 local`) automatically allows all subnets within the VPC to communicate with each other by default [[08:12:01](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=29521)].

### [[07:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=27900)] - [[08:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=28800)] Linking Custom Route Tables & Internet Gateway Attachments

* **Isolating Network Routing Rules:** Explains why you should create separate custom route tables instead of relying on the default main route table, allowing you to manage routing rules independently for your public and private subnets [[08:18:48](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=29928)].
* **Deploying Internet Access Doorways:** Recreates the process of building an Internet Gateway and attaching it to a custom VPC network to provide an explicit internet connection path [[08:32:06](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=30726)].
* **Configuring Public Routes:** Adds a catch-all route (`0.0.0.0/0`) pointing to the newly attached Internet Gateway to enable internet connectivity for resources in public subnets [[08:35:37](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=30937)].

### [[08:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=28800)] - [[08:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=29700)] Binding Subnets to Route Tables & Defining VPC Peering Links

* **Activating Public Routing Channels:** Explains that a subnet becomes public when you explicitly bind it to a custom route table that has a default route to an Internet Gateway [[08:40:26](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=31226)].
* **Private Subnet Isolation Rules:** Shows that if a route table lacks a route to an Internet Gateway, any subnets bound to it remain private and isolated from the public internet [[08:41:23](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=31283)].
* **VPC Peering Interconnections:** Introduces VPC Peering as a private, high-speed routing connection that allows instances in separate VPC networks to communicate securely using private IP addresses [[08:47:59](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=31679)].

### [[08:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=29700)] - [[08:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=30600)] Handshaking VPC Peering Requests & Cross-Region Routing Channels

* **Resolving Network Address Overlaps:** Highlights a critical prerequisite for VPC Peering: you cannot peer two VPC networks if their CIDR blocks overlap, as this causes IP routing conflicts [[09:05:13](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=32713)].
* **Handshaking Peering Links:** Demonstrates establishing a VPC Peering link by sending a connection request from the initiator VPC and accepting it from the target VPC within the console [[09:09:45](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=32985)].
* **Cross-VPC Route Propagation:** Updates both route tables with explicit routes pointing to the peered network block via the peering connection ID (`pcx-...`), enabling secure, private communication between the two VPCs [[09:14:54](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=33294)].

### [[08:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=30600)] - [[08:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=31500)] Peering Mesh Scalability Bottlenecks & NAT Gateway Foundations

* **Peering Mesh Limits:** Explains that while transitive peering isn't natively supported, you can scale connections by creating full-mesh peering links between multiple VPC networks [[09:21:05](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=33665)]. However, as your network grows, managing a complex web of individual peering links becomes difficult, which is when you should transition to a centralized Transit Gateway hub architecture [[09:26:27](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=33987)].
* **NAT Gateway Architecture:** Introduces Network Address Translation (NAT) Gateways, which allow resources in private subnets to securely connect outbound to the internet (e.g., for software patches) while blocking any unsolicited inbound connections from the public internet [[09:27:55](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=34075)].
* **NAT Analogy:** Relates a NAT Gateway to a healthy family member picking up food for an isolated, sick relative; it handles external requests on behalf of the private resource without exposing its identity [[09:28:34](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=34114)].

### [[08:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=31500)] - [[09:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=32400)] Source Network Address Translation (SNAT) Protocols & Private Proxies

* **Hiding Private IP Identities:** Explains how a NAT Gateway handles traffic using Source Network Address Translation (SNAT) [[09:40:04](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=34804)]. It strips out the private source IP address of the internal instance and substitutes its own public IP to route requests to the internet, keeping your internal infrastructure hidden from public view [[09:40:36](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=34836)].
* **Deploying NAT Gateways:** Provisions a new public NAT Gateway in the AWS Console, ensuring it runs within a public subnet so it has an outbound path to the internet gateway [[09:44:29](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=35069)].
* **Allocating Static Elastic IPs:** Binds a static Elastic IP address to the NAT Gateway to ensure it uses a consistent, unchanging public IP for all outbound internet requests [[09:46:10](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=35170)].

### [[09:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=32400)] - [[09:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=33300)] Routing Traffic Through NAT Gateways & Stateful Packet Evaluation

* **Private Subnet Route Redirection:** Updates your private subnet's route table, adding a default route (`0.0.0.0/0`) that redirects all outbound internet requests directly through the newly created NAT Gateway [[09:49:32](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=35372)].
* **Tracing Inbound Network Traffic Flows:** Traces an inbound network traffic path from the internet through the Internet Gateway, where it passes through the stateless subnet NACL rules first before being evaluated by stateful instance Security Groups [[10:13:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=36780)].
* **Layered Network Defense Architectures:** Explains how combining instance-level Security Groups with subnet-level Network Access Control Lists (NACL) creates a secure, layered defense-in-depth model [[10:15:22](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=36922)].

### [[09:15:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=33300)] - [[09:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=34200)] Stateless NACL Processing Mechanics & Portfolio Project Scoping

* **Processing Stateless Firewall Rules:** Details how a stateless NACL processes traffic, requiring you to configure explicit rules for both inbound traffic and outbound response paths to ensure smooth communication [[10:29:54](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=37794)].
* **Elastic Network Interface (ENI) Mappings:** Explains that instance-level Security Groups are bound directly to virtual network cards called Elastic Network Interfaces (ENIs), rather than just being wrapped around the instance as a whole [[10:35:35](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=38135)].
* **Portfolio Project Overview:** Launches a practical, hands-on capstone project: building and deploying an automated, responsive portfolio website on AWS using an automated CI/CD deployment pipeline [[10:41:13](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=38473)].

### [[09:30:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=34200)] - [[09:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=35100)] Designing GitHub Actions Pipelines & YAML Schema Configurations

* **Automated CI/CD Pipelines:** Outlines the automated deployment workflow: developers push website code updates to a GitHub repository, which triggers a GitHub Actions pipeline to automatically build, sync, and deploy the updated files directly to an S3 bucket [[10:45:51](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=38751)].
* **Configuring GitHub Workflows:** Creates a customized GitHub Actions workflow configuration file (`.github/workflows/deploy.yml`) within the code repository [[10:51:28](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=39088)].
* **Deconstructing Workflow Code Blocks:** Explains the core components of the configuration file, including the main branch push trigger, the virtual runner environment (Ubuntu), and the deployment step that uses an S3 sync utility to update bucket files [[10:53:34](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=39214)].

### [[09:45:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=35100)] - [[11:09:24](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=40164)] Activating S3 Static Site Hosting & Real-Time Code Sync Verifications

* **Provisioning the Hosting Bucket:** Provisions a target S3 bucket in the AWS Console, disables the public access blocks, and enables bucket versioning to track file changes [[10:56:49](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=39409)].
* **Generating IAM Automation Keys:** Creates a dedicated IAM automation user profile, grants it full S3 management permissions, and generates secure access keys to authenticate the deployment pipeline [[10:58:23](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=39503)].
* **Securing Environment Secrets:** Configures the access keys and target bucket name as encrypted repository secrets inside settings to ensure secure authentication [[11:00:00](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=39600)].
* **Enabling Public Web Access:** Configures the S3 bucket properties to enable static website hosting, sets the default home page to `index.html`, and attaches an open public read bucket policy to allow public web access [[11:01:40](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=39700)].
* **Testing Live Pipeline Updates:** Tests the automated pipeline by modifying the portfolio title string in the source repository code and pushing the commit to GitHub [[11:04:54](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=39894)]. The pipeline detects the update, triggers the workflow runner to sync files to S3, and reflects the changes live on the hosted portfolio website in real time [[11:05:58](https://www.youtube.com/watch?v=N3QYOjSXxiU&t=39958)].
