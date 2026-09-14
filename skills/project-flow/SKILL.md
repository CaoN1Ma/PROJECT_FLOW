---
name: project-flow
description: "Draw a project's execution flow as a Mermaid diagram: read the code, then write docs/project_flow.md — one `flowchart TD` that goes 📥 input → one subgraph per file → 📤 output, where every node's ① line names the real code path (`api/app.py::Router.get_user()`) plus what it does, ②③ carry parameters/state/triggers, every edge states what is passed and why, and loop back-edges and fallback paths are drawn rather than omitted. Closes with a key-notes table and mechanism notes. Also triggers on 画项目流程图 / PROJECT_FLOW / 执行链路 / 项目结构图 / 看懂项目怎么跑起来."
---

Produce `docs/project_flow.md`: a single Mermaid flowchart that traces the project from entry point to output, plus a key-notes table underneath.

## Document skeleton (follow this structure)

````markdown
# Project Flow (PROJECT_FLOW)

> Project:<name> — <one-line architecture, e.g. "LangGraph ReAct loop">
> Note: entry point is `<entry>`; execution passes through the methods/nodes below and ends in the output.

## Overall Flow (Mermaid)

```mermaid
flowchart TD
    %% ==================== Input:<source> ====================
    IN1["📥 <input name><br/>① <protocol/entry> carries <fields><br/>② <meaning of the key fields><br/>③ <meaning of special switches>"]

    %% ==================== <stage name>:<what this layer does> ====================
    subgraph SUB_XXX["<file path>(<role>)"]
        direction TB
        X1["① <file path>::<method()> <what this node does><br/>② <where the inputs come from><br/>③ <what it returns / how state changes / what happens on failure>"]
    end

    %% ==================== Output:<artifact> ====================
    OUT1["📤 <output name><br/>① <response body/artifact> fields<br/>② <meaning of the key fields><br/>③ <how the caller uses it>"]

    %% ==================== Edges (every one carries a concrete explanation) ====================
    IN1 -- "<who sends what to whom, and why>" --> X1
    X1  -. "<resume / dependency / bypass relationship>" .-> X2
```

## Key Notes

| Step | file::Class.method / function | Purpose |
|---|---|---|
| <stage> | `<file path>::<method>()` | <one-line purpose> |

> <Mechanism>:<loop, memory, fallback, thresholds — 2 to 4 lines>
````

When the repo's documentation is in Chinese, use the Chinese headings instead (`# 项目流程图 (PROJECT_FLOW)`, `## 整体流程 (Mermaid 图)`, `## 关键说明`, `> 说明:`).

## Steps

### 1. Find the entry point and the output

Read `README`, `AGENTS.md`, startup scripts, and the `main`/`app`/`cmd` directories to establish: the **entry point** (what the outside world triggers), the **output** (what the response or artifact looks like), and the **external dependencies** (models, databases, third-party APIs). If there is no obvious entry, work backwards from the method nobody calls.

### 2. Layer by execution order, not by directory

The subgraph order is the order a request travels, **not** alphabetical directory order:

```
input → entry (startup/config) → interface (routes/handlers) → orchestration (main flow/state machine)
      → core loop (think ⇄ act) → support (model factory, tool registry, storage) → output
```

Use **one subgraph per file** — closely related small files may share one subgraph, listing them in the title with `, `. Titles carry the **project-relative path plus a role**: `subgraph SUB_APP["api/app.py(interface)"]`. Wherever a loop exists (agent ⇄ tools), the **back-edge must be drawn**.

### 3. Write the nodes

Node ids are **short mnemonics plus a number**: derive them from the role (`IN1`/`OUT1`/`M1`/`APP1`/`MEM1`/`AG1`/`RT1`/`TL1`/`RS1`/`LLM1`…). The boundaries are always `IN1` and `OUT1`.

Label structure (separated by `<br/>`; use 2 to 4 numbered lines, extending to ④⑤ when there is more to say):

```
X1["① <path>::<Class.method()> <what this node does><br/>② <where parameters/inputs come from><br/>③ <what it returns / how state changes / fallback on failure>"]
```

**How to write the location in line ①:**

