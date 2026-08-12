# HUNT V0 — BUILD SPECIFICATION

ได้เลย เราเข้าสู่ **Build Phase** แล้ว

ผมจะถือบทบาทเป็น technical partner ของ Hunt ต่อจากนี้: เราจะไม่คุยแค่ architecture แต่จะค่อย ๆ **เลือก → สร้าง → รัน → ทดสอบ → วัดผล → แก้ → version** จน Hunt ใช้งานได้จริง

จาก research ล่าสุด ผมขอ freeze architecture แรกไว้ก่อน และมีการปรับหนึ่งเรื่องจาก teardown เดิม:

> **Hunt V0 จะไม่สร้าง coding engine เอง และจะไม่สร้าง multi-agent swarm**
>
> เราจะสร้าง “Engineering Brain” ครอบอยู่บน coding hand ที่มีอยู่แล้ว

Hermes มี architecture ที่น่าสนใจมากสำหรับเรา เพราะแยก agent loop, tools, memory, skills, context compression และ gateway ออกจากกันชัดเจน และมี learning loop สำหรับสร้าง/ปรับ skills จากประสบการณ์จริง ([GitHub][1])

---

# HUNT V0 — BUILD SPECIFICATION

## Mission

> **Build the first engineering partner of Anant Agent.**

Hunt ต้องรับ:

> “สร้าง X ให้ผม”

แล้วสามารถ:

```text
Understand
   ↓
Research
   ↓
Design
   ↓
Plan
   ↓
Build
   ↓
Test
   ↓
Review
   ↓
Report
```

ได้ด้วยตัวเอง

แต่ **ต้องไม่ sacrifice engineering quality เพื่อทำตามคำสั่งเร็ว ๆ**

---

# 1. Architecture V0

```text
                         MOS
                          │
                          │
                    Product Intent
                          │
                          ▼
                ┌───────────────────┐
                │       HUNT        │
                │                   │
                │ Engineering Brain │
                └─────────┬─────────┘
                          │
                   ┌──────▼──────┐
                   │   ROUTER    │
                   └──────┬──────┘
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
      RESEARCH          PLAN            BUILD
          │               │                │
          └───────────────┼────────────────┘
                          │
                          ▼
                       VERIFY
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
          Tests        Security     Quality
             │            │            │
             └────────────┼────────────┘
                          ▼
                       REPORT
```

โดย implementation จริง:

```text
Python
│
├── LangGraph
│
├── LLM
│
├── Tools
│   ├── filesystem
│   ├── shell
│   ├── git
│   ├── web
│   └── MCP
│
├── Coding Hand
│   └── Codex CLI
│
├── Memory
│   ├── SQLite
│   ├── Markdown
│   └── Git
│
└── Skills
```

LangGraph เหมาะกับตรงกลางเพราะ checkpoint state ของ graph ได้ และสามารถ pause/resume ผ่าน interrupt + persistent checkpointer ซึ่งเป็น primitive ที่เราต้องการสำหรับ human approval และ long-running engineering tasks ([Docs by LangChain][2])

---

# 2. ทำไม Coding Hand แยกจาก Hunt

นี่เป็น architectural decision สำคัญ

สมมติ Hunt ตัดสินใจ:

> “Implement this feature.”

Hunt ไม่ต้องรู้วิธีเขียน Python ทุกบรรทัด

มันสามารถเรียก:

```text
CODING_HAND.run(
    task="Implement authentication",
    workspace="./project"
)
```

แล้ว Coding Hand ไปทำงาน

ตอนนี้เราใช้ **Codex CLI** เป็น candidate แรก เพราะมันเป็น coding agent ที่รัน local ใน terminal โดยตรง และมีทั้ง installer, npm และ Homebrew distribution อยู่แล้ว ([GitHub][3])

แต่ interface ของ Hunt จะไม่ผูกกับ Codex:

```text
CodingHand
    │
    ├── Codex
    ├── ClaudeCode
    ├── OpenHands
    └── Future
```

วันหนึ่งเรา benchmark แล้วพบว่า Claude Code ดีกว่า task แบบหนึ่ง:

```text
if task == "large_refactor":
    use ClaudeCode
else:
    use Codex
```

Hunt เป็นคนตัดสินใจ

---

# 3. Hunt State Machine

นี่คือหัวใจของ V0

