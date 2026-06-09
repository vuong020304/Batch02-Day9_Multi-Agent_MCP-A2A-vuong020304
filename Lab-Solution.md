# Lab Solution - Day 09: Multi-Agent MCP & A2A

## Stage 1: Direct LLM Calling

**Bài Tập 1.1** — Đổi biến `QUESTION` trong `stages/stage_1_direct_llm/main.py:22`:

```python
QUESTION = "Luật nghĩa vụ quân sự cần những thủ tục gì ?"
```

**Bài Tập 1.2** — Thêm `temperature=0.3` vào `common/llm.py:14-18`:

```python
return ChatOpenAI(
    model=os.getenv("OPENROUTER_MODEL", "anthropic/claude-sonnet-4-5"),
    openai_api_key=os.getenv("OPENROUTER_API_KEY"),
    openai_api_base="https://openrouter.ai/api/v1",
    temperature=0.3,
)
```

Run:
```bash
python stages/stage_1_direct_llm/main.py
```

---

## Stage 2: LLM + RAG & Tools

**Bài Tập 2.1** — Thêm entry `labor_law` vào `LEGAL_KNOWLEDGE` trong `stages/stage_2_rag_tools/main.py`:

```python
{
    "id": "labor_law",
    "keywords": ["lao động", "sa thải", "hợp đồng lao động", "labor", "termination"],
    "text": (
        "Theo Bộ luật Lao động Việt Nam 2019, người sử dụng lao động có thể "
        "đơn phương chấm dứt hợp đồng trong các trường hợp: (1) người lao động "
        "thường xuyên không hoàn thành công việc; (2) bị ốm đau, tai nạn đã điều trị "
        "12 tháng chưa khỏi; (3) thiên tai, hỏa hoạn; (4) người lao động đủ tuổi nghỉ hưu."
    ),
}
```

**Bài Tập 2.2** — Tạo tool `check_statute_of_limitations`:

```python
@tool
def check_statute_of_limitations(case_type: str) -> str:
    """Kiểm tra thời hiệu khởi kiện theo loại vụ án."""
    limits = {
        "contract": "4 năm (UCC § 2-725)",
        "tort": "2-3 năm tùy bang",
        "property": "5 năm",
        "labor": "1 năm đối với một số tranh chấp lao động cá nhân theo Bộ luật Lao động Việt Nam",
    }
    return limits.get(case_type.lower(), "Không xác định")
```

Thêm vào `TOOLS` list: `TOOLS = [search_legal_database, calculate_damages, check_statute_of_limitations]`

Run:
```bash
python stages/stage_2_rag_tools/main.py
```

---

## Stage 3: Single Agent (ReAct)

**Bài Tập 3.1** — Thêm tool `search_case_law` vào `stages/stage_3_single_agent/main.py`:

```python
@tool
def search_case_law(keywords: str) -> str:
    """Tìm kiếm án lệ theo từ khóa."""
    cases = {
        "breach": "Hadley v. Baxendale (1854) - Consequential damages",
        "negligence": "Donoghue v. Stevenson (1932) - Duty of care",
        "contract": "Carlill v. Carbolic Smoke Ball Co (1893) - Unilateral contract",
    }
    for key, case in cases.items():
        if key in keywords.lower():
            return case
    return "Không tìm thấy án lệ phù hợp"
```

Thêm vào `TOOLS`: `TOOLS = [search_legal_database, calculate_penalty, check_compliance_requirements, search_case_law]`

**Bài Tập 3.2** — Bật debug bằng `langchain_core.globals.set_debug(True)` (thay vì `verbose=True` bị deprecated):

```python
from langchain_core.globals import set_debug
set_debug(True)
graph = create_react_agent(model=llm, tools=TOOLS, prompt=SYSTEM_PROMPT)
```

Run:
```bash
python stages/stage_3_single_agent/main.py
```

---

## Stage 4: Multi-Agent (In-Process)

**Bài Tập 4.1** — Thêm Privacy Agent vào system:
- Thêm `needs_privacy` và `privacy_result` vào `LegalState`
- Tạo tool `search_privacy_law` với knowledge về GDPR, CCPA, data breach
- Tạo node `call_privacy_specialist` với ReAct agent
- Cập nhật `aggregate` để include `privacy_result`

**Bài Tập 4.2** — Conditional routing:

```python
if any(kw in question_lower for kw in ["data", "privacy", "gdpr", "dữ liệu"]):
    sends.append(Send("call_privacy_specialist", state))
```

Graph mới:
```
analyze_law -> check_routing -> parallel [tax, compliance, privacy] -> aggregate -> END
```

