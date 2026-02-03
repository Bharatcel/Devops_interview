# Chapter 16: The Final Project - Building a FAANG-Level DevOps Platform from Scratch

## Abstract

Theory is essential, but practical application is what separates a good engineer from a great one. This final chapter is a capstone project that synthesizes everything we have learned into a single, cohesive, real-world system. We will build a complete, production-grade DevOps platform for a microservices application. Starting with nothing but an application idea, we will:

1.  **Containerize** the application services with Docker.
2.  **Provision cloud infrastructure** (a Kubernetes cluster and a container registry) using Terraform.
3.  **Build a secure CI pipeline** with GitHub Actions that tests, scans, and builds our code.
4.  **Implement a GitOps-based CD pipeline** with Argo CD for safe, automated deployments.
5.  **Add monitoring and observability** with Prometheus and Grafana.

This project is not a simple tutorial; it is a blueprint for a modern software delivery platform that reflects the best practices used at top-tier tech companies. Completing this project will provide the hands-on experience needed to confidently discuss system design and implementation in any DevOps interview.

---

### Project Overview: The "Pollr" Application

We will build a simple two-tier web application:

*   **`pollr-api` (The Backend):** A Node.js/Express microservice that provides a simple REST API to vote on a poll and view the results. It will store the poll data in memory.
*   **`pollr-web` (The Frontend):** A simple Node.js/Express app that serves a static HTML page. This page will use JavaScript to make AJAX calls to the `pollr-api` service to cast votes and display the results.

This simple architecture is perfect for demonstrating the complex interactions in a microservices environment.

---

### Phase 1: Local Development & Containerization

**Goal:** Get the application running on your local machine using Docker.

1.  **Application Code:**
    *   Create a new monorepo directory. Inside, create two folders: `pollr-api` and `pollr-web`.
    *   **`pollr-api/server.js`:** Create a simple Express app with two endpoints:
        *   `GET /results`: Returns the current poll results as JSON (e.g., `{ "optionA": 10, "optionB": 20 }`).
        *   `POST /vote`: Takes a vote for an option (e.g., `{"option": "A"}`).
    *   **`pollr-web/server.js`:** Create an Express app that serves an `index.html` file.
    *   **`pollr-web/index.html`:** Create a basic HTML page with buttons to vote for "Option A" and "Option B" and a section to display the results. Write the JavaScript to call the `pollr-api` endpoints.

2.  **Dockerfiles:**
    *   Create a `Dockerfile` in both the `pollr-api` and `pollr-web` directories. These will be standard Node.js Dockerfiles, following the best practice of using multi-stage builds to create small, secure production images.

    ```dockerfile
    # Example Dockerfile for pollr-api
    FROM node:16-alpine AS builder
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    COPY . .
    # Add a step here to run unit tests if you have them

    FROM node:16-alpine
    WORKDIR /app
    COPY --from=builder /app/package*.json ./
    RUN npm install --production
    COPY --from=builder /app .
    CMD ["node", "server.js"]
    ```

3.  **Docker Compose:**
    *   Create a `docker-compose.yml` file in the root of the monorepo. This file will define the two services and the network connecting them. This allows you to spin up the entire application stack with a single command: `docker-compose up`.

    ```yaml
    version: '3.8'
    services:
      pollr-api:
        build: ./pollr-api
        ports:
          - "8081:8080" # Expose API on host port 8081
      pollr-web:
        build: ./pollr-web
        ports:
          - "8080:8080" # Expose web on host port 8080
    ```

**Outcome:** You can now run `docker-compose up --build` and access the web application at `http://localhost:8080`.

---

### Phase 2: Infrastructure as Code with Terraform

**Goal:** Provision the cloud infrastructure needed to run our application.

1.  **Setup:**
    *   Create a new `terraform` directory in your project.
    *   Configure the AWS provider.

2.  **Provision EKS (Elastic Kubernetes Service):**
    *   Use the official Terraform EKS module to provision a production-grade Kubernetes cluster. This module handles the complexity of setting up the control plane, worker nodes, VPC, subnets, and IAM roles.

3.  **Provision ECR (Elastic Container Registry):**
    *   Create two ECR repositories using Terraform, one for `pollr-api` and one for `pollr-web`. These are where our CI pipeline will push the Docker images.

**Outcome:** After running `terraform apply`, you will have a running Kubernetes cluster in AWS and two container registries ready to receive images.

---

### Phase 3: The Secure CI Pipeline with GitHub Actions

**Goal:** Create a CI pipeline that automatically builds, tests, and scans our application code.

1.  **Repository Setup:**
    *   Push your monorepo to a new GitHub repository.
    *   Configure AWS credentials as secrets in your GitHub repository settings so that GitHub Actions can access your AWS account.

