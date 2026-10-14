# 售后资格裁定助手

电商售后资格的自动裁定系统。顾客用大白话描述诉求，系统抽取关键事实 → 检索店铺售后规则 →
用代码校验硬性条件 → 输出**是否受理的裁定 + 依据条款 + 客服话术**；信息不足主动追问，规则覆盖不到转人工。

> AI 智能体项目 · A 档 · 单人作业
> 题目方向：售后资格裁定
---

## 一、环境要求

- **Python 3.10 或更高**- 一个 **DeepSeek API Key**（[platform.deepseek.com](https://platform.deepseek.com) 申请）
- 无需显卡，本地就能跑

检查 Python 版本：

```bash
py -3 --version
```

> 如果你的系统里 `python` 命令可用，下文所有 `py -3` 都可以换成 `python`。

---

## 二、安装

在项目根目录执行：

```bash
py -3 -m pip install -r requirements.txt
```

依赖只有三个：`streamlit`、`requests`、`python-dotenv`。

---

## 三、配置 API Key

**这一步不能跳过**——任务书要求"必须调用真实模型"，不配 Key 会走模拟模式。

1. 复制 `.env.example` 为 `.env`（如果 `.env` 已存在则直接编辑）
2. 填入你的 Key：

```ini
DEEPSEEK_API_KEY=sk-你的密钥
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_MODEL=deepseek-chat
```

> `.env` 已在 `.gitignore` 中，**不会随代码提交**。

---

## 四、运行

### 4.1 启动网页界面（主要演示方式）

```bash
py -3 -m streamlit run app.py
```

浏览器会自动打开 `http://localhost:8501`。

界面分两栏：左侧是客服对话，右侧实时展示**识别到的要素 / 检索到的条款及得分 / 硬条件校验明细 / 执行轨迹**。

侧边栏有 5 个示例场景可以一键加载，方便快速演示。

### 4.2 运行测试

```bash
# 跑全部测试并生成报告（会真实调用模型，约 35 秒）
py -3 tests/run_tests.py

# 只跑规则引擎单元测试（不调用模型、不耗 token、1 秒内完成）
py -3 tests/run_tests.py --unit-only

# 只跑指定用例
py -3 tests/run_tests.py --case T03
```

报告输出到 `docs/测试报告.md`。

### 4.3 检查模型连接

```bash
py -3 -m src.llm
```

会打印当前运行模式、请求的模型名、以及一次真实的往返调用结果。

---

## 五、目录结构

```
AI智能体项目_A档_张_2/
├── app.py                  Streamlit 界面入口
├── requirements.txt
├── .env                    你的密钥（不要提交）
├── .env.example            配置模板
│
├── src/
│   ├── config.py           配置集中管理（温度/超时/重试都可在 .env 调）
│   ├── llm.py              模型调用层：Provider 抽象 + 重试 + 全量调用日志
│   ├── prompts.py          提示词集中管理（带版本号，便于对比改动影响）
│   ├── kb.py               知识库加载与品类匹配
│   ├── schema.py           要素结构 + "还缺什么、该问什么"的判定规则
│   ├── extractor.py        要素抽取：口语 → 结构化，并做本地校正
│   ├── retriever.py        规则检索：结构化过滤 + 关键词打分
│   ├── rules.py            硬条件校验：代码算出确定性结论
│   ├── adjudicator.py      裁定编排：两道防线的所在地
│   ├── session.py          多轮会话状态机
│   └── mock_llm.py         无 Key 时的兜底（非真实模型）
│
├── data/
│   ├── policy_kb.json      售后规则知识库（16 品类 + 16 条款）
│   └── testcases.json      测试用例集
│
├── tests/
│   └── run_tests.py        测试运行器
│
├── docs/
│   ├── 项目说明.md          做了什么、怎么做的、有什么限制
│   ├── 测试报告.md          自动生成
│   ├── AI编程记录.md        4 段编程过程记录
│   └── 演示PPT.pptx         演示材料
│
└── logs/
    └── llm_calls.jsonl     每次模型调用的完整记录（自动生成）
```

---

## 六、常见问题

**Q：报错「鉴权失败（HTTP 401）」**
A：`.env` 里的 `DEEPSEEK_API_KEY` 没填或填错了。去平台确认 Key 是否有效、账户是否有余额。

**Q：界面顶部显示"运行模式：Mock（模拟）"**
A：说明没读到 Key，走的是规则模拟。此时**不满足任务书"必须调用真实模型"的要求**。
检查 `.env` 文件是否在项目根目录、`DEEPSEEK_API_KEY=` 后面有没有内容、有没有多余空格。

**Q：测试跑得慢**
A：集成测试每一步都要真实调用模型，12 个用例约 35 秒是正常的。
如果只想验证判定逻辑，用 `--unit-only`，1 秒内出结果。

**Q：想换模型怎么办**
A：改 `.env` 里的 `DEEPSEEK_MODEL` 即可，代码不用动。
注意接口返回的模型名可能与请求的不同（本项目观测到 `deepseek-chat` 被路由为 `deepseek-flash`），
`logs/llm_calls.jsonl` 里记录的是**实际**返回值。

**Q：想改售后规则**
A：直接编辑 `data/policy_kb.json`。品类例外、拆封限制都是数据驱动的，
改数据即改逻辑，不需要动 Python 代码。改错了会在启动时立刻报错。

**Q：怎么查看某次裁定到底发生了什么**
A：两种方式——界面右栏的「执行轨迹」面板，或者 `logs/llm_calls.jsonl`
（每次模型调用的完整提示词和原始返回都在里面）。

---

## 七、说明

- 知识库条款结构参照《消费者权益保护法》与《网络购买商品七日无理由退货暂行办法》编写，
  **内容为课程作业虚构**，不构成任何真实商家的售后承诺。
- 本系统输出仅供辅助判断，涉及金额或责任的最终裁定建议由人工复核。
