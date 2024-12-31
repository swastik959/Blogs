## Observing and Monitoring Large Language Model Workloads with Ray

### Introduction

The emergence of Large Language Models (LLMs) such as GPT-4, PHI2, BERT, and T5 revolutionized natural language processing, with these models empowering high-end applications, including chatbots, recommendation systems, and analytics. Yet the scale and complexity of workloads in LLMs make them a great challenge to guarantee performance and reliability. It is under such circumstances that monitoring and observability practices are more than essential while deploying workloads using frameworks such as Ray.&#x20;

Ray is a distributed computing framework that offers a powerful platform to scale LLM workloads efficiently across clusters. Therefore, it becomes an excellent choice for hosting, managing, and observing LLMs. The observability of critical metrics with Ray's built-in features in conjunction with Prometheus and Grafana will help users to monitor them efficiently, optimize the use of resources, and rapidly diagnose problems in production.

This article explores the importance of observability in Ray-hosted LLM workloads, key metrics to monitor, and a detailed guide to setting up observability using Prometheus and Grafana.

### Why Ray for LLM Workloads?

Ray is designed for distributed, scalable applications, making it ideal for hosting and managing LLM workloads. Key features that make Ray an excellent choice include:

- **Dynamic Task Scheduling:** Ray’s fine-grained task scheduling ensures efficient resource utilization, especially when processing LLM inference tasks that can vary significantly in size and complexity.
- **Ease of Integration:** Ray integrates seamlessly with frameworks like Hugging Face Transformers, enabling easy deployment of pre-trained LLMs.
- **Autoscaling:** Ray’s cluster autoscaler dynamically adjusts resources based on workload demands, ensuring cost-effectiveness and scalability.
- **Observability Support:** Ray provides metrics endpoints compatible with Prometheus, simplifying the monitoring setup for distributed systems.

These features make Ray not just a compute framework but a foundational tool for running, monitoring, and scaling LLMs in real-world applications.

### Key Metrics for Observing Ray-Hosted LLM Workloads

To ensure the smooth operation of Ray-hosted LLM workloads, it’s critical to track a range of performance, resource utilization, and operational metrics. Below are the key categories:

#### Performance Metrics

- **Task Latency:** Measures the time taken for individual Ray tasks to complete, essential for identifying bottlenecks in the inference pipeline.
- **Throughput:** Tracks the number of tasks completed per second, reflecting the system’s ability to handle high request volumes.
- **Token Processing Rate:** Measures the number of tokens processed per second, particularly relevant for transformer-based models like GPT-4.

#### Resource Utilization Metrics

- **CPU and GPU Utilization:** Monitors resource usage across the cluster to ensure efficient workload distribution.
- **Memory Usage:** Tracks memory consumption to prevent out-of-memory errors, especially critical for hosting large models.
- **Object Store Utilization:** Observes the usage of Ray’s in-memory object store for efficient data sharing across tasks.

#### Operational Metrics

- **Error Rates:** Monitors task failure rates to detect and resolve issues quickly.
- **Node Availability:** Tracks the health of nodes in the Ray cluster, ensuring reliability.
- **Queue Length:** Measures the number of pending tasks, signaling potential bottlenecks in processing.

### Setting Up Observability for Ray-Hosted Workloads

Observability in Ray involves using metrics to understand system performance and diagnose issues. By integrating Ray with Prometheus and Grafana, you can gain deep insights into workload behavior.

#### Step 1: Setting Up Prometheus Monitoring

Prometheus is an open-source monitoring system that collects metrics from Ray’s endpoints. Follow the guide below to set up Prometheus with Ray on Kubernetes.

**Install Prometheus with KubeRay:**

```bash
# Path: kuberay/
./install/prometheus/install.sh

# Check the installation
kubectl get all -n prometheus-system
```

#### Configure Pod and Service Monitors

Set up PodMonitor and ServiceMonitor resources to scrape metrics from Ray head and worker nodes:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: ray-workers-monitor
  namespace: prometheus-system
  labels:
    release: prometheus
    ray.io/cluster: rayservice-sample-raycluster-bpkgv
spec:
  jobLabel: ray-workers
  namespaceSelector:
    matchNames:
      - raysvc
  selector:
    matchLabels:
      ray.io/node-type: worker
  podMetricsEndpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: resume-analyzer-monitor
  namespace: prometheus-system
  labels:
    release: prometheus
spec:
  jobLabel: resume-analyzer
  namespaceSelector:
    matchNames:
      - raysvc
  selector:
    matchLabels:
      ray.io/node-type: head
    endpoints:
      - port: metrics
    targetLabels:
      - ray.io/cluster
