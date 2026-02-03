# Chapter 10: Monitoring, Logging, and Observability - Seeing Inside Your Systems

## Abstract

In distributed systems, failure is not an exception; it is a constant. The ability to detect, diagnose, and recover from these failures is what separates a reliable system from a fragile one. This is the domain of monitoring, logging, and the modern paradigm of observability. For a DevOps engineer, this is not about simply having a pretty dashboard; it's about deeply understanding the health and performance of your applications and infrastructure. It requires a first-principles understanding of different data types (metrics, logs, traces), the architectures of industry-standard tools like Prometheus and Grafana, and the ability to instrument code to ask arbitrary questions of your system in production. This chapter provides that definitive, book-level expertise, moving from legacy monitoring to the modern practice of observability.

---

### Part 1: The Evolution - From Monitoring to Observability

*   **Monitoring (The Old Way):** Monitoring is about collecting a predefined set of metrics and alerting when they cross a certain threshold. You decide in advance what you want to look for (e.g., "alert me if CPU usage is over 90%"). This is like asking a specific set of known questions.
    *   **Problem:** This works well for predictable failures but fails completely when faced with "unknown unknowns"—problems you didn't anticipate. A high CPU alert tells you *that* there is a problem, but it doesn't tell you *why*.

*   **Observability (The New Way):** Observability is a property of a system. An observable system is one that can be understood and debugged from the outside, without needing to ship new code to ask new questions. It's about having data rich enough to allow you to explore and understand any state the system might get itself into.
    *   Observability is built on **The Three Pillars**:
        1.  **Metrics:** Time-series data. A numerical representation of a value at a specific point in time (e.g., `cpu_usage{host="web-01"}` is `85.5` at `t1`). Metrics are great for aggregation and for understanding the overall health and trends of a system.
        2.  **Logs:** An immutable, timestamped record of a discrete event that occurred over time (e.g., an application error, a user login). Logs provide detailed, event-specific context.
        3.  **Traces:** A representation of a single request as it flows through all the different services in a distributed system. Traces are essential for debugging latency and understanding dependencies in a microservices architecture.

---

### Part 2: Metrics-Based Monitoring with Prometheus and Grafana

Prometheus has become the de facto standard for metrics-based monitoring in the cloud-native world.

#### The Prometheus Architecture: A Pull-Based Model

1.  **Prometheus Server:** The core of the system. It is responsible for:
    *   **Scraping Metrics:** Prometheus is **pull-based**. It periodically reaches out to configured "targets" over HTTP, ingests the metrics they expose, and stores them in its time-series database.
    *   **Time-Series Database (TSDB):** A highly optimized local database for storing metrics data.
    *   **Querying:** It provides a powerful query language called **PromQL** to select and aggregate the time-series data.

2.  **Targets / Exporters:**
    *   An application can expose metrics directly in the Prometheus format on a `/metrics` HTTP endpoint. This is called **instrumentation**.
    *   For systems that can't be instrumented directly (like a Linux kernel, a database, or a hardware load balancer), you use an **exporter**. An exporter is a small helper application that sits next to the target system, gathers metrics from it, and converts them into the Prometheus format. Common examples include the `node_exporter` (for host-level metrics) and the `mysqld_exporter` (for MySQL metrics).

3.  **Alertmanager:**
    *   Handles alerts sent by the Prometheus server. It is responsible for deduplicating, grouping, and routing alerts to the correct notification channel (e.g., PagerDuty, Slack, email).

4.  **Grafana:**
    *   The de facto visualization tool for Prometheus. Grafana is a separate application that uses Prometheus as a data source. It allows you to build rich, interactive dashboards by querying Prometheus with PromQL.

#### The Prometheus Data Model

*   All data is stored as a **time series**. A time series is identified by its **metric name** and a set of key-value pairs called **labels**.
    *   `http_requests_total{method="POST", handler="/api/users"}`
*   This dimensional model is incredibly powerful. You can slice and dice your data on any label. For example, you can get the total number of requests, or just the POST requests, or just the POST requests to the `/api/users` handler.

#### PromQL: The Power to Query

PromQL is what makes Prometheus so powerful. It's a functional query language that lets you select and aggregate time-series data in real time.

*   **Selecting Data:**
    *   `node_cpu_seconds_total{mode="idle"}`: Selects the time series for idle CPU time.
    *   `http_requests_total{job="my-app", status_code=~"5.."`: Selects all requests for the "my-app" job where the status code was in the 500s (using a regex match).

