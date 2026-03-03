# 🎙️ Interview LevelUp

> AI 驱动的模拟面试平台 — 按岗位和级别进行全流程面试练习，获得实时评分与语音交互体验。

用户输入目标岗位和级别，AI 面试官便进入角色：逐轮提问、评估回答、在薄弱点深挖追问，最终给出结构化复盘报告。全程支持语音作答与朗读，尽量还原真实面试的紧张感。

---

## 仓库构成

| 仓库 | 职责 |
|---|---|
| [interview-levelup-backend](https://github.com/interview-levelup/interview-levelup-backend) | 🦫 Go + Gin REST API，用户认证、面试生命周期管理、SSE 推流 |
| [interview-levelup-agent](https://github.com/interview-levelup/interview-levelup-agent) | 🤖 Python + LangGraph AI 面试官，出题 / 评分 / 追问决策 |
| [interview-levelup-website](https://github.com/interview-levelup/interview-levelup-website) | 🎙️ React + TypeScript 前端，流式对话 UI、语音输入、TTS 朗读 |

---

## 系统架构

```
  Browser (React + TypeScript)
 ┌─────────────────────────────────────────────────┐
 │                                                 │
 │  ┌─────────────────────────────────────────┐    │
 │  │            InterviewPage                │    │
 │  │   streaming bubbles / TTS playback      │    │
 │  └───────────────┬─────────────────────────┘    │
 │                  │ SSE + REST (fetch / axios)   │
 │  ┌───────┐  ┌────┴────┐                         │
 │  │  STT  │  │  TTS    │                         │
 │  │WebSpch│  │WebSpeech│                         │
 │  │/Whspr │  │  API    │                         │
 │  └───────┘  └─────────┘                         │
 └─────────────────┼───────────────────────────────┘
                   │
                   ▼
  Backend (Go + Gin)
 ┌─────────────────────────────────────────────────┐
 │                                                 │
 │  JWT Auth  │  PostgreSQL 16  │  Whisper direct  │
 │                                                 │
 │  POST /interviews/stream      ← create + SSE    │
 │  POST /interviews/:id/answer/stream  ← answer   │
 │  POST /interviews/:id/transcribe    ← audio     │
 │                                                 │
 └─────────────────┬───────────────────────────────┘
                   │  HTTP  /chat/stream
                   ▼
  Agent (Python + FastAPI + LangGraph)
 ┌─────────────────────────────────────────────────┐
 │                                                 │
 │  route_entry                                    │
 │   ├─ no answer ──► generate_question ──► END    │
 │   └─ has answer ─► check_sub                    │
 │                        │                        │
 │           ┌────────────┼───────────┐            │
 │         SUB          END         ANSWER         │
 │           │      (user_end)        │            │
 │      handle_sub       │    evaluate_answer      │
 │      (reply to        │    (score 0-100,        │
 │      candidate)       │    detail breakdown)    │
 │           │           │            │            │
 │          END          │     decide_next_step    │
 │                       │      │    │    │   │    │
 │                       │     fu   next  ✓   ✗    │
 │                       │      │    │    │   │    │
 │                       │  gen_fu gen_q  │   │    │
 │                       │    │    │      │   │    │
 │                       │   END  END     │   │    │
 │                       │           finished abort│
 │                       └──────────────┬──────┘   │
 │                                      ▼          │
 │                               generate_report   │
 │                                      │          │
 │                                     END         │
 │                                                 │
 │  Question/report nodes stream tokens via SSE    │
 │  (report tokens use separate event type so      │
 │   they never appear in the chat bubble)         │
 │                                                 │
 └─────────────────────────────────────────────────┘
```

### 关键数据流

```
用户提交回答
    │
    ▼
backend  POST /interviews/:id/answer/stream
    │  持久化答案
    ▼
agent  POST /chat/stream
    │
    ├─ check_sub: 三分类
    │    ├─ SUB  → handle_sub → 回答候选人问题 → END
    │    ├─ END  → generate_report（用户主动结束）
    │    └─ ANSWER → evaluate_answer → decide_next_step
    │                        │
    │         ┌──────────────┼──────────────┐
    │       next/fu      finished         aborted
    │         │         (轮次跑完)      (即时辱骂或
    │    继续面试              └──────┬──────┘ 累计消极)
    │    SSE token ──► frontend       ▼
    │                          generate_report
    │                    report_token 事件（Go 层丢弃，
    │                    不进 chat bubble / 不被 TTS 朗读）
    │                          │
    │                        done 事件携带完整报告
    ▼
backend 收到 done → 持久化 → 推 saved 给前端
    ▼
frontend rounds 更新 → TTS 逐句朗读
```

---

## 核心功能

- **LangGraph 多节点 Agent** — 出题、评分、追问、反问检测、终止判断、最终报告，完整模拟真实面试官决策链路
- **候选人反问支持** — 检测候选人是否在问面试官问题，如是则由 `handle_sub` 作答后交还控制权，不计入正式轮次
- **实时流式对话** — SSE 推流 + 流式气泡渲染，面试官"边想边说"，切换时无闪烁
- **语音双向** — Web Speech API / OpenAI Whisper 语音输入；TTS 句子队列朗读（1.5× 速），不因新 token 到来中断
- **即时导航** — 新建面试时后端一创建 DB 行即推 `created` 事件，前端立刻跳转，首问在后台并行生成
- **评分复盘** — 每条回答附 0–100 分与多维评价详情，面试结束展示结构化总结报告
