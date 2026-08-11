This video explains how to build a **serverless CI/CD pipeline** on *AWS* that deploys a web application to an *EC2* instance without using *SSH* keys, *Jenkins*, or *GitHub Actions* (0:36-0:45). 

**Key components of the architecture include:**
* **S3 Bucket:** Acts as the "Dropbox" for code artifacts; uploading a zip file triggers the entire process (0:56-1:01, 5:00-5:25).
* **Lambda Function:** Serves as the "brains" of the pipeline, triggered by the *S3* upload to send deployment instructions (3:55-4:33).
* **Systems Manager:** Acts as the delivery service, sending commands to the *EC2* instance through a secure outbound connection on port 443, eliminating the need for inbound management ports (2:32-2:59).
* **EC2 Instance:** A hardened server running *Nginx* that uses an *IAM* role to pull the code directly from *S3* and execute the deployment (1:34-1:57, 6:00-6:15).

**Security highlights:**
* The server operates with **no SSH key pair**, shifting administration to *AWS* identity-based roles (2:07-2:25, 3:43-3:53).
* By using *Systems Manager* with a pre-installed agent, the server remains isolated from external access, keeping the "blast radius" tiny (2:45-2:55, 3:39-3:42).

The narrator emphasizes that this is a repeatable pattern—not a product—consisting of an event, glue code, and an agent (7:07-7:16). The video concludes by inviting viewers to try the provided hands-on labs to build this pipeline themselves (7:31-8:06).


https://www.youtube.com/watch?v=evU0d1mVfOw