```text
START
  │
  ▼
UNDERSTAND
  │
  ├── unclear ──────→ ASK_MOS
  │                       │
  │                       ▼
  └──────────────────── UNDERSTAND
  │
  ▼
RESEARCH
  │
  ├── enough knowledge
  │
  ▼
DESIGN
  │
  ▼
PLAN
  │
  ├── risky?
  │      │
  │      └────────→ APPROVAL
  │                    │
  │                    ▼
  └────────────────── BUILD
                         │
                         ▼
                       TEST
                         │
                   ┌─────┴─────┐
                   │           │
                 FAIL        PASS
                   │           │
                   ▼           ▼
                 DEBUG       REVIEW
                   │           │
                   └──→ TEST   │
                               ▼
                             SHIP
                               │
                               ▼
                             LEARN
                               │
                               ▼
                              END
```

นี่คือสิ่งที่ทำให้ Hunt เป็น **Agent** ไม่ใช่ chatbot

---

# 4. State ของ Hunt

V0 จะมี state ประมาณนี้:

```python
class HuntState:
    request
    intent
    requirements

    research_findings

    architecture
    plan

    current_task
    task_results

    test_results
    security_findings
    technical_debt

    approval_required
    approval_result

    lessons
    final_report
```

ยังไม่ต้องมี 50 fields

**Keep it boring.**

---

# 5. Engineering Skills

เราจะเอาแนวคิดที่ผมคิดว่าน่าสนใจที่สุดจาก Hermes มาใช้

Hermes แยก **memory = what** กับ **skills = how** และโหลด skills แบบ progressive disclosure เพื่อไม่เอาความรู้ทั้งหมดเข้า context ตั้งแต่ต้น ([GitHub][4])

Hunt จะทำเหมือนกัน

```text
skills/

requirements/
    SKILL.md

research/
    SKILL.md

architecture/
    SKILL.md

planning/
    SKILL.md

implementation/
    SKILL.md

debugging/
    SKILL.md

testing/
    SKILL.md

security/
    SKILL.md

technical-debt/
    SKILL.md

code-review/
    SKILL.md

experimentation/
    SKILL.md
```

แต่ **Hunt จะไม่โหลดทั้งหมด**

Router จะเลือก:

```text
Task:
"Build authentication"

→ requirements
→ research
→ architecture
→ implementation
→ security
→ testing
```

นี่ทำให้ context ไม่บวม

---

# 6. Engineering Constitution V0

ไฟล์แรกของ Hunt:

```text
constitution/engineering.md
```

เนื้อหาหลัก:

### HUNT PRINCIPLES

**Understand before build.**

**Prefer the simplest architecture that satisfies the requirement.**

**Research before reinventing.**

**Measure before optimizing.**

**Security is part of design.**

**Technical debt must be explicit.**

**Never hide uncertainty.**

**Prefer reversible decisions.**

**Use experiments when reasoning alone is insufficient.**

**Challenge the founder when technical judgment disagrees.**

**Do not introduce complexity without measurable value.**

**A feature is not finished until it is verified.**

**Leave the system better than you found it.**

นี่คือ “นิสัย” ของ Hunt

ไม่ใช่ personality prompt

---

# 7. Memory Architecture

เราจะเริ่มง่ายกว่าที่คิด

```text
memory/
├── MEMORY.md
├── decisions/
├── lessons/
├── research/
└── sessions/
```

### MEMORY.md

สิ่งที่ Hunt ต้องจำถาวร

เช่น:

```text
Anant uses Python for backend services.

Current engineering priority:
Build Hunt V0.

Production deployments require human approval.

Prefer simple infrastructure during garage stage.
```

### decisions/

Architecture Decision Records:

```text
ADR-001-coding-hand.md
ADR-002-agent-runtime.md
ADR-003-memory.md
```

### lessons/

สิ่งที่ Hunt เรียนจาก failure

```text
LESSON-001.md
LESSON-002.md
```

---

# 8. Self-Improvement V0

ยังไม่ให้ Hunt แก้ source code ตัวเอง

เราจะเริ่มจาก:

```text
Task
 ↓
Outcome
 ↓
Evaluate
 ↓
Lesson
 ↓
Skill improvement
```

ตัวอย่าง:

Hunt ลอง deploy แล้ว fail เพราะไม่ได้ตรวจ environment variable

มันเขียน:

```text
lesson:
Always verify required environment variables
before deployment.
```

แล้ว update skill:

```text
skills/deployment/SKILL.md
```

แต่ **การเปลี่ยน core system ต้องผ่าน approval**

นี่เป็นบทเรียนสำคัญจากระบบ self-improving อย่าง Hermes: การมี memory/skills ที่ persistent มีประโยชน์มาก แต่ก็ต้องมี governance เพราะระบบที่มี memory และ skills สามารถเกิดปัญหาการทำตามกฎหรือ stale knowledge ได้จริง ([GitHub][5])

---

# 9. Security Boundary

ตั้งแต่วันแรก

Hunt มี 3 ระดับ:

### SAFE

ทำได้ทันที:

```text
read files
search web
git status
git diff
run tests
inspect logs
```

### WRITE

ทำได้ใน workspace:

```text
create file
edit file
install package
git commit
```

### DANGEROUS

ต้องถาม Mos:

```text
delete large directory
push production
deploy
change secrets
modify system config
database destructive operation
financial action
```

LangGraph interrupt เหมาะกับ approval gate แบบนี้โดยตรง เพราะสามารถ pause graph แล้ว resume ด้วย human decision โดย state ถูก checkpoint ไว้ ([Docs by LangChain][6])

---

# 10. MCP

เราจะสร้าง Tool interface ให้ Hunt เป็น MCP-compatible ตั้งแต่ต้น แต่ไม่สร้าง MCP ecosystem เอง

Official MCP Python SDK ปัจจุบันเป็น v2 stable และรองรับทั้งการสร้าง server/client รวมถึง stdio, Streamable HTTP และ SSE ([MCP Python SDK][7])

ดังนั้น Hunt:

```text
Hunt
 │
 ├── MCP GitHub
 ├── MCP Browser
 ├── MCP Filesystem
 ├── MCP Research
 └── future MCPs
```

แต่ V0 เริ่มจาก local tools ก่อน

MCP เป็น **extension port**

ไม่ใช่ core dependency ของ brain

---

# 11. Research Engine

นี่เป็นส่วนที่ผมอยากให้ Hunt แตกต่าง

Hunt ต้องสามารถรับ:

> “Research วิธีสร้าง local agent ที่สามารถแก้ code autonomously”

แล้วทำ:

```text
Search
 ↓
Collect
 ↓
Filter
 ↓
Read docs
 ↓
Read source
 ↓
Compare
 ↓
Experiment
 ↓
Recommendation
```

ผลลัพธ์ต้องไม่ใช่ summary อย่างเดียว

แต่:

```text
RECOMMENDATION

Use:
X

Because:
A
B
C

Reject:
Y

Because:
D
E

Experiment:
Z

Expected benefit:
...
```

---

# 12. Hunt Evaluation

เราไม่สามารถบอกว่า Hunt “เก่ง” เพราะมันคุยดูฉลาด

เราต้องมี benchmark

### Level 1

```text
Can it create a Python project?
```

### Level 2

```text
Can it implement a feature?
```

### Level 3

```text
Can it debug a failing test?
```

### Level 4

```text
Can it research an unfamiliar technology?
```

### Level 5

```text
Can it choose between architectures?
```

### Level 6

```text
Can it identify security problems?
```

### Level 7

```text
Can it recognize technical debt?
```

### Level 8

```text
Can it learn from a failure?
```

### Level 9

```text
Can it improve a skill?
```

### Level 10

```text
Can it independently ship a small product?
```

**Hunt จะไม่มีคำว่า “เก่งแล้ว” จนกว่าจะผ่าน benchmark**

---

# 13. Apple I Test

นี่คือ acceptance test ของ Hunt V0

เราจะให้ Hunt โจทย์:

> **Build a small web application from an empty directory.**

Requirements:

```text
1. Python backend
2. Simple frontend
3. Persistent data
4. Tests
5. README
6. Git repository
7. Security review
```

Mos ไม่เขียน code

Mos ไม่บอก architecture

Mos บอกแค่:

> “Build it.”

แล้ว Hunt ต้อง:

```text
research
→ architecture
→ plan
→ implementation
→ tests
→ security review
→ documentation
→ git commit
→ report
```

ถ้าทำได้:

**Hunt V0 works.**

---

# 14. Repository Structure

นี่คือ repo ที่เราจะสร้าง:

```text
anant-hunt/
│
├── README.md
├── pyproject.toml
├── .env.example
├── .gitignore
│
├── hunt/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── core/
│   │   ├── state.py
│   │   ├── graph.py
│   │   ├── router.py
│   │   └── policies.py
│   │
│   ├── brain/
│   │   ├── llm.py
│   │   ├── reasoning.py
│   │   └── prompts.py
│   │
│   ├── tools/
│   │   ├── shell.py
│   │   ├── filesystem.py
│   │   ├── git.py
│   │   └── research.py
│   │
│   ├── hands/
│   │   ├── base.py
│   │   └── codex.py
│   │
│   ├── memory/
│   │   ├── store.py
│   │   ├── lessons.py
│   │   └── decisions.py
│   │
│   └── skills/
│       ├── loader.py
│       └── registry.py
│
├── constitution/
│   └── engineering.md
│
├── skills/
│   ├── requirements/
│   ├── research/
│   ├── architecture/
│   ├── planning/
│   ├── implementation/
│   ├── debugging/
│   ├── testing/
│   ├── security/
│   ├── technical-debt/
│   ├── code-review/
│   └── experimentation/
│
├── memory/
│   ├── MEMORY.md
│   ├── decisions/
│   ├── lessons/
│   └── research/
│
├── tests/
│   ├── test_router.py
│   ├── test_skills.py
│   └── test_policies.py
│
└── docs/
    ├── architecture.md
    ├── evaluation.md
    └── security.md
```

---

# 15. Technology Stack — LOCKED FOR V0

| Layer       | Choice                | เหตุผล            |
| ----------- | --------------------- | ----------------- |
| Language    | **Python**            | ecosystem + AI    |
| Runtime     | **LangGraph**         | stateful workflow |
| Brain       | Frontier LLM          | ไม่ train model   |
| Coding Hand | **Codex CLI**         | พร้อมใช้ local    |
| Memory      | **SQLite + Markdown** | simple            |
| Knowledge   | Git                   | versioned         |
| Tools       | Local Python tools    | เริ่มง่าย         |
| Protocol    | MCP-ready             | extensibility     |
| Execution   | Terminal              | garage-native     |
| Container   | Docker later          | isolation         |
| UI          | Terminal              | V0                |
| Git         | GitHub                | collaboration     |
| DB          | SQLite                | no infrastructure |
| Deployment  | Local                 | V0                |

LangGraph มี SQLite checkpointer integration โดยตรงสำหรับ local experimentation และ Postgres สามารถย้ายมาใช้เมื่อเข้าสู่ production ได้ภายหลัง ([Docs by LangChain][8])

---

# 16. สิ่งที่เราจะยังไม่ทำ

ผมขอ “ห้ามตัวเอง” ไว้เลย:

```text
❌ Multi-agent swarm
❌ Redis
❌ Kubernetes
❌ Vector DB
❌ Web UI
❌ Slack
❌ LINE
❌ Voice
❌ Own model
❌ Fine-tuning
❌ Cloud deployment
❌ Autonomous production deployment
❌ Self-modifying source code
```

เพราะตอนนี้:

> **We are in the garage.**

---

# 17. Phase 0 — วันนี้

เราจะมี 5 งานเท่านั้น

### Step 1

สร้าง repo:

```text
anant-hunt
```

### Step 2

สร้าง Python environment

ผมแนะนำ `uv` เพราะเร็วและเหมาะกับ Python project ปัจจุบัน

### Step 3

ติดตั้ง:

```text
langgraph
langgraph-checkpoint-sqlite
```

### Step 4

สร้าง Hunt ที่:

```text
$ hunt
```

สามารถรับข้อความ

### Step 5

ทำ **Understand → Respond**

ยังไม่ build code

---

# 18. Phase 1 — First Agent Loop

ต่อด้วย:

```text
Understand
   ↓
Research
   ↓
Plan
   ↓
Ask approval
   ↓
Build
   ↓
Test
   ↓
Report
```

---

# 19. Phase 2 — Coding Hand

เสียบ Codex:

```text
Hunt
 ↓
CodingHand
 ↓
Codex CLI
 ↓
workspace
```

Codex CLI สามารถติดตั้ง local ได้ทั้ง installer, npm หรือ Homebrew ตาม platform ที่ใช้ ([GitHub][3])

---

# 20. Phase 3 — Memory

เพิ่ม:

```text
MEMORY.md
ADR
Lessons
Research
Session history
```

---

# 21. Phase 4 — Engineering Skills

เริ่มด้วย 5 skills:

```text
requirements
research
architecture
implementation
review
```

แล้วค่อยเพิ่ม:

```text
security
debugging
testing
technical-debt
experimentation
```

---

# 22. Phase 5 — Hunt Learning

หลังจาก Hunt ทำงานได้จริง:

```text
Experience
 ↓
Evaluation
 ↓
Lesson
 ↓
Skill update
 ↓
Regression test
```

นี่คือจุดที่เราจะเริ่มเอา **Hermes philosophy** เข้ามาเต็มตัว

Hermes เองก็แยก skills เป็น procedural knowledge และ memory เป็น factual knowledge และมีระบบให้ agent สร้าง/แก้ skill ได้เอง ([GitHub][4])

---

# 23. แล้วเราจะไม่ “สร้าง Hunt จากศูนย์”

ผมอยากให้จำ architecture นี้ไว้:

```text
             ┌──────────────────────┐
             │       HUNT           │
             │                      │
             │  Judgment            │
             │  Memory              │
             │  Skills              │
             │  Research            │
             │  Alignment           │
             │  Self-improvement    │
             └──────────┬───────────┘
                        │
                 ┌──────▼───────┐
                 │ LangGraph    │
                 │ State/Loop   │
                 └──────┬───────┘
                        │
                 ┌──────▼───────┐
                 │ Coding Hand  │
                 │ Codex / ...  │
                 └──────┬───────┘
                        │
                 ┌──────▼───────┐
                 │ MCP / Tools  │
                 └──────────────┘
```

**เราสร้างเฉพาะส่วนที่ทำให้ Hunt เป็น Hunt**

ส่วนที่เป็น commodity เราประกอบเอา

---

## และนี่คือจุดที่ผมอยากให้เราเริ่มลงมือจริง

**เราไม่ควรออกแบบ architecture เพิ่มอีก 3 วัน**

พอแล้ว

ขั้นถัดไปคือ **สร้าง repo และให้ Hunt พูดกับเราได้**

ผมเสนอให้ milestone แรกมีชื่อว่า:

> # HUNT-001 — First Breath
>
> _Hunt can start. Hunt can listen. Hunt can think. Hunt can answer._

หลังจากนั้น milestone:

```text
HUNT-002  → First Tool
HUNT-003  → First Research
HUNT-004  → First Plan
HUNT-005  → First Build
HUNT-006  → First Test
HUNT-007  → First Memory
HUNT-008  → First Skill
HUNT-009  → First Self-Improvement
HUNT-010  → Apple I
```

**Apple I ของเราคือ HUNT-010:** ให้ Hunt รับโจทย์จาก Mos แล้วสร้าง software เล็ก ๆ จาก empty directory จนรันได้จริง โดย Mos ไม่ต้องเขียน code ให้มัน

ถ้าจะทำต่อแบบจริงจัง ขั้นต่อไปของผมคือ **HUNT-001** — ผมจะสร้าง skeleton ของ repository, `pyproject.toml`, LangGraph state/graph, terminal CLI, constitution และ test suite ชุดแรกให้พร้อมรันบนเครื่องคุณ แล้วเราค่อยเสียบ Coding Hand เข้าไปใน HUNT-005.

[1]: https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/architecture.md?utm_source=chatgpt.com 'hermes-agent/website/docs/developer-guide/architecture.md at main · NousResearch/hermes-agent · GitHub'
[2]: https://docs.langchain.com/oss/python/langgraph/persistence?utm_source=chatgpt.com 'Persistence - Docs by LangChain'
[3]: https://github.com/openai/codex/blob/main/README.md?utm_source=chatgpt.com 'codex/README.md at main · openai/codex · GitHub'
[4]: https://github.com/hermes-agent-org/hermes/blob/main/website/docs/guides/work-with-skills.md?utm_source=chatgpt.com 'hermes/website/docs/guides/work-with-skills.md at main · hermes-agent-org/hermes · GitHub'
[5]: https://github.com/NousResearch/hermes-agent/issues/49764?utm_source=chatgpt.com '[Bug]: Hermes Agent fails to follow rules in memory and skills · Issue #49764 · NousResearch/hermes-agent · GitHub'
[6]: https://docs.langchain.com/oss/python/langgraph/interrupts?utm_source=chatgpt.com 'Interrupts - Docs by LangChain'
[7]: https://py.sdk.modelcontextprotocol.io/?utm_source=chatgpt.com 'MCP Python SDK - MCP Python SDK'
[8]: https://docs.langchain.com/oss/python/integrations/checkpointers/index?utm_source=chatgpt.com 'Checkpointer integrations - Docs by LangChain'

---
