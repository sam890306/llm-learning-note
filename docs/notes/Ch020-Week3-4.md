# Week 3–4：LoRA/QLoRA 微调实操

目标是：用你自己的少量领域数据，把 7B~8B 开源模型微调到“好用”，并 **无缝接到你前面已经跑通的 vLLM 服务**。

---

# Week 3–4：从 0 到可用的 LoRA 微调

## 0) 环境与硬件（沿用你前面的实例即可）

* **云机**：A100 40–80GB（更舒服）或 24GB 级（走 QLoRA）
* **系统**：Ubuntu 22.04，CUDA 已可用
* **Python 依赖**：

```bash
source vllm-env/bin/activate  # 继续用之前的虚拟环境也行
pip install --upgrade pip
pip install "transformers>=4.44" datasets peft accelerate bitsandbytes trl evaluate rouge-score
```

---

## 1) 准备你的训练数据（越贴近你的真实业务越好）

最简单有效的是 **SFT 指令微调** 格式，JSONL 每行一条样本：

**schema A（通用 instruction 格式）**

```json
{"instruction":"给我一段Python函数，打印1到10","input":"","output":"def f():\n    for i in range(1,11):\n        print(i)"}
```

**schema B（chat 格式）** —— 适配对话模型

```json
{"messages":[
  {"role":"system","content":"你是资深平台工程师，回答只给可运行代码与要点"},
  {"role":"user","content":"写个Terraform例子，用变量控制地域"},
  {"role":"assistant","content":"# main.tf...\nprovider ..."}
]}
```

建议：

* **数量**：先做 300–2,000 条高质量样本（比量更重要）
* **风格统一**：输出格式要稳定（比如总是 Markdown 代码块+简短说明）
* **切分**：`train.jsonl` / `eval.jsonl`（9:1）

> 快速把你手头 FAQ/工单/文档转为 JSONL？写个小脚本正则清洗即可；不需要完美，先跑起来。

---

## 2) 选一个基座模型（任选其一）

* **Mistral-7B-Instruct**：稳、资源友好
  `mistralai/Mistral-7B-Instruct-v0.2`
* **Llama-3-8B-Instruct**：中文+英文都不错
  `meta-llama/Meta-Llama-3-8B-Instruct`（需同意条款）

---

## 3) 训练两条路径（二选一）

### 路径 A：LoRA（显存 ≥ 40GB，效果更佳）

```python
# train_lora.py
import json, datasets
from transformers import AutoTokenizer, AutoModelForCausalLM
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from trl import SFTTrainer, SFTConfig

BASE="mistralai/Mistral-7B-Instruct-v0.2"  # 或 Llama3-8B
train_path="train.jsonl"; eval_path="eval.jsonl"

def load_jsonl(path):
    return datasets.Dataset.from_list([json.loads(l) for l in open(path)])

ds_train = load_jsonl(train_path)
ds_eval  = load_jsonl(eval_path)

tok = AutoTokenizer.from_pretrained(BASE, use_fast=True)
if tok.pad_token is None: tok.pad_token = tok.eos_token

model = AutoModelForCausalLM.from_pretrained(
    BASE, torch_dtype="auto", device_map="auto"
)

# 选常见注意力投影为 LoRA 目标
lora = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj","k_proj","v_proj","o_proj","gate_proj","up_proj","down_proj"],
    bias="none", task_type="CAUSAL_LM"
)
model = get_peft_model(model, lora)

def format_sample(example):
    if "messages" in example:
        # chat 格式 → 拼成一个prompt
        msgs = example["messages"]
        text = ""
        for m in msgs:
            role = m["role"]
            text += f"<|{role}|>: {m['content']}\n"
        return {"text": text}
    # instruction 格式
    return {"text": f"指令：{example['instruction']}\n输入：{example.get('input','')}\n回答：{example['output']}"}

ds_train = ds_train.map(format_sample, remove_columns=ds_train.column_names)
ds_eval  = ds_eval.map(format_sample,  remove_columns=ds_eval.column_names)

cfg = SFTConfig(
    output_dir="outputs_lora",
    per_device_train_batch_size=2,
    gradient_accumulation_steps=8,
    num_train_epochs=2,
    learning_rate=2e-4,
    logging_steps=20,
    save_steps=500,
    eval_strategy="steps",
    eval_steps=500,
    max_seq_length=2048,
    bf16=True
)

trainer = SFTTrainer(
    model=model, tokenizer=tok,
    train_dataset=ds_train, eval_dataset=ds_eval,
    args=cfg, dataset_text_field="text"
)
trainer.train()
trainer.save_model("outputs_lora")
```