| Case | Form |
|---|---|
| Module-level function | `api/app.py::calculate()` |
| Method on a class | `agent/memory.py::MilvusCheckpointer.get_tuple()` |
| Module-level statement | `(module level) main.py::load_dotenv()` |
| Graph node / framework node | `agent/graph.py::compress node (agent/summarizer.py) entry` |
| Something from an external library | `langgraph ToolNode execution (mounted in agent/graph.py)` |

**What ②③ carry:** where parameters come from, what is returned, how state changes, what condition triggers this node, how failure degrades, what side effects occur. Use concrete values, thresholds and field names — never "processes data" or "does some work".

**Boundary nodes:** start with `📥` (input) or `📤` (output), carry no method name; ① states the protocol and key fields, ② the field semantics, ③ any special switches.

### 4. Write the edges

Three forms — and **every edge must say what is passed, why, and under what condition**. Long labels are fine:

```text
A -- "main path: who hands what to whom" --> B
A -. "resume / dependency / bypass / compile-time choice" .-> B
A == "data flow: batch or persist" ==> B
```

- Always draw loop back-edges and say so in the label ("back to X to continue (loop)")
- Always draw fallback paths, e.g. "Milvus unavailable → in-memory fallback"
- Inside an edge label, wrap literal strings in 「」 or single quotes — never double quotes, which delimit the label itself

### 5. After the diagram

- **Stage banners**: precede each stage with `%% ==================== <stage name>:<what> ====================`. `%%` comments must sit on their own line.
- **Key Notes table**: `Step | file::Class.method / function | Purpose`. **Every method that appears in the diagram gets a row**; the purpose column says what problem it solves.
- **Mechanism notes**: 2 to 4 `>` lines covering what the diagram cannot show — how the loop terminates, how memory resumes, how history is compressed, what happens when a dependency is down.

### 6. Write the file and self-check

Write `docs/project_flow.md` (create `docs/` if needed), overwriting any previous version.

- [ ] The diagram parses: starts with `flowchart TD`, every `subgraph` has a matching `end`
- [ ] Subgraph order follows execution order; methods inside a file follow call order
- [ ] Every method in the diagram reads `path::method()` and actually exists in the code (never invent one)
- [ ] Every node's ① line has an action; ② onwards carries concrete detail; no filler
- [ ] Every edge has a concrete explanation; loop back-edges and fallback paths are present
- [ ] The Key Notes table covers every method in the diagram; 2 to 4 mechanism notes follow
- [ ] Language matches the repo's documentation language

## Worked example

````markdown
# Project Flow (PROJECT_FLOW)

> Project:Calc Service — FastAPI + multi-turn memory
> Note: entry point is `main`; execution passes through the methods/nodes below and ends in the output.

## Overall Flow (Mermaid)

