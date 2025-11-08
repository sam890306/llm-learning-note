# Week 1–2（第一阶段：能跑起来）
目标非常明确：

> ✅ 在一台 GPU 云主机上，启动一个可交互的开源大语言模型（Mistral 7B 或 Llama 3 8B）
> ✅ 能通过 OpenAI API 格式调用
> ✅ 理解显存、延迟、吞吐的真实含义

---

## 🚀 Week 1–2 执行计划总览

| 模块     | 目标                          | 工具 / 框架                       |
| ------ | --------------------------- | ----------------------------- |
| 环境准备   | 拥有一台能运行 vLLM 的 GPU 主机       | Vast.ai / RunPod / Paperspace |
| 模型部署   | 启动 Mistral-7B-Instruct 推理服务 | vLLM                          |
| API 接口 | 用 FastAPI 或 curl 访问         | OpenAI-compatible API         |
| 监控学习   | 查看 GPU 利用率、显存、吞吐            | `nvidia-smi`, Grafana（可选）     |

---

## 🧩 Step 1：租一台合适的云主机

### 💻 推荐配置（约 $1.2 / 小时）

| 组件  | 最低要求                         | 建议           |
| --- | ---------------------------- | ------------ |
| GPU | 1× A100 (40 GB 以上) 或 2× L40S | A100 80GB 最佳 |
| CPU | ≥ 8 vCPU                     |              |
| RAM | ≥ 64 GB                      |              |
| 系统  | Ubuntu 22.04 LTS             |              |
| 存储  | 100 GB SSD                   |              |

推荐平台：

* [https://vast.ai](https://vast.ai)
* [https://runpod.io](https://runpod.io)
* [https://lambdalabs.com](https://lambdalabs.com)

> 📝 若在 Vast.ai 选机，搜索 A100 80GB + Ubuntu 22.04，点击“Start Instance”后即可 SSH 连接。

---

## ⚙️ Step 2：安装基础环境

SSH 登录后执行：

```bash
sudo apt update && sudo apt install -y python3-venv git nvidia-smi
python3 -m venv vllm-env
source vllm-env/bin/activate
pip install --upgrade pip
pip install vllm
```

测试显卡：

```bash
nvidia-smi
```

输出里应能看到 A100 或 L40S GPU 和 显存信息。

---

## 🧠 Step 3：启动一个 vLLM 模型服务

下载并运行：

```bash
python -m vllm.entrypoints.openai.api_server \
  --model mistralai/Mistral-7B-Instruct-v0.2 \
  --port 8000 \
  --tensor-parallel-size 1
```

如果你的机器是 A100 80GB ，可以直接跑；
若显存小（< 40GB），改用 `mistralai/Mistral-7B-v0.1` 或 `TheBloke/Mistral-7B-Instruct-v0.2-GPTQ`（量化版）。

vLLM 会自动从 Hugging Face 下载权重（约 14 GB），下载完毕后启动本地 API 服务。

---

## 🔗 Step 4：测试 OpenAI API 兼容接口

另开一个终端（或在本地），执行：

```bash
curl http://<your_server_ip>:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mistralai/Mistral-7B-Instruct-v0.2",
    "prompt": "Explain why the sky is blue in one paragraph.",
    "max_tokens": 100
  }'
```

返回应类似：

```json
{
  "id": "...",
  "choices": [
    {"text": "The sky appears blue because..."}
  ]
}
```

> 💡 这说明你现在拥有一个自部署版的“ChatGPT API”。

---

## 📊 Step 5：学习观察系统资源

1. **GPU 显存与利用率**

   ```bash
   watch -n 1 nvidia-smi
   ```

   留意：

   * 显存占用（Memory Used）≈ 模型权重 + KV-cache + batch
   * GPU Utilization (%) 代表推理负载

2. **性能测试（可选）**

   ```bash
   pip install locust
   ```

   或直接用 ab / hey 等压测工具，观察 RPS 与延迟变化。

3. **可视化监控（进阶）**

   * 安装 Prometheus + Grafana Docker 栈查看 GPU metrics。

---

## 🧱 Step 6：整理你的第一周学习日志

建议每天记录：

* GPU 显存随 prompt 长度的变化
* 响应延迟随 batch 大小的变化
* 不同模型（Mistral、Llama 3 等）资源对比

这会让你非常直观地理解「为什么 OpenAI 要设计 batching、KV-cache、分布式推理」这些工程逻辑。

---

## 🪜 Step 7（可选）：添加一个前端界面

```bash
pip install gradio
```

```python
import gradio as gr
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="none")

def chat(prompt):
    r = client.completions.create(model="mistralai/Mistral-7B-Instruct-v0.2",
                                  prompt=prompt, max_tokens=200)
    return r.choices[0].text

gr.Interface(fn=chat, inputs="text", outputs="text").launch()
```

> 现在你就有了一个“自己托管的 ChatGPT 网页”。

---

## ✅ Week 1–2 成果检查表

| 能力             | 检查方式                     |
| -------------- | ------------------------ |
| ✅ 能登录 GPU 主机   | `nvidia-smi` 能看到 A100    |
| ✅ 能运行 vLLM API | `curl` 返回正常结果            |
| ✅ 能观测显存        | 显存随 prompt 变化            |
| ✅ 理解基础结构       | 知道什么是模型权重、token、KV cache |
| ✅ 可交互网页（可选）    | Gradio 界面可用              |

---
