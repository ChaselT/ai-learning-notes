---
tags: [AI学习, 阶段1, LLM]
created: 2026-07-24
---

# 多模态视觉API

## 这是什么/为什么重要

视觉模型（VLM）让"传图给 LLM"像传文本一样简单：截图报错发给它直接定位问题、发票拍照直接出结构化数据。对应用工程师来说这不是新接口范式——还是 Chat Completions，只是 `content` 从字符串变成了"文本+图片"的数组。会文本 API 的你，学这个是增量成本极低的能力扩展。

## 核心内容

### 1. 视觉模型能做什么

- **图片问答**：这张图里有什么？两张图有何区别？
- **OCR**：识别票据、截图、手写体文字（配合 [[结构化输出]] 直接出 JSON）
- **截图分析**：报错截图定位问题、UI 截图生成前端代码
- **图表理解**：读折线图/柱状图，提取数据和趋势结论

常见边界：数清密集小物体、精确坐标、复杂表格的行列对齐仍容易出错，关键数据要人工或规则复核。

Java 工程师视角：把 VLM 当成一个"通用视觉微服务"——过去要接 OCR 服务、图像分类服务、表格解析服务三套接口，现在一个 Chat 接口 + 不同 prompt 全覆盖，代价是延迟更高、结果需要校验。

### 2. 传图方式：URL 或 base64

消息的 `content` 变为数组，混排 `text` 与 `image_url` 两种块。本地文件用 base64 data URL：

```python
import base64, os
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])  # 官方端点示例

def img_to_data_url(path: str) -> str:
    with open(path, "rb") as f:
        b64 = base64.b64encode(f.read()).decode()
    ext = path.rsplit(".", 1)[-1].lower()
    return f"data:image/{'jpeg' if ext == 'jpg' else ext};base64,{b64}"

resp = client.chat.completions.create(
    model="gpt-5.6-terra",     # 2026-07 主流：gpt-5.6-sol/terra/luna 三档全支持视觉
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "这张报错截图是什么问题？给出排查步骤。"},
            {"type": "image_url",
             "image_url": {"url": img_to_data_url("error_screenshot.png")}},
            # 公网图片可直接: {"type": "image_url", "image_url": {"url": "https://..."}}
        ],
    }],
)
print(resp.choices[0].message.content)
```

要点：

- 一条消息可带多张图（对比类任务），文本块和图片块的先后顺序会影响模型关注点
- 大图会被服务端缩放，也按 token 计费（一张图几百到上千 token，见 [[Token与上下文窗口]]）
- 传前把图压缩到 2000px 以内可省钱提速；截图类内容保证文字清晰即可

### 3. 通义千问（阿里云端）示例

同样的 openai SDK，换 base_url 和 model 即可：

```python
client = OpenAI(
    api_key=os.environ["DASHSCOPE_API_KEY"],
    # 百炼兼容模式端点，WorkspaceId 在控制台取；老的 dashscope.aliyuncs.com 域名仍在服务但已非文档主推
    base_url=f"https://{os.environ['DASHSCOPE_WORKSPACE_ID']}.cn-beijing.maas.aliyuncs.com/compatible-mode/v1",
)
resp = client.chat.completions.create(
    model="qwen3.7-plus",   # 2026 起通义把视觉能力并进了主力模型，不再有独立的 qwen-vl-* 系列
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "识别发票中的：开票日期、金额、购买方名称，输出JSON。"},
            {"type": "image_url", "image_url": {"url": img_to_data_url("invoice.jpg")}},
        ],
    }],
    response_format={"type": "json_object"},  # OCR + 结构化输出组合拳
)
print(resp.choices[0].message.content)
```

### 4. 本地跑视觉模型的可行性（22G 显存）

结论：**完全可行，而且 2026 年你甚至不用单独装 VL 模型**——`qwen3.5:27b` 这一代主力模型已经原生吃图，文本和视觉是同一个模型。

| Ollama tag | 下载体积 | 你的 22G |
|---|---|---|
| `qwen3.5:27b` | 17 GB | **首选**：一个模型同时管文本和看图，256K 上下文 |
| `qwen3-vl:8b` | 6.1 GB | 轻量看图专用，留显存给别的进程 |
| `qwen3-vl:30b` | 20 GB | 更强看图，贴边能跑 |
| `qwen3-vl:32b` | 21 GB | 极限贴边，KV cache 几乎没余量，不推荐 |

API 调用方式与上面完全一致，base_url 换成 `http://localhost:11434/v1`（见 [[Ollama本地模型]]）。注意 qwen3-vl 系列**要求 Ollama ≥ 0.12.7**，老版本拉下来会跑不起来。

图片越大、越多，激活占用越高——本地跑建议先压缩图片。8B 级 VLM 的 OCR 与常规问答已可用，复杂图表推理与云端旗舰仍有差距；开发调试用本地、生产关键路径用云端，是合理组合。

### 5. 工程实践要点

- **成本**：图片 token 远多于等量文字，批量 OCR 场景先估算再上量（见 [[Token与上下文窗口]]）
- **隐私**：身份证、合同等敏感图片优先走本地 VLM，不出内网
- **可靠性**：OCR 结果务必配合 pydantic 校验 + 重试（复用 [[结构化输出]] 模式），金额/日期等关键字段建议二次规则校验（正则、区间检查）
- **prompt 技巧**：明确告诉模型"图中没有该字段时输出 null"，能显著减少视觉幻觉（把水印当正文、编造看不清的数字）
- **与传统 OCR 的取舍**：版式固定、量大的场景（如统一格式的表单）传统 OCR（PaddleOCR 等）更快更便宜；版式多变、需要"理解"的场景 VLM 优势明显，两者也常组合使用（OCR 出文本 + LLM 做结构化）

## 动手任务

1. 写 `E:\workspace\AiStudy\phase1-llm-api\ex09_vision_basic.py`：截一张你 IDE 的报错图，让本地 `qwen3.5:27b`（或云端 `qwen3.7-plus`）分析报错原因
2. 写 `E:\workspace\AiStudy\phase1-llm-api\ex09_ocr_json.py`：找一张购物小票/发票照片，OCR 后用 pydantic 模型校验输出（复用 [[结构化输出]] 的重试函数），金额字段额外做一次正则/区间校验
3. 对比实验：同一张图表图片分别发给云端模型和本地模型，对比数据提取准确度；再把同一张图压缩到 1000px 和保持原图各跑一次，对比 `usage.prompt_tokens` 的差距，体会"图片很贵"。结论记录到脚本注释

## 相关笔记

- [[02-LLM应用开发-MOC]]
- [[Chat Completions API]]
- [[结构化输出]]
- [[Token与上下文窗口]]
- [[Ollama本地模型]]
- [[Prompt工程]]
- [[AI学习地图]]