*   **Rates of Change:**
    *   `rate(http_requests_total[5m])`: Calculates the per-second average rate of increase of HTTP requests over the last 5 minutes. This is one of the most common and important functions in PromQL. It turns a constantly increasing counter into a useful rate.

*   **Aggregation:**
    *   `sum(rate(http_requests_total[5m])) by (handler)`: Calculates the total request rate, summed up and grouped by the `handler` label. This would show you the request rate for each of your API endpoints.

*   **Alerting Rules:**
    *   Alerts in Prometheus are just PromQL expressions that are evaluated at a regular interval.
    *   ```yaml
        - alert: HighErrorRate
          expr: sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
          for: 10m
          labels:
            severity: page
          annotations:
            summary: "High HTTP error rate detected for job {{ $labels.job }}"
        ```
        This rule fires if the ratio of 5xx errors to total requests is greater than 5% for a continuous period of 10 minutes.

---

### Part 3: Centralized Logging with the EFK Stack

While metrics tell you *that* something is wrong, logs tell you *why*. In a distributed system, logs are scattered across hundreds of containers. You need a centralized logging solution. The EFK stack is a popular open-source choice.

1.  **Elasticsearch:**
    *   **What it is:** A powerful, distributed search and analytics engine built on Apache Lucene.
    *   **Function:** It is the central store for all your log data. It indexes the logs, making them searchable in near real-time.

2.  **Fluentd (or Fluent Bit):**
    *   **What it is:** A data collector. It is deployed as a **DaemonSet** in Kubernetes, meaning an instance runs on every single worker node.
    *   **Function:** It tails the log files from all the containers running on its node, parses them, adds metadata (like the pod name and namespace), and forwards them to a central location, in this case, Elasticsearch. Fluent Bit is a more lightweight version often preferred for this task.

3.  **Kibana:**
    *   **What it is:** A web UI for Elasticsearch.
    *   **Function:** It allows you to search, visualize, and explore the log data stored in Elasticsearch. You can create dashboards, filter by fields (like `pod_name` or `status_code`), and perform full-text searches to quickly find the error message you are looking for.

---

### Part 4: Distributed Tracing - Understanding the Request Lifecycle

In a microservices architecture, a single user request might travel through dozens of different services before a response is returned. If that request is slow, how do you know which service is the bottleneck? This is the problem that distributed tracing solves.

*   **How it works:**
    1.  **Instrumentation:** The application code in each service must be instrumented with a tracing library.
    2.  **Trace Context Propagation:** When the first service (e.g., the public API gateway) receives a request, it generates a unique **Trace ID**. It then injects this Trace ID into the HTTP headers of any downstream requests it makes to other services.
    3.  **Span Creation:** Each service that receives a request with a Trace ID creates a **Span**. A span represents a single unit of work within that service (e.g., a database query or a call to another service). The span records its start time, end time, and other metadata.
    4.  **Trace Assembly:** All these spans, from all the different services but sharing the same Trace ID, are sent to a central tracing backend (like **Jaeger** or **Zipkin**). The backend assembles them into a single, hierarchical trace, which can be visualized as a waterfall graph.

*   **What it gives you:** A visual representation of the entire request lifecycle. You can see exactly how long the request spent in each service and which calls were made in series vs. in parallel. This makes it trivial to pinpoint the source of latency in a complex system.

---

### Part 5: A Deep Dive into Prometheus and Grafana

While Part 2 provided a high-level overview, a FAANG-level understanding requires a deeper dive into the architectural nuances and advanced capabilities of this powerful duo.

#### 5.1 Prometheus Architecture Unveiled

Let's dissect the components of the Prometheus server itself.

*   **The Scrape Engine:** This is the heart of Prometheus's pull model.
    *   **Service Discovery:** The scrape engine needs to know *what* to scrape. It integrates with various service discovery mechanisms (e.g., Kubernetes, Consul, or static files) to get a list of targets. For Kubernetes, it will query the K8s API to find pods and services that have specific annotations (`prometheus.io/scrape: 'true'`).
    *   **Scrape Loop:** At a configured interval (`scrape_interval`), the engine iterates through the list of targets and makes an HTTP GET request to their `/metrics` endpoint.
    *   **Target Relabeling:** Before scraping, Prometheus can apply "relabeling" rules to modify the labels of a target. This is incredibly powerful for standardizing labels, filtering targets, or adding metadata.

