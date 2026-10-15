# english-reader-model-public

轻阅英语（LightRead）**端侧本地模型**的发布仓库：语法剖析 + 整句翻译用的自训练权重。

基座 `Qwen3-1.7B` + R5_repro LoRA（QLoRA r16 α32）合并后量化为 GGUF。

## 身份（勿改）

| 项 | 值 |
|---|---|
| 基座 | `unsloth/Qwen3-1.7B-unsloth-bnb-4bit` |
| **R5_repro LoRA** | sha256 `3972c3bd8b3719b833bf104d439d952a9f1301b6f28897b0421b3b12a5f35bb5` |
| 标注规范 | `ANNOTATION_SPEC v1.1`（冻结） |
| schema | `v3-pilot`（不含 tags） |
| Chat 模板 | `enable_thinking=false`（空 think 块 `<think>\n\n</think>\n\n`） |
| 转换 | unsloth `save_method=merged_16bit` → llama.cpp `0.6.0 build 11429 commit d81235049` |

## 发布物（Release 资产）

| 文件 | 字节 | sha256 |
|---|---|---|
| `Qwen3-1.7B-R5repro-Q5_K_M.gguf` | 1,257,879,872 | `31681ae4c7827ac519e518d85ee80e63f18e9452c910ce249538a3f78261616c` |
| `training_bundle_r5repro.tar.gz` | 66,807,102 | `7882afa2a5e5bf54e5ed9421b88d169e66f36ea16bc187b08e0f26d86931f766` |
| `CONVERSION_MANIFEST.json` | — | 转换全记录（含 F16 / Q8_0 / Q4_K_M 指纹） |
| `Modelfile-R5repro` | — | llama.cpp Modelfile |

下载直链（App 使用的就是它）：

```
https://github.com/robin421/english-reader-model-public/releases/download/r5repro-q5_k_m-v1/Qwen3-1.7B-R5repro-Q5_K_M.gguf
```

## 量化取舍（已在 154 句开发集 + 42 句盲测集上验证）

- ✅ **Q5_K_M 唯一采用**：S+V 80.5% / 骨架 59.1% / clause F1 0.5038，≈ F16（77.9%）。
- ⛔ **Q4_K_M 禁用**：S+V 相对 F16 **−11.7pp**，超出 5pp 阈值。
- Q8_0 / F16 不设备选（仅作对照；F16 仅作再量化源，体积过大）。

## App 侧接线

`lib/ai/ai_model_manager.dart` 与本仓库必须一致（改一处要同步改另一处）：

| 常量 | 值 |
|---|---|
| `fileName` | `Qwen3-1.7B-R5repro-Q5_K_M.gguf` |
| `expectedBytes` | `1257879872` |
| `modelVersion` / `modelId` | `qwen3-1.7b-r5repro-q5_k_m-v1` / `qwen3-1.7b-r5repro-q5_k_m` |
| `downloadUrls` | 本 release 直链 + `ghfast.top` / `ghproxy.net` 两个加速镜像 |

> `expectedBytes` 是完整性判定的基准（用 `>=`）：换文件必须同步改，
> 否则截断文件会被误判为「已安装」。

## training_bundle_r5repro.tar.gz 内容

自包含训练材料，可直接复用再训练：

```
data/      train_v5.jsonl(554) inner_val_v5.jsonl(67)
           eval_holdout154.jsonl(154) blind_set_v3.jsonl(42) new_sentences_r5.jsonl
config/    RUN_CONFIG_R5repro.md train_log_r5repro.json chatml_template.py
           syntax_schema.py syntax_v3_pilot.schema.json ANNOTATION_SPEC.md
scripts/   colab_setup.py colab_r5repro_driver.py colab_r5repro_run.sh build_dataset_r5.py
adapter_r5repro/  ★ adapter_model.safetensors（70MB，不可再生）+ tokenizer + chat_template
```

每个文件的 bytes + sha256 见包内 `MANIFEST.json`。

## 约束

- 不改 `ANNOTATION_SPEC v1.1`；不新增标签；不改金标。
- ⛔ `eval_holdout154.jsonl` / `blind_set_v3.jsonl` **禁止用于训练**。
- 免费 NVIDIA NIM 仅限评估，不得用于生成训练数据。