2.  **GitHub Actions Workflow:**
    *   Create a `.github/workflows/ci.yml` file. This workflow will be triggered on every push to the `main` branch.
    *   **The CI workflow will have the following jobs:**
        1.  **Lint & Test:** A job that runs `npm install` and then runs any linters or unit tests.
        2.  **Security Scan (SCA):** Use the official Snyk GitHub Action to scan your open-source dependencies for vulnerabilities.
        3.  **Build & Push:** A job that:
            *   Logs into the AWS ECR registry.
            *   Builds the Docker images for `pollr-api` and `pollr-web`.
            *   **Crucially, it tags the images with the Git commit SHA.** This is how we link a specific version of the code to a specific Docker image.
            *   Pushes the tagged images to their respective ECR repositories.
        4.  **Container Scan:** After the images are pushed, use a tool like Trivy to scan the final images for OS-level vulnerabilities.

**Outcome:** When you push a code change, a fully automated pipeline will run. If all steps pass, two new, versioned, and scanned Docker images will be available in ECR.

---

### Phase 4: GitOps-based CD with Argo CD

**Goal:** Set up a fully automated, pull-based deployment pipeline.

1.  **GitOps Repository:**
    *   Create a **new, separate** GitHub repository. This will be your GitOps repository.
    *   Inside this repo, create the Kubernetes manifest files for the application:
        *   `pollr-api-deployment.yaml`
        *   `pollr-api-service.yaml`
        *   `pollr-web-deployment.yaml`
        *   `pollr-web-service.yaml`
    *   In the Deployment manifests, use a placeholder for the image tag, e.g., `image: <your-ecr-repo-url>:latest`.

2.  **Install and Configure Argo CD:**
    *   Install the Argo CD CLI and install Argo CD onto your EKS cluster.
    *   Configure Argo CD to watch your new GitOps repository. Create an "Application" in Argo CD that points to the path containing your manifests.
    *   Argo CD will now automatically "sync" and deploy the application to your cluster.

3.  **Connecting CI to CD:**
    *   This is the final, critical step. We need to update the CI pipeline from Phase 3.
    *   Add a new job to the end of your `ci.yml` workflow. This job will run only after the images have been successfully built and pushed.
    *   This job will:
        1.  Check out the **GitOps repository**.
        2.  Use a tool like `kustomize` or `sed` to update the image tag in the `pollr-api-deployment.yaml` and `pollr-web-deployment.yaml` files with the **Git commit SHA** from the build.
        3.  Commit and push this change back to the GitOps repository.

**The Complete Flow:**
1.  A developer pushes code to the application repository.
2.  The CI pipeline runs, builds, and pushes a new image tagged with the commit SHA (e.g., `...:a1b2c3d`).
3.  The CI pipeline checks out the GitOps repo, changes the image tag in the deployment YAML to `a1b2c3d`, and pushes the change.
4.  Argo CD, running in the cluster, detects the change in the GitOps repo.
5.  Argo CD pulls the new manifest and applies it to the cluster, triggering a rolling update of the deployment with the new image.

**Outcome:** You have a fully automated, end-to-end, GitOps-driven software delivery pipeline. A `git push` to the app repo results in a safe, automated deployment to production.

---

### Phase 5: Monitoring and Observability

**Goal:** Add basic monitoring to our application.

1.  **Install Prometheus and Grafana:**
    *   Use the `kube-prometheus-stack` Helm chart to easily install a full monitoring stack onto your cluster. This is the industry-standard way to do it.

2.  **Scrape Application Metrics:**
    *   Add a metrics library (like `prom-client` for Node.js) to your `pollr-api` service. Expose a `/metrics` endpoint that Prometheus can scrape.
    *   Create a `ServiceMonitor` Kubernetes object to tell Prometheus to automatically discover and scrape the `/metrics` endpoint of your `pollr-api` pods.

3.  **Build a Dashboard:**
    *   Log into the Grafana UI.
    *   Create a new dashboard to visualize the key metrics from your application:
        *   Request rate (`http_requests_total`).
        *   Error rate.
        *   Request latency.

**Outcome:** You have a live dashboard showing the health and performance of your application running in Kubernetes.

---

### Conclusion: You've Built the Platform

By completing this project, you have built a platform that mirrors the core principles of DevOps at a FAANG-level company. You have demonstrated mastery of:

*   **Containers and Orchestration:** Docker, Kubernetes.
*   **Infrastructure as Code:** Terraform.
*   **CI/CD:** GitHub Actions, security scanning.
*   **GitOps:** Argo CD.
*   **Monitoring:** Prometheus, Grafana.

You now have the practical experience to not only answer interview questions but to lead discussions on system design, architecture, and best practices. This project is your proof of competence. You can now confidently say, "I haven't just read about it; I've built it."
```