运行：

```bash
accelerate launch --mixed_precision=bf16 train_lora.py
```

### 路径 B：**QLoRA（4-bit）**（显存 24GB 也能跑）

把基座加载为 4bit，再做 LoRA：

```python
# 仅展示不同点
from transformers import BitsAndBytesConfig
bnb = BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_use_double_quant=True,
                         bnb_4bit_quant_type="nf4", bnb_4bit_compute_dtype="bfloat16")
model = AutoModelForCausalLM.from_pretrained(
    BASE, quantization_config=bnb, device_map="auto"
)
model = prepare_model_for_kbit_training(model)
# 下面同路径A
```

> 提示：若 OOM，降低 `r`，或减小 `max_seq_length`，或增大 `gradient_accumulation_steps`。

---

## 4) 训练时长 & 成本（参考）

* **数据 1k 条，2 epoch**：A100 80GB 约 1–2 小时；24GB（QLoRA）约 2–4 小时
* 成本粗算：$1.2/h × 4h ≈ **$5** 左右/次实验（很友好）

---

## 5) 简单评估（别卷复杂指标，先对效果）

* **小集 eval**：准备 50–100 条看家提问，跑微调前后对比
* **自动指标**：ROUGE/L/BLEU 粗看趋势（`evaluate` 包）
* **人工打分**：一致性、格式、是否幻觉（最重要）

---

## 6) 导出与部署（接回 vLLM）

### 方案 1：**保持 LoRA 适配器**（推荐，灵活）

vLLM 现已支持热加载 LoRA 适配器。先保留 `outputs_lora`（含 adapter_config、adapter_model.bin）。

启动 vLLM（基座 + LoRA）：

```bash
python -m vllm.entrypoints.openai.api_server \
  --model mistralai/Mistral-7B-Instruct-v0.2 \
  --port 8000 \
  --tensor-parallel-size 1 \
  --max-model-len 4096 \
  --enable-lora \
  --lora-modules mylora=./outputs_lora
```

调用时指定 LoRA：

```bash
curl http://<ip>:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mistralai/Mistral-7B-Instruct-v0.2",
    "prompt": "（你的领域问题）",
    "max_tokens": 200,
    "extra_body": {"lora_adapter": "mylora"}
  }'
```

### 方案 2：**合并权重（便于分发，略失灵活）**

```python
from peft import PeftModel
from transformers import AutoModelForCausalLM
base = AutoModelForCausalLM.from_pretrained(BASE, torch_dtype="auto")
ft = PeftModel.from_pretrained(base, "outputs_lora")
merged = ft.merge_and_unload()
merged.save_pretrained("merged_model")
```

然后让 vLLM 直接加载 `merged_model`。

---

## 7) 最小可行的“上线”动作

* **保底监控**：继续 `watch -n 1 nvidia-smi`；高阶：Prometheus + Grafana
* **速率限制**：Nginx/Traefik 做限流，避免一次性打满 GPU
* **日志**：保存输入/输出样例（注意隐私），迭代修数据

---

## 8) 常见坑位速查

* **显存炸**：缩短上下文、减 batch、QLoRA、减少 LoRA target modules
* **训练无提升**：数据脏或风格不统一；调大学习率/epoch；把系统 prompt 写清楚
* **部署答案退化**：没带 LoRA 适配器；或把合并模型加载错路径
* **中文效果差**：换 Llama-3-Instruct 或在数据里加中文风格/术语

---

## 9) 你可以立刻动手的小清单

1. 选模型：Mistral-7B-Instruct 或 Llama-3-8B-Instruct
2. 清洗 300–1000 条高质量 QA 到 `train.jsonl` / `eval.jsonl`
3. 路径 A（LoRA）或 B（QLoRA）二选一，直接跑 `accelerate launch ...`
4. 用 **方案 1** 把 LoRA 适配器热挂进 vLLM，接口保持不变
5. 做一张“前后对比”的小评估表，决定要不要继续加数据/调参

---
