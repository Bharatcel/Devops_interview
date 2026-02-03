# Chapter 12: Security and DevSecOps - Shifting Left

## Abstract

In modern software development, security is not a separate phase or the responsibility of a separate team; it is an integral part of the entire development lifecycle. This is the core philosophy of DevSecOps. The goal is to "shift left"—to move security practices from the end of the pipeline (just before release) to the very beginning (at the developer's workstation). For a DevOps engineer, this means being a security champion. It requires a deep understanding of common vulnerabilities, the tools and techniques to detect them automatically, and the ability to architect secure infrastructure and deployment pipelines from the ground up. This chapter provides that definitive, book-level expertise, embedding security into every stage of the DevOps process.

---

### Part 1: The DevSecOps Philosophy - Security as Code

*   **The Old Way (The "Security Gate"):** In traditional models, the security team would perform a penetration test or a code audit just before a release. If they found vulnerabilities, the release would be blocked, and the code would be thrown back over the wall to the developers, causing massive delays and friction.

*   **The New Way (DevSecOps):** DevSecOps is about integrating security practices into the existing DevOps pipeline. The goal is to make security an automated, continuous, and collaborative effort.
    *   **Automation:** Replace manual security reviews with automated security testing tools that run inside the CI/CD pipeline.
    *   **Collaboration:** Break down the silos between development, security, and operations teams.
    *   **Empowerment:** Give developers the tools and knowledge to find and fix vulnerabilities themselves, providing them with fast feedback directly in their workflow.

---

### Part 2: Security in the CI/CD Pipeline - A Multi-Layered Approach

A secure pipeline integrates different types of security testing at different stages.

1.  **Pre-Commit (The Developer's Workstation):**
    *   **What:** The shift left starts here. Developers can run tools locally to catch issues before they are ever committed to version control.
    *   **Tools:**
        *   **Pre-commit hooks:** Git hooks can be configured to run scripts that scan for secrets (like AWS keys) before allowing a commit. Tools like `talisman` or `git-secrets` are excellent for this.
        *   **IDE Plugins:** Plugins for IDEs like VS Code can provide real-time feedback on security issues as the developer is writing code.

2.  **Commit/Build Stage (Continuous Integration):**
    *   This is where the bulk of automated security testing happens. When a developer pushes code, the CI server triggers a series of scans.
    *   **Static Application Security Testing (SAST):**
        *   **What it is:** SAST tools analyze the application's source code from the "inside out" to find potential vulnerabilities like SQL injection, cross-site scripting (XSS), and insecure cryptographic practices. They work like a sophisticated linter for security.
        *   **Examples:** SonarQube, Snyk Code, Checkmarx.
    *   **Software Composition Analysis (SCA):**
        *   **What it is:** Modern applications are built on open-source dependencies. SCA tools scan your dependency files (`package.json`, `pom.xml`, `requirements.txt`) and identify any components with known CVEs (Common Vulnerabilities and Exposures). This is absolutely critical, as a huge percentage of breaches originate from vulnerable third-party libraries.
        *   **Examples:** Snyk Open Source, OWASP Dependency-Check, Dependabot (built into GitHub).
    *   **Container Scanning:**
        *   **What it is:** As discussed in the Docker chapter, tools like Trivy, Clair, or Snyk Container scan your Docker images for known vulnerabilities in the base OS and system libraries. This should be done at build time.

3.  **Test/Staging Stage (Continuous Delivery):**
    *   After the application is deployed to a running environment, you can perform tests from the "outside in."
    *   **Dynamic Application Security Testing (DAST):**
        *   **What it is:** DAST tools attack a running application, just like a real hacker would. They probe the application's endpoints for vulnerabilities like XSS, SQL injection, and insecure server configuration. They have no knowledge of the source code.
        *   **Examples:** OWASP ZAP (Zed Attack Proxy), Burp Suite.

4.  **Production Stage (Continuous Monitoring):**
    *   Security doesn't stop at deployment.
    *   **Runtime Security:** Tools like Falco can detect anomalous behavior within your production containers at runtime (e.g., a shell being spawned in a container, or unexpected network connections).
    *   **Cloud Security Posture Management (CSPM):** Tools that continuously monitor your cloud environment (like AWS) for misconfigurations that violate security best practices (e.g., an S3 bucket being made public, or a security group opening SSH to the world). Examples include AWS Security Hub and Prisma Cloud.

---

### Part 3: Core Security Concepts for Infrastructure

#### Secrets Management: The Crown Jewels

*   **The Problem:** Never, ever, ever store secrets (database passwords, API keys, TLS certificates) in plaintext in a Git repository, a configuration file, or a Docker image.
*   **The Solution: A Secret Vault**
    *   A dedicated secrets management tool provides a central, secure place to store and manage secrets.
    *   **HashiCorp Vault** is the industry standard.
        *   **How it works:**
            1.  **Secure Storage:** Secrets are encrypted at rest.
            2.  **Dynamic Secrets:** This is Vault's killer feature. Instead of storing a static, long-lived database password, you can configure Vault to generate a **new, temporary** database username and password every time an application needs one. These credentials can be configured to expire after a short period (e.g., one hour). This dramatically reduces the risk of a leaked credential.
            3.  **Authentication:** Applications don't just get to ask for secrets. They must first authenticate with Vault to prove their identity. In Kubernetes, an application pod can use its Service Account Token to authenticate with Vault. In AWS, an EC2 instance can use its IAM Role.
            4.  **Audit Trail:** Every action in Vault is logged, so you have a clear audit trail of who accessed what secret and when.

#### Network Security in the Cloud

*   **Security Groups (The Instance Firewall):** Act as a stateful firewall at the instance level. The best practice is to be as restrictive as possible. For example, a web server's security group should only allow inbound traffic on port 443 from the Application Load Balancer's security group, and nothing else.
*   **Network ACLs (The Subnet Firewall):** Act as a stateless firewall at the subnet level. They provide a second layer of defense.
*   **Web Application Firewall (WAF):**
    *   **What it is:** A firewall that operates at Layer 7 (Application). It inspects incoming HTTP/S traffic and can block common web-based attacks like SQL injection and cross-site scripting before they even reach your application.
    *   **Example:** **AWS WAF**, which can be attached to an Application Load Balancer or CloudFront distribution.

#### The Principle of Least Privilege (PoLP)

*   This is the most fundamental concept in all of security.
*   **What it means:** Any user, program, or system should have only the bare minimum permissions required to perform its function, and nothing more.
*   **Practical Examples:**
    *   An application that only needs to read from an S3 bucket should have an IAM policy that only grants `s3:GetObject`, not `s3:*`.
    *   A CI/CD pipeline that deploys to a staging environment should have credentials that only grant access to staging, not production.
    *   A container should be run as a non-root user.

---

### ★ FAANG-Level Interview Scenarios ★

*   **Scenario 1: The Leaked AWS Key**
    *   **Question:** "You are on-call. At 2 AM, you receive a notification from GitHub that an AWS access key has been found in a public repository belonging to your company. What is your immediate, step-by-step incident response plan?"
    *   **Answer:** "This is a critical security incident, and speed is paramount. My response plan follows the 'Identify, Contain, Eradicate, Recover' model.
        1.  **Containment (Immediate Priority):** The absolute first step is to **immediately disable the leaked access key** in the AWS IAM console. This stops the bleeding and prevents any further unauthorized access. I do not need to delete the key yet, just deactivate it. This is the most critical action and must be done within minutes.
        2.  **Identification:** Now that the immediate threat is contained, I need to understand the blast radius. I will use the IAM user associated with the key to check its permissions. What services and resources could it access? I will then go to **AWS CloudTrail** and look for any API activity from that access key. I need to determine:
            *   Has the key been used by anyone other than its intended owner?
            *   What actions were performed? Was data exfiltrated? Were new resources created?
        3.  **Eradication:**
            *   I will now **delete the compromised access key**.
            *   I will work with the developer who committed the key to remove it from the Git history. The correct way to do this is using a tool like the **BFG Repo-Cleaner** or `git filter-repo`, not just reverting the commit, as the key would still exist in the history. This is a destructive action that requires coordination.
            *   I will also immediately rotate any other credentials that might have been exposed by the attacker's actions (e.g., if they used the key to read a database password from Secrets Manager).
        4.  **Recovery & Post-Mortem:**
            *   I will write a detailed post-mortem report.
            *   The key follow-up actions would be to implement preventative measures. I would set up a **pre-commit hook** using `git-secrets` or `talisman` on all developer machines to scan for secrets before they can even be committed. I would also ensure our CI/CD pipeline has a secret scanning step to act as a second line of defense."

*   **Scenario 2: Designing a Secure CI/CD Pipeline**
    *   **Question:** "You are designing a CI/CD pipeline for a containerized application that will be deployed to Amazon EKS. Describe the security controls you would embed at each stage of the pipeline."
    *   **Answer:** "My goal is to build a 'DevSecOps' pipeline that automates security at every step.
        1.  **CI Stage (on every Pull Request):**
            *   **SAST:** I'll integrate a Static Application Security Testing tool like **SonarQube** to scan the source code for vulnerabilities. The PR build will fail if it finds any critical issues.
            *   **SCA:** I'll use **Snyk** or **Dependabot** to scan all open-source dependencies for known CVEs. The build will fail if a new high-severity vulnerability is introduced.
            *   **Container Scanning:** During the `docker build` step, I will use **Trivy** to scan the base image and any installed packages for vulnerabilities.
        2.  **Artifact Repository Stage:**
            *   The built image will be pushed to **Amazon ECR**. I will enable ECR's built-in vulnerability scanning to re-scan the image whenever new CVE information is published.
            *   I will also implement **image signing** using a tool like Notary or Sigstore. This cryptographically signs the image, and I can configure the EKS cluster to only allow signed images to be deployed.
        3.  **CD Stage (Deployment):**
            *   **Secrets Management:** The application will be configured to fetch its secrets (database passwords, API keys) at runtime from **HashiCorp Vault** or **AWS Secrets Manager**. The Kubernetes Service Account for the pod will be granted permission to access only the specific secrets it needs, using Vault's Kubernetes Auth Method or IAM Roles for Service Accounts (IRSA) in AWS. There will be zero secrets in the container image or environment variables.
            *   **DAST:** After deploying to a staging environment, the pipeline will trigger a **DAST** scan using **OWASP ZAP** to probe the running application for any runtime vulnerabilities.
        4.  **Runtime (In the EKS Cluster):**
            *   **Network Policies:** I will use Kubernetes Network Policies to enforce a "zero-trust" network. By default, pods will not be able to communicate with each other. I will create explicit policies to allow only the required traffic (e.g., allow the `frontend` pods to talk to the `backend` pods on port 8080, and nothing else).
            *   **Runtime Security:** I will deploy **Falco** as a DaemonSet to monitor for anomalous behavior within the running containers and alert on suspicious activity."

*   **Scenario 3: Secrets in Containers**
    *   **Question:** "A developer wants to pass a database password to a container. They suggest using an environment variable (`-e DB_PASSWORD=...`). Why is this not a secure method, and what are the better alternatives in Kubernetes?"
    *   **Answer:** "Using environment variables for secrets is a common but insecure pattern.
        *   **The Problem:** Environment variables are easily exposed. Anyone who can `exec` into the container can run `env` and see the secret in plaintext. More importantly, if the application crashes, the environment variables might be captured in a crash dump or log file. In Kubernetes, anyone with permission to view the Pod's definition (`kubectl describe pod ...`) can see the environment variables in plaintext.
        *   **The Better Alternatives in Kubernetes:**
            1.  **Kubernetes Secrets (The Good Way):** The standard way is to store the password in a Kubernetes `Secret` object. Then, you can expose this secret to the pod in one of two ways:
                *   **As an environment variable:** `envFrom: secretRef: name: my-secret`. This is better than putting the secret directly in the deployment YAML, but it still has the same risk of the environment variable being exposed within the container.
                *   **As a file mounted into the container (The Better Way):** You can mount the secret as a file into a temporary filesystem (`tmpfs`) in the container (e.g., at `/etc/secrets/password`). The application can then read the password from this file. This is more secure because the secret is not part of the process environment, and you can use standard file permissions to restrict access to it.
            2.  **External Vault (The Best Way):** The most secure, enterprise-grade solution is to use an external secrets manager like **HashiCorp Vault** or **AWS Secrets Manager**. The application, upon starting, would authenticate to the vault using its Kubernetes Service Account identity and fetch the secret directly. This provides a full audit trail, centralized management, and the ability to use dynamic, short-lived secrets, which is the gold standard for security."
        ```