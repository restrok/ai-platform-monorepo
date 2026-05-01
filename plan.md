Technical Study Plan: Low-Cost End-to-End LLMOps Lab on GKE

This study plan is designed to turn theoretical knowledge into real architectural capability. Unlike packaged codelabs that hide operational complexity, this lab requires building the stack from scratch using Infrastructure as Code (IaC). As architects, our goal is not only to "make it work," but to optimize the cost-performance ratio using accessible hardware such as NVIDIA L4 GPUs, avoiding the bureaucracy and prohibitive costs of TPUs.

1. Fundamentals of Disaggregated Inference Architecture

Transformer inference is inherently inefficient when run monolithically. We must understand the physical separation of phases to remove bottlenecks:

* Prefill (Prefix Phase): Processes the input prompt to compute initial attention states. It is a compute-bound operation due to intensive matrix multiplications. In shared architectures, a heavy prefill blocks other users' decoding steps.
* Decode (Decoding Phase): Generates tokens autoregressively. It is a memory-bound process limited by how fast weights can be read from HBM (High Bandwidth Memory). Here, compute cycles are often underutilized.

Prefill-Decode (PD) Disaggregation breaks this conflict by assigning each phase to specialized nodes, allowing prefill to stop degrading decode interactivity.

Critical Performance Metrics (North Star Metrics)

Metric	Critical Phase	Architectural Impact
TTFT (Time to First Token)	Prefill	System responsiveness. Reduce through massive compute or KV cache hits.
TPOT (Time Per Output Token)	Decode	Perceived reading speed. Depends on memory bandwidth.
Throughput (Tokens/Sec)	Both	Total system capacity. Optimized through batching and decoupling.
Queue Latency	Global	Waiting time in queue before processing.

2. Physical Layer Setup (Module 1: Infrastructure)

For this lab, we will use Terraform in the `1-infra/` folder. The deployment is based on a simplified AI Hypercomputer design focused on low cost.

Networking and VPC: The MTU Mandate

Network configuration is non-negotiable. A custom VPC must be implemented with an MTU of 8896 (Jumbo Frames).

* Justification: Tensor traffic between prefill and decode nodes via NCCL (NVIDIA Collective Communications Library) is massive. A standard MTU of 1500 would cause packet fragmentation, degrading distributed inference stability and performance.

GKE Standard: Node Pool Management

We will avoid GKE Autopilot to keep full control over hardware scheduling. We will configure three specific pools:

1. System Pool: Standard CPU nodes for management services.
2. GPU Pools (L4): Two specialized pools using `g2-standard-12` (1x L4) or `g2-standard-24` (2x L4) instances.
  * Taints: Applying strict taints (`nvidia.com/gpu:NoSchedule`) is mandatory to ensure VRAM is exclusive model territory and avoid noisy neighbors.
  * Spot Instances: For budget-conscious professionals, Spot nodes are the de facto standard, enabling operation below $5 USD/hour.

3. Middleware and Orchestration (Module 2: Platform)

The platform layer (`2-platform/`) acts as the operating system of our AI infrastructure.

* KubeRay Operator: Fundamental for coordination. Unlike a static pod, vLLM requires a "Head" node that manages global memory state and "Worker" nodes that execute compute.
* Inference Extension API & kgateway: We will implement routing through kgateway (Envoy-based). This layer is not a simple HTTP load balancer; it is an intelligent routing engine implementing the Endpoint Picker Protocol (EPP), enabling backend selection based on KV cache utilization and queue depth.

4. LLMOps Observability and Monitoring (Module 3: LLMOps)

Without granular metrics, the architect is "flying blind." The dcgm-exporter setup must be the core pillar of telemetry.

Collection Specification (PromQL Targets)

We must capture metrics that reveal hardware saturation:

* `dcgm_gpu_temp`: To monitor thermal throttling.
* `vllm:kv_cache_usage_ratio`: The key health indicator. A 100% ratio means the system will begin evicting context or recomputing.
* `dcgm_tensor_copy_util`: To identify whether we are compute-bound.

Grafana Dashboard

The design must include panels that correlate KV cache usage with queue latency. If latency rises while VRAM is full, we have identified a memory bottleneck requiring horizontal scaling or tiered storage policies.

5. Workload Deployment and Distributed Inference (Module 4: Workloads)

We will use llm-d, the framework from Red Hat, Google, and IBM, to manage distributed inference.

* Model Setup: For this lab, the workhorse model will be `Llama-3-8B-Instruct`. Although the infrastructure can serve Qwen3-32B or Llama-3.1-405B, the 8B model ensures load tests do not exceed budget.
* Workload Identity: Static keys are not allowed. We must configure Workload Identity so vLLM pods securely access GCS buckets for weights and Hugging Face secrets.
* Inference Routing: In `inference-routing.yaml`, we will define `InferencePool` (resource set) and `InferenceModel` (service abstraction), allowing the Gateway to route based on request criticality (`InferenceObjective`).

6. Advanced KV Cache Management and GKE Inference Gateway

KV cache is the most expensive and critical resource in inference. We will implement a Tiered KV Cache (LMCache) strategy to expand capacity beyond GPU HBM.

Storage and Performance Hierarchy

1. Tier 1: GPU HBM: Maximum speed, minimum capacity.
2. Tier 2: CPU RAM: Medium capacity, moderate latency.
3. Tier 3: Local SSD (via `emptyDir`): Massive capacity.

Impact Evidence: According to GKE benchmarks, implementing Tier 3 (Local SSD) for long contexts of 100k tokens enables:

* A 79% reduction in TTFT, by avoiding prefix recomputation.
* A 264% increase in input throughput.

Prefix-Aware Routing

The GKE Inference Gateway uses prefix-aware routing to direct requests with similar prompts to the same nodes. This maximizes cache hit ratio and is the difference between a system that scales and one that collapses under load.

7. Execution Protocol and Budget Optimization

As senior architects, budget handling is a success metric. This lab is designed to cost between **$2 and $4 USD per session**, compared to the >$60 USD that a TPU v6e-based implementation would cost.

Lifecycle Checklist

1. Provisioning (15 min): `terraform apply`. Bring up the VPC with MTU 8896 and L4 node pools.
2. Orchestration: `kubectl apply` for platform manifests (KubeRay, kgateway).
3. Deploy & Test (30 min): Deploy Llama-3-8B and run load tests. Mandatory monitoring of VRAM saturation in Grafana.
4. Immediate Teardown: `terraform destroy`.

Final mandate: Do not persist resources. Infrastructure must be treated as ephemeral. Technical mastery comes from the ability to bring up and tear down this full stack in minutes, not from keeping it running.
