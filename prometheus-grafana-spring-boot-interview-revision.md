# Prometheus and Grafana with Spring Boot: Interview Revision

## 1. What are Prometheus and Grafana, and how do they work together?

Spring Boot exposes application metrics through Micrometer and Spring Boot Actuator. Prometheus periodically **scrapes** the metrics endpoint and stores the time series. Grafana queries Prometheus using PromQL and displays the results in dashboards or alerts.

```text
Spring Boot + Actuator/Micrometer
              ↑ scrape
         Prometheus
              ↑ query (PromQL)
           Grafana
```

Grafana usually does not collect or store metrics itself. It queries Prometheus as a data source.

## 2. How do you expose Prometheus metrics from Spring Boot?

Add Spring Boot Actuator and the Micrometer Prometheus registry dependency. Expose the Prometheus endpoint so Prometheus can scrape `/actuator/prometheus`. The endpoint requires the registry dependency and is not exposed by default.

Maven dependencies:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Application configuration:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

Protect management endpoints with appropriate network controls and application security. Do not expose sensitive management endpoints publicly.

## 3. What does Prometheus scrape configuration look like?

A scrape job defines the target address and metrics path. A static target is suitable for a local example or stable hosts:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: spring-app
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["spring-app:8080"]
```

Prometheus supports static targets and dynamic service-discovery mechanisms.

## 4. What are the common deployment patterns?

| Pattern | How it works | Typical fit |
|---|---|---|
| Direct scrape | Prometheus scrapes each app's Actuator endpoint | Local development or a few stable hosts |
| Service discovery | Prometheus discovers app instances through Kubernetes or cloud discovery | Dynamic workloads |
| Exporter sidecar | A helper container translates or exposes metrics for an app/process that cannot expose Prometheus metrics directly | Legacy apps, JMX, custom protocols |
| Pushgateway | Short-lived jobs push metrics to an intermediary that Prometheus scrapes | Batch jobs that may finish before a scrape |
| Managed Prometheus | A cloud service operates some or all of the Prometheus-compatible metrics backend | Teams reducing Prometheus server operations |

For a normal Spring Boot app, the app can expose `/actuator/prometheus` directly, so an exporter sidecar is usually unnecessary. Clarify what is meant by “sidecar” in an interview: it could mean an exporter, proxy, or another helper container running alongside the application.

## 5. How does the sidecar pattern work?

A sidecar runs next to the application in the same pod or deployment unit and provides a supporting function, such as translating JMX metrics into Prometheus format. Prometheus scrapes the sidecar's metrics endpoint. The app and sidecar share lifecycle and may share networking or volumes.

```text
Kubernetes Pod
 ├── legacy application (JMX on 127.0.0.1:9010)
 └── JMX exporter sidecar (metrics on :9404)
                    ↑
              Prometheus scrapes :9404
```

Illustrative Kubernetes fragment:

```yaml
containers:
  - name: legacy-app
    image: example/legacy-app
    ports:
      - containerPort: 9010

  - name: jmx-exporter
    image: prom/jmx-exporter:latest
    args: ["9404", "/etc/jmx/config.yaml"]
    ports:
      - name: metrics
        containerPort: 9404
```

The exact exporter image and arguments depend on the exporter version and deployment. A Spring Boot application that already exposes Micrometer metrics can normally be scraped directly.

## 6. How does Kubernetes service discovery fit in?

Pod IPs change, so maintaining a static target list is brittle. Prometheus can discover pods or services through the Kubernetes API and select targets using labels and relabeling rules. Another common approach is the Prometheus Operator, where `ServiceMonitor` or `PodMonitor` resources describe targets.

Conceptual example:

```yaml
scrape_configs:
  - job_name: spring-services
    kubernetes_sd_configs:
      - role: pod
```

Production configurations need the relevant Kubernetes permissions, relabeling, and target-selection rules. With the Operator, scrape configuration is often expressed through custom resources rather than manually maintained `prometheus.yml` entries.

## 7. How do you connect Grafana to Prometheus?

Configure a Prometheus data source in Grafana with the Prometheus server URL. Then create or import dashboards that query the data source. The URL must be reachable from Grafana's runtime environment. If they run in separate containers, `localhost` means the Grafana container itself; use the Prometheus service/container hostname instead.

Example provisioning file:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

## 8. What does a local end-to-end setup look like?

Run the Spring Boot app, Prometheus, and Grafana on a shared Docker network. Prometheus scrapes the app by its Compose service name; Grafana queries Prometheus by its service name.

```yaml
services:
  spring-app:
    image: example/spring-app
    ports:
      - "8080:8080"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

