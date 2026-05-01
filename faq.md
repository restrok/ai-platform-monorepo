# LLMOps Lab FAQ

## 1) What is this lab for?

This lab is for building a real end-to-end AI platform, instead of only running demos.

In simple terms:

- It defines a complete architecture to run an LLM on Kubernetes.
- It lets you measure real performance (latency, throughput, GPU/KV cache usage).
- It helps you operate with a cost-performance focus.
- It trains production-grade practices: IaC, observability, routing, and ephemeral operations.

## 2) What are we actually making available?

We are making available an **LLM inference service** (for example, `Llama-3-8B-Instruct`) that can be consumed through an API.

That means:

- The model is deployed as a backend service on Kubernetes.
- A chatbot (or any other application) calls that endpoint to generate answers.
- This repo focuses on the backend engine and its operations, not on chatbot UI.

## 3) How is this different from opening LM Studio and loading a model?

LM Studio and this lab solve different needs:

- **LM Studio:** ideal for quick, local, personal testing.
- **This lab:** designed to expose a multi-user service with monitoring, routing, security, and cost control.

Summary:

- LM Studio = local experimentation.
- GKE + vLLM + Kubernetes = operating an LLM backend closer to production.

## 4) What changes between doing this on GKE versus an on-prem server?

The core technical concepts are similar, but operations are very different:

- **GKE:** lower operational friction, faster provisioning, cloud integrations, pay-as-you-go.
- **On-prem:** more control and possible long-term savings, but much higher operational complexity (hardware, networking, failures, capacity planning, physical security).

Practical rule:

- For learning, fast iteration, and validation: GKE is usually better.
- For steady workloads with a mature platform team: on-prem can make sense.

## 5) What is the CV/career impact of learning this lab?

It has high impact because it demonstrates that you can operate AI workloads on real infrastructure.

It positions you better for roles such as:

- ML Platform Engineer
- MLOps / LLMOps Engineer
- AI Infrastructure Engineer
- AI-focused SRE

It also gives concrete interview evidence in:

- Architecture design
- Cost vs. performance trade-offs
- Metrics-driven and reliable operations
- Practical use of Kubernetes, GKE, Terraform, and GPU observability