*   **The Time-Series Database (TSDB):**
    *   **On-Disk Format:** The TSDB stores data on disk in "blocks." Each block contains all the time-series data for a specific time range (by default, 2 hours). A block is made up of a chunk directory (containing the actual compressed data points), an index file (for quickly looking up metrics by labels), and a meta.json file.
    *   **Write-Ahead Log (WAL):** For the most recent 2-hour block, new data is first written to a Write-Ahead Log (WAL) in memory and on disk. This prevents data loss if the server crashes. The in-memory data is then periodically "compacted" into the on-disk block format.
    *   **Compaction:** Over time, smaller blocks are compacted into larger ones in the background to improve query performance and reduce storage space.

*   **The PromQL Engine:** This component processes PromQL queries. It uses the TSDB's index to quickly find the relevant time series and then performs calculations (like `rate()`, `sum()`, `histogram_quantile()`) on the retrieved data points.

*   **The Alerting Engine:** This engine runs a separate loop that evaluates the PromQL expressions defined in your alerting rules at a regular interval (`evaluation_interval`). If an expression is true for its configured `for` duration, it fires an alert and sends it to the Alertmanager.

#### 5.2 The Pull Model vs. The Push Model

Prometheus's pull-based architecture is a deliberate design choice with significant trade-offs.

*   **Pull Model (Prometheus):**
    *   **Pros:**
        *   **Centralized Control:** You configure scraping from a central location. You can easily see what is being monitored and what isn't.
        *   **Simplified Clients:** The application/exporter just needs to expose an HTTP endpoint. It doesn't need to know where the Prometheus server is.
        *   **Health Monitoring:** By scraping a target, you are inherently testing its health. If a target is down, the scrape will fail, and you can immediately alert on the `up` metric.
    *   **Cons:**
        *   **Firewall/NAT Traversal:** It can be difficult to scrape targets that are behind a firewall or in a different network.
        *   **Short-Lived Jobs:** It's not good for monitoring very short-lived jobs (like a serverless function) that may start and stop between scrape intervals.

*   **Push Model (e.g., Graphite, InfluxDB):**
    *   **Pros:**
        *   Works well for short-lived jobs.
        *   Simpler in complex network environments.
    *   **Cons:**
        *   **Configuration Sprawl:** Every client needs to be configured with the address of the monitoring server.
        *   **"Dumb" Server:** The server doesn't know what it's supposed to be monitoring. It just passively receives data. It's harder to tell if a service is down or if it just stopped sending metrics.

*   **The Pushgateway:** Prometheus has a component called the `Pushgateway` to accommodate use cases that require a push model. A short-lived batch job can push its metrics to the Pushgateway, which then holds onto them to be scraped by Prometheus. However, its use is explicitly discouraged for most cases, as it can easily become a single point of failure and a metrics bottleneck.

#### 5.3 Advanced PromQL

*   **Histograms and `histogram_quantile()`:** Histograms are a powerful way to measure the distribution of values, like request latencies. A histogram metric exposes multiple time series with an `le` (less than or equal to) label, representing buckets.
    *   `http_request_duration_seconds_bucket{le="0.1"}`: Number of requests that took less than or equal to 0.1 seconds.
    *   The `histogram_quantile()` function is used to calculate percentiles from these buckets.
    *   `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, handler))`: This calculates the 95th percentile latency for each API handler over the last 5 minutes. This is a critical SLO metric.

*   **Vector Matching (`on`, `ignoring`, `group_left`, `group_right`):** When performing binary operations (e.g., `metric1 / metric2`), you need to control how labels are matched between the two sides.
    *   `instance_memory_usage_bytes / on(instance) instance_memory_limit_bytes`: Calculates memory usage as a percentage, matching only on the `instance` label.
    *   `group_left`: Includes all the labels from the "left" side of the expression in the result. This is essential for preserving the dimensionality of your data.

#### 5.4 Grafana Deep Dive

*   **Data Sources:** While Grafana is famous for its Prometheus integration, it's a polyglot tool. It can connect to dozens of data sources, including Elasticsearch, Loki, InfluxDB, and even SQL databases. This allows you to create a single dashboard that combines metrics, logs, and traces from different systems.