In `prometheus.yml`, use `spring-app:8080` as the target. In Grafana, use `http://prometheus:9090` as the data source URL.

## 9. What metrics should you monitor for a Spring Boot service?

Start with metrics that help diagnose availability, traffic, latency, and errors:

- Request rate, error rate, and latency distributions
- JVM heap and non-heap memory, garbage collection, and thread count
- CPU and process metrics
- Connection-pool usage and database operation latency
- Queue depth or Kafka consumer lag where applicable
- Business metrics that answer operational questions, such as orders processed

Micrometer supports custom counters, timers, and gauges:

```java
Counter counter = meterRegistry.counter("orders.created");
counter.increment();

Timer timer = meterRegistry.timer("inventory.lookup");
timer.record(() -> inventoryClient.fetch(sku));
```

Use bounded labels such as HTTP method or status. Avoid unbounded values such as user IDs, order IDs, and raw URLs as metric labels; high cardinality can overwhelm the metrics system.

## 10. What is the difference between a counter, gauge, and timer?

- **Counter:** A value that increases over time, such as total requests. Use a rate query to see its rate of increase.
- **Gauge:** A value that can rise or fall, such as active connections.
- **Timer:** Measures the duration and count of events, such as request latency. Histogram buckets and percentile behavior depend on configuration.

Example rate query for a cumulative request counter:

```promql
rate(http_server_requests_seconds_count[5m])
```

## 11. Does Prometheus scrape or receive metrics from Spring Boot?

In the standard setup, Prometheus scrapes (pulls) from Spring Boot's metrics endpoint. Prometheus controls the scrape frequency and can observe when a target is unavailable. Pushgateway is an exception for certain short-lived jobs, not the default approach for long-running services.

## 12. What is Pushgateway, and when would you use it?

A short-lived batch job may finish before Prometheus's next scrape. It can push its final metrics to Pushgateway, which remains available for Prometheus to scrape. Use this selectively: it changes the normal pull model and requires attention to metric lifecycle and stale values. A long-running Spring Boot web service should generally expose a scrape endpoint instead.

## 13. How would you add alerts?

Define alert rules for meaningful symptoms, such as a sustained high error rate, elevated latency, or an unavailable target. Prometheus evaluates alert rules; an alerting component routes notifications to the appropriate team or channel. Grafana can also manage alert rules, depending on the setup.

Illustrative Prometheus rule:

```yaml
groups:
  - name: spring-app
    rules:
      - alert: SpringAppTargetDown
        expr: up{job="spring-app"} == 0
        for: 2m
        labels:
          severity: page
```

A useful alert has a clear condition, a duration that avoids noise, and enough context to guide action.

## 14. What is PromQL? Give examples.

PromQL is Prometheus's query language for selecting and aggregating time-series data.

```promql
# Request rate over the last 5 minutes
rate(http_server_requests_seconds_count[5m])

# 95th percentile latency, when histogram buckets are enabled
histogram_quantile(
  0.95,
  sum by (le) (rate(http_server_requests_seconds_bucket[5m]))
)

# Target health
up{job="spring-app"}
```

Metric names depend on the metrics exported and their configuration. For latency queries, state whether histogram buckets are enabled and how labels are grouped.

## 15. How do you secure and operate the metrics endpoint?

Keep the endpoint reachable by Prometheus but restrict access from untrusted networks. Depending on the deployment, use a dedicated management port, network policies, authentication, and TLS. Monitor Prometheus storage and retention, scrape failures, target health, and Grafana-to-Prometheus connectivity. Spring Boot endpoint exposure and endpoint access are distinct concerns, so configure both appropriately.

## Quick interview answer

> “I add Spring Boot Actuator and Micrometer's Prometheus registry, expose `/actuator/prometheus`, and configure Prometheus to scrape the service. For static hosts I can use static targets; in Kubernetes I prefer service discovery or an Operator-based monitor. Grafana connects to Prometheus as a data source and uses PromQL for dashboards and alerts. A sidecar exporter is useful when an app cannot expose Prometheus metrics itself, such as a legacy JMX-only service; a typical Spring Boot service usually exposes metrics directly.”

## Remember the responsibilities

- **Spring Boot, Actuator, and Micrometer:** Instrumentation and metric exposure
- **Prometheus:** Scraping, time-series storage, querying, and rule evaluation
- **Grafana:** Data-source queries, dashboards, and visualization/alerting features

## Official references

- [Spring Boot metrics and Prometheus](https://docs.spring.io/spring-boot/reference/actuator/metrics.html)
- [Spring Boot Actuator endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html)
- [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Grafana Prometheus data source](https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/)