Run:
```bash
python stages/stage_4_milti_agent/main.py
```

---

## Stage 5: Distributed A2A System

Run 5 services qua các terminal riêng:

```bash
# Terminal 1 (Git Bash) — start all services
./start_all.sh

# Terminal 2 — gửi câu hỏi test
python test_client.py
```

Architecture:
```
Registry (port 10000) — service discovery
         |
    Customer Agent (port 10100) — entry point
         |
    Law Agent (port 10101) — orchestrator
         |
    ┌────┴────┐
 Tax Agent    Compliance Agent
(port 10102)  (port 10103)
```

Kết quả test E2E: thành công, agents giao tiếp qua A2A protocol với dynamic discovery.

---

## Exercises

### Exercise 2: Tools & Knowledge Base

**File:** `exercises/exercise_2_tools.py`

Các TODO đã implement:
1. Thêm `labor_law` entry vào `LEGAL_KNOWLEDGE`
2. Tạo tool `check_statute_of_limitations` với `@tool` decorator
3. Bind tool vào `tools = [search_legal_knowledge, check_statute_of_limitations]`
4. Xử lý tool call `elif tool_call["name"] == "check_statute_of_limitations":`
5. Fix stdout UTF-8 encoding cho Windows

```bash
python exercises/exercise_2_tools.py
```

### Exercise 4: Multi-Agent with Privacy Agent

**File:** `exercises/exercise_4_multiagent.py`

Các TODO đã implement:
1. Thêm `privacy_analysis` vào `State` TypedDict
2. Implement `privacy_agent()` function với domain-specific prompt
3. Thêm routing keywords: "data", "privacy", "gdpr", "dữ liệu", "rò rỉ"
4. Thêm `privacy_agent` node vào graph
5. Thêm edge `privacy_agent → aggregate_results`
6. Cập nhật `aggregate_results` include privacy section
7. Fix graph: dùng `add_conditional_edges("law_agent", check_routing)` thay vì `check_routing` node riêng

```bash
python exercises/exercise_4_multiagent.py
```

---

## Assignment: Supervisor-Workers Pattern

**Mục tiêu:** Improve Agent Day08 bằng pattern Supervisor-Workers với ít nhất 2-3 workers.

### Cấu trúc thư mục `Lab_Assignment/`

```
Lab_Assignment/
├── supervisor.py                  # Supervisor agent — routing + aggregation
├── workers.py                     # 3 workers: Legal, Tax, Compliance
├── graph.py                       # LangGraph StateGraph với parallel Send
├── main.py                        # Entry point chạy hệ thống
└── Day08_RAG_pipeline_cohort2-d3/ # Code RAG pipeline từ Day08
    ├── src/                       # Các task pipelines
    ├── data/                      # Legal & news data
    ├── tests/                     # Unit tests
    └── group_project/             # Chatbot với RAG
```

### Supervisor-Workers Architecture

```
Supervisor → phân tích câu hỏi → route đến workers
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            Legal Worker    Tax Worker    Compliance Worker
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
                            Supervisor → aggregate → final answer
```

### Workers

| Worker | Expertise | Prompt Focus |
|--------|-----------|-------------|
| Legal Worker | General law | Contracts, civil liability, remedies |
| Tax Worker | Tax law | IRS, evasion, penalties, FBAR/FATCA |
| Compliance Worker | Regulatory | SEC, SOX, FCPA, AML, GDPR, CCPA |

### How it works

1. `supervisor_route()`: LLM nhận câu hỏi → quyết định workers cần gọi (JSON: `{"workers": ["legal", "tax", "compliance"]}`)
2. `route_to_workers()`: LangGraph `Send` dispatch workers song song
3. Mỗi worker chạy độc lập với domain-specific prompt
4. `supervisor_aggregate()`: LLM tổng hợp kết quả thành báo cáo cuối cùng

### Run

```bash
python Lab_Assignment/main.py
```

### Yêu cầu checklist đã đạt

| Item | Status |
|------|--------|
| File `Lab-Solution.md` | ✅ Hoàn thành |
| Thư mục `Lab_Assignment/` | ✅ Đã tạo |
| Supervisor-Workers pattern | ✅ 3 workers (legal, tax, compliance) |
| Code Day08 tích hợp | ✅ Trong `Day08_RAG_pipeline_cohort2-d3/` |

---

## Kết luận

Tất cả 5 stages, 2 bài exercises, và assignment Supervisor-Workers đã hoàn thành. Hệ thống multi-agent sử dụng LangGraph và A2A protocol chạy thành công end-to-end.