*   **Variables:** This is what makes Grafana dashboards interactive and reusable. A variable is a dropdown at the top of the dashboard.
    *   *Example:* You can create a variable named `job` that is populated by a PromQL query like `label_values(up, job)`. Then, in your panel queries, you can use `rate(http_requests_total{job="$job"}[5m])`. Now, when you select a job from the dropdown, all the panels on the dashboard will update to show data for only that job.

*   **Panel Types:** Grafana offers a huge variety of panel types beyond simple graphs, such as:
    *   **Stat:** For showing a single, important number (e.g., current error rate).
    *   **Gauge:** For showing a value relative to a min/max (e.g., CPU usage).
    *   **Table:** For displaying multi-dimensional data.
    *   **Heatmap:** For visualizing distributions over time.
    *   **Node Graph:** For visualizing dependencies.

*   **Provisioning:** Manually creating dashboards via the UI is fine for exploration, but for a production system, you should provision them as code. You can define your dashboards as JSON files and have Grafana automatically load them on startup. This allows you to version control your dashboards in Git.

---
### ★ FAANG-Level Interview Scenarios (Prometheus & Grafana) ★

*   **Scenario 4: The Missing Metric**
    *   **Question:** "You've deployed a new service, and you've confirmed it's exposing a `/metrics` endpoint. However, when you go to Grafana and query for your new metric, nothing shows up. What are the steps you would take to debug this?"
    *   **Answer:** "This is a common problem that involves tracing the path of the metric from the source to the dashboard.
        1.  **Check the Source:** First, I'd `curl` the `/metrics` endpoint of the pod directly to ensure the metric is actually being exposed with the correct name and labels.
        2.  **Check Prometheus Service Discovery:** My next step is to see if Prometheus is even aware of my new service. I'd go to the Prometheus UI, to the 'Targets' page. I would check if my service's pods are listed as targets. If not, the problem is with service discovery. I'd check if the service has the correct `prometheus.io/scrape: 'true'` annotation that our `ServiceMonitor` is configured to look for.
        3.  **Check Target Health:** If the target is listed, what is its status? Is it 'UP' or 'DOWN'? If it's 'DOWN', Prometheus is failing to scrape it. This could be a network policy issue, an mTLS problem, or the pod could be crash-looping. I'd check the error message on the Targets page.
        4.  **Check Relabeling Rules:** If the target is 'UP', it's being scraped successfully. The problem might be that a relabeling rule is accidentally dropping the metric or changing its name. I'd check the Prometheus configuration for any `metric_relabel_configs` that might be affecting it.
        5.  **Query Prometheus Directly:** Before blaming Grafana, I would use the Prometheus UI's query bar to search for the metric. If it appears here, then Prometheus has the data, and the problem is likely in Grafana (e.g., a typo in the panel's query, an incorrect data source selected, or a time range issue)."

*   **Scenario 5: Choosing the Right Aggregation**
    *   **Question:** "You have a metric `request_latency_seconds` which is a Gauge that records the latency of each individual request. You want to calculate the average latency over the last 5 minutes. Why is `avg(request_latency_seconds[5m])` the wrong way to do this, and what is the right way?"
    *   **Answer:** "That's a great question that highlights a common misunderstanding of how Prometheus processes data.
        *   **Why `avg(request_latency_seconds[5m])` is Wrong:** The `[5m]` range selector can only be used on a *range vector*, which is the output of a query that returns multiple data points over time for each series (like a raw counter). A Gauge like `request_latency_seconds` is a *instant vector*—it only returns the single most recent value for each time series. Applying `[5m]` to it is syntactically invalid.
        *   **The Deeper Problem:** Even if it were valid, you are trying to average the raw latencies. But Prometheus scrapes at an interval. It doesn't see *every* request. It just sees a *sample* of the latency at a specific point in time. Averaging these samples is not the true average latency.
        *   **The Right Way (The Rate of Sums):** The correct way to calculate average latency is to use two Counters:
            1.  `http_requests_total`: A counter for the total number of requests.
            2.  `http_request_duration_seconds_total`: A counter that sums the latency of all requests.
        *   With these two metrics, you can calculate the true average latency over the last 5 minutes with this PromQL query:
            `rate(http_request_duration_seconds_total[5m]) / rate(http_requests_total[5m])`
        *   This works because `rate()` calculates the per-second increase of each counter. Dividing the per-second increase in total latency by the per-second increase in total requests gives you the average latency per request. This is a fundamental pattern in Prometheus monitoring."