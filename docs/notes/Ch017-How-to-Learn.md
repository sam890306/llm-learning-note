# 这些知识从哪获得
---

## 🧭 一、这类知识属于什么范畴？

一句话概括：

> **大模型系统工程（LLM Systems Engineering） = 分布式计算 + 深度学习 + 编译器 + 云基础设施的交叉领域。**

它不是纯算法、也不是纯后端，而是 **跨层系统协同设计**（Co-Design）。
主要归属以下方向：

| 学术领域                           | 对应课程/关键词                                     |
| ------------------------------ | -------------------------------------------- |
| 分布式系统 (Distributed Systems)    | RPC、容错、调度、微服务、K8s                            |
| 高性能计算 (HPC)                    | MPI、AllReduce、InfiniBand、并行算法                |
| 深度学习系统 (Deep Learning Systems) | PyTorch internals、XLA、TensorRT、FSDP          |
| 编译器与图优化 (Compiler for ML)      | IR、Graph Optimization、Triton、TVM             |
| 云基础设施 (Cloud Infra)            | K8s、Service Mesh、Observability、Infra-as-Code |

OpenAI、Google、Anthropic 的工程师几乎都是从这几个方向拼起来的。

---

## 🧠 二、国外怎么学：从开源系统入手

下面这几条路径是最接近工业界真相的学习路线：

### 1️⃣ **PyTorch 内核机制**

* 📘 《Deep Learning Systems: Algorithms, Compilers, and Processors》
  → Berkeley CS267 / CMU 10-414 课程（可在 YouTube 找）
* 🧩 学 PyTorch Autograd、Graph Tracing、TorchScript、FSDP 源码
  → 能理解“计算图”与“分布式并行”的底层实现。

### 2️⃣ **HPC + 分布式通信**

* 学 MPI、NCCL、AllReduce、Ring Reduce、Pipeline Parallelism
  推荐：Stanford CS149、Berkeley CS267
* 熟悉 NVIDIA 官方文档《Scaling Deep Learning with NCCL》

### 3️⃣ **编译器优化**

* 研究 XLA、TVM、Triton、AITemplate
  推荐：MIT 6.S897《Compiler for Machine Learning》公开课
* TVM 官方教程 + PyTorch 2.0 Dynamo / Inductor 源码。

### 4️⃣ **推理优化与 Serving**

* 实践 TensorRT-LLM、vLLM、DeepSpeed-Inference
* 阅读 HuggingFace Text Generation Inference (TGI) 源码。
* 了解 speculative decoding、KV-Cache 管理、batch scheduler。

### 5️⃣ **系统编排与监控**

* Kubernetes、Istio、Prometheus、Grafana
* 了解云原生的 scaling / fault-tolerant / autoscaler 概念。
* Azure / AWS AI Supercomputer 架构文档。

---

## 🔬 三、如果你要“学会自己搭一套小规模版”

### Step 1：分布式训练

* 用 **ColossalAI** 或 **DeepSpeed** 跑一个 1B 参数模型。
* 学会配置 ZeRO、Tensor Parallel、Pipeline Parallel。

### Step 2：计算图与编译器

* 导出 PyTorch 模型成 ONNX，观察图结构。
* 用 **TensorRT** 或 **AITemplate** 优化图。
* 分析 kernel 融合、显存利用。

### Step 3：推理服务化

* 用 **vLLM** 搭分布式推理服务。
* 加上 **FastAPI / Ray Serve** 作为调度层。

### Step 4：部署与监控

* 部署到 GKE 或阿里 ACK，使用 Prometheus + Grafana 监控 GPU 负载。
* 体验自动扩缩容与容错迁移。

这四步跑通后，你就具备国内大厂“模型系统工程师”的核心能力。

---

## 🧭 四、推荐资料路线图（精华级）

| 阶段                 | 学习内容                                | 推荐资料                                                   |
| ------------------ | ----------------------------------- | ------------------------------------------------------ |
| **阶段 1：理解分布式训练原理** | Data / Model / Pipeline Parallelism | DeepSpeed 官方文档 + Megatron-LM 论文                        |
| **阶段 2：理解计算图与编译器** | 图优化、IR、算子融合                         | TVM 教程 + PyTorch 2.0 Dynamo 设计文档                       |
| **阶段 3：推理系统与性能优化** | vLLM、TensorRT、KV Cache              | vLLM Paper + NVIDIA TensorRT-LLM Blog                  |
| **阶段 4：系统部署与调度**   | K8s, Ray, Observability             | 《Designing Data-Intensive Applications》 + Ray Serve 教程 |
| **阶段 5：综合项目实践**    | 自建微型 LLM Cloud                      | FlowMind / ColossalAI / DeepSpeed 实战                   |

---

## 🧱 五、总结路径

| 阶段        | 国外代表                 | 国内代表                  | 核心思想       |
| --------- | -------------------- | --------------------- | ---------- |
| 1️⃣ 分布式训练 | DeepSpeed / Megatron | ColossalAI / FleetX   | 算力分片与容错    |
| 2️⃣ 图编译优化 | XLA / TensorRT       | MindIR / AITemplate   | 计算图优化与混合精度 |
| 3️⃣ 推理服务化 | vLLM / TGI           | ByteInfer / DashInfer | 高吞吐推理与调度   |
| 4️⃣ 云编排   | Azure AI / Pathways  | Volcano / Lingjun     | 资源调度与可观测性  |

---