```

#### Step 2: Configure Recording Rules

Recording rules allow you to precompute PromQL expressions for faster queries. For example, calculating the availability of the Ray Global Control Store (GCS):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ray-cluster-gcs-rules
  namespace: prometheus-system
  labels:
    release: prometheus
spec:
  groups:
  - name: ray-cluster-main-staging-gcs.rules
    interval: 30s
    rules:
    - record: ray_gcs_availability_30d
      expr: |
        (
          100 * (
            sum(rate(ray_gcs_update_resource_usage_time_bucket{container="ray-head", le="20.0"}[30d]))
            /
            sum(rate(ray_gcs_update_resource_usage_time_count{container="ray-head"}[30d]))
          )
        )
```

**Explanation of the Expression:**

- `ray_gcs_update_resource_usage_time_bucket`: Tracks the latency of resource usage updates.
- `ray_gcs_update_resource_usage_time_count`: Counts the total number of updates.
- The expression calculates the percentage of updates completed within a specific latency threshold over the last 30 days.

#### Step 3: Set Up Alerting Rules

Alert rules help identify issues proactively. For example, detecting missing GCS metrics:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ray-cluster-gcs-rules
  namespace: prometheus-system
  labels:
    release: prometheus
spec:
  groups:
  - name: ray-cluster-main-staging-gcs.rules
    interval: 30s
    rules:
    - alert: MissingMetricRayGlobalControlStore
      expr: |
        absent(ray_gcs_update_resource_usage_time_count)
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Missing Ray GCS metrics"
```

### Setting Up Grafana Dashboards

Grafana provides rich visualizations for metrics. Here’s how to set up dashboards for Ray:

#### Step 1: Capture Default Dashboards

Copy default dashboards from the Ray head pods:

```bash
kubectl cp <head-pod>:/tmp/ray/session_latest/metrics/grafana/dashboards/ ./dashboards
```

#### Step 2: Access the Grafana Dashboard

```bash
kubectl port-forward deployment/prometheus-grafana -n prometheus-system 3000:3000
```

Default login credentials:

- Username: `admin`
- Password: `prom-operator`

### Enable Profiling in Ray Serve Pods

Advanced Profiling in Ray Serve Pods The profiling of inference workloads relies on sophisticated techniques for monitoring, debugging, and optimizing performance. This section digs into specific tools, configurations, and scenarios to augment your profiling abilities.

#### Memory Profiling

Memory profiling is essential for memory leaks detection and usage optimization. For example, with Memray, trace memory allocations and understand the behavior of inference tasks. To enable memory profiling in Ray Serve pods, update the container's security context to allow tracing:

```yaml
securityContext:
  capabilities:
    add:
    - SYS_PTRACE
```

Once configured, Memray can be used to generate memory usage reports, which can help identify high-memory-consuming tasks or bottlenecks in the system.

**Example Use Case:**

- Profiling memory usage during a batch inference task with a large transformer model to optimize batch sizes and reduce memory overhead.

#### CPU Profiling

For CPU profiling, tools like `gdb`, `lldb`, or `py-spy` can be installed within the worker pods to collect detailed CPU usage data. These tools allow you to monitor which functions consume the most CPU time, enabling targeted optimizations.

To set up CPU profiling:

1. Install `gdb` or `lldb` in the ray worker pod.
2. Use profiling scripts or tools to capture CPU usage snapshots during inference tasks.

**Example Commands:**

```bash
apt-get update && apt-get install -y gdb
py-spy top --pid <worker-pid>
```

**Example Use Case:**

- Identifying CPU-bound operations in pre-processing pipelines to offload them to GPUs or optimize their implementation.

#### End-to-End Profiling Example

When you integrate memory and CPU profiling, it gives you an overarching view of system performance. To illustrate this better, consider an LLM inference task where you have latency spikes. If you correlate your memory and CPU profiles you will find:



The main culprit behind the memory usage is that huge batches of input data.

CPU bottlenecks are caused due to inefficiencies in tokenization functions.

If you optimize batch sizes and refactor bottleneck functions your performance might increase up to a considerable extent.

### Conclusion

Using Ray's distributed LLM workloads with the observability of robust tools will ensure that teams get performance, reliability, and scalability out of these systems. This is a guide to set up and monitor LLM workloads on Ray in a very practical way. Proper observability will help developers and operators find issues early, optimize the use of resources, and further improve the experience users get when using NLP applications.