```mermaid
flowchart TD
    %% ==================== Input:client request ====================
    IN1["📥 Client request<br/>① POST /calc carries JSON:{user, expr, purge}<br/>② user is the memory isolation unit (one conversation per user)<br/>③ purge=true clears that user's memory before starting a new conversation"]

    %% ==================== Entry:process startup (once per process) ====================
    subgraph SUB_MAIN["main.py(entry point)"]
        direction TB
        L1["① (module level) main.py::load_dotenv()<br/>② writes .env into environment variables (once per process)<br/>③ runs before importing business modules so config is ready first"]
        M1["① main.py::main()<br/>② os.getenv reads APP_HOST / APP_PORT / APP_RELOAD<br/>③ calls uvicorn.run('main:app') to start the server"]
    end

    %% ==================== Interface:receive and orchestrate ====================
    subgraph SUB_APP["api/app.py(interface)"]
        direction TB
        A1["① api/app.py::calculate() receives the body<br/>② the Query model validates user/expr/purge<br/>③ maps user onto a thread_id and hands it to the graph"]
        A2["① api/app.py::health() liveness probe<br/>② GET /health, touches neither memory nor model<br/>③ returns {status: ok}"]
    end

    %% ==================== Core:calculation loop ====================
    subgraph SUB_CORE["core/calc.py(expression evaluation)"]
        direction TB
        C1["① core/calc.py::Calculator.evaluate() evaluates<br/>② expr is the expression string to evaluate<br/>③ returns a number or symbolic expression (e.g. 3*x**2); a syntax error becomes ValueError"]
        C2["① core/calc.py::Calculator.history() reads history<br/>② fetches the most recent rows for this thread_id<br/>③ feeds the previous turns back into the next prompt"]
    end

    %% ==================== Output:HTTP response ====================
    OUT1["📤 Final response<br/>① Response JSON:{user, result, reply}<br/>② result = a number or symbolic expression (e.g. 3*x**2)<br/>③ reply = the natural-language answer shown to the user"]

    %% ==================== Edges (every one carries a concrete explanation) ====================
    IN1 -- "an HTTP POST carrying the JSON body {user, expr, purge} reaches the server" --> A1
    IN1 -- "the GET /health liveness probe reaches the same server" --> A2
    L1  -- "loaded before importing api.app so config exists when app is imported" --> A1
    L1  -- "so that os.getenv inside main() can see the .env values" --> M1
    M1  -- "once the server is listening, FastAPI dispatches HTTP requests to it" --> A1
    A1  -- "happy path: hand expr to Calculator.evaluate() for evaluation" --> C1
    A1  -. "when purge=true, read history first to decide what to clear, then start fresh" .-> C2
    C1  -- "on ValueError, record the error on the history side so the next turn can correct it" --> C2
    C2  -- "with the previous context loaded, return to evaluation and recompute (loop)" --> C1
    C1  -- "once evaluated, assemble result and reply into the response" --> OUT1
```

## Key Notes

| Step | file::Class.method / function | Purpose |
|------|-------------------------------|---------|
| Entry | `main.py::main()` | Reads env vars, starts uvicorn, loads the FastAPI app |
| Config | `main.py::load_dotenv()` | Writes .env into env vars (called exactly once) |
| Interface | `api/app.py::calculate()` | Receives user/expr/purge, maps user to thread_id |
| Probe | `api/app.py::health()` | GET /health, returns {status: ok} |
| Evaluate | `core/calc.py::Calculator.evaluate()` | Evaluates an expression, raises ValueError on failure |
| History | `core/calc.py::Calculator.history()` | Fetches recent rows for a thread_id to rebuild the prompt |
| Output | `api/app.py::Response` | Returns JSON:user / result / reply |

> Loop: evaluate → load history → recompute with context, until no expression fails, then respond.
> Memory: one thread_id per user, history isolated per user; `purge: true` clears before recording.
```
````

## Syntax red lines (verified against mermaid 11)

1. **No double quotes inside a node label.** `A["say "hi" now"]` fails to parse; `A["say "hi""]` parses but silently swallows the quotes; backslash-escaping (`\"`) is not supported either. Use 「」 or single quotes.
2. **Angle brackets vanish silently** — in node *and* edge labels. `List<String>` becomes `List`; `<https://example.com>` disappears entirely. Since line ① carries real signatures, write generics as `List&lt;String&gt;` (or full-width `＜ ＞`). The `#lt;` / `#gt;` entity syntax is broken in mermaid 11 and renders as mojibake, like `#quot;`.
3. `\n` does not break a line — use `<br/>`.
4. A `%%` comment cannot share a line with code; it must sit on its own line.
5. Brackets cannot be omitted: `B[load(x)]` fails → **always quote labels**.
6. Do not use `end`, `class`, `graph`, `style`, `subgraph`, `flowchart`, `linkStyle` as node ids (all fail), and do not use file names as ids.
7. Declare a node once, inside its own subgraph — referencing it inside another subgraph silently **moves** it there.
8. Edge labels never wrap (`white-space: nowrap`); break long ones with `<br/>` yourself.

> Optional: if a dark-mode renderer leaves edge text too faint, add this line above `flowchart TD` to pin the edge label foreground/background pair:
> `%%{init: {"theme":"base","themeVariables":{"textColor":"#111111","edgeLabelBackground":"#f2f4f7"}},"themeCSS":".edgeLabel rect{opacity:1 !important;fill:#f2f4f7 !important}.edgeLabel,.edgeLabel span{color:#111111 !important;background-color:#f2f4f7 !important}"}}%%`
> It is deliberately **not** part of the default output.
