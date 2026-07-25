# 🐶 BugHound

BugHound is a small, agent-style debugging tool. It analyzes a Python code snippet, proposes a fix, and runs basic reliability checks before deciding whether the fix is safe to apply automatically.

---

## What BugHound Does

Given a short Python snippet, BugHound:

1. **Analyzes** the code for potential issues  
   - Uses heuristics in offline mode  
   - Uses Gemini when API access is enabled  

2. **Proposes a fix**  
   - Either heuristic-based or LLM-generated  
   - Attempts minimal, behavior-preserving changes  

3. **Assesses risk**  
   - Scores the fix  
   - Flags high-risk changes  
   - Decides whether the fix should be auto-applied or reviewed by a human  

4. **Shows its work**  
   - Displays detected issues  
   - Shows a diff between original and fixed code  
   - Logs each agent step

---

## Setup

### 1. Create a virtual environment (recommended)

```bash
python -m venv .venv
source .venv/bin/activate   # macOS/Linux
# or
.venv\Scripts\activate      # Windows
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

---

## Running in Offline (Heuristic) Mode

No API key required.

```bash
streamlit run bughound_app.py
```

In the sidebar, select:

* **Model mode:** Heuristic only (no API)

This mode uses simple pattern-based rules and is useful for testing the workflow without network access.

---

## Running with Gemini

### 1. Set up your API key

Copy the example file:

```bash
cp .env.example .env
```

Edit `.env` and add your Gemini API key:

```text
GEMINI_API_KEY=your_real_key_here
```

### 2. Run the app

```bash
streamlit run bughound_app.py
```

In the sidebar, select:

* **Model mode:** Gemini (requires API key)
* Choose a Gemini model and temperature

BugHound will now use Gemini for analysis and fix generation, while still applying local reliability checks.

---

## Instructor Summary

The core concept here is that "agentic" doesn't mean "trust the LLM more" — it means adding explicit workflow steps (analyze, act, test, reflect) so the system can catch and reject its own bad output, whether that output comes from an LLM or a heuristic. Students most often struggle at the boundary between the risk score and the auto-fix decision: they read the score as "how buggy is the original code" when it actually measures "how safe is this specific fix to auto-apply," and those two things diverge whenever the fixer does something unexpected. AI assistance was genuinely helpful for tracing control flow across `bughound_agent.py` and `risk_assessor.py` quickly, but misleading when asked to explain *why* a given score came out a certain way without first being shown the actual fixed_code — it will confidently rationalize a score against the wrong inputs if you let it guess instead of checking the diff. A good way to guide a student stuck here without giving the answer: ask them to compare the "Detected issues" panel against the "Proposed fix" diff side by side and describe, in their own words, whether the fix shown could plausibly justify every reason listed in the risk report — if it can't, the bug is upstream of the risk assessor, not in it.

---

## Running Tests

Tests focus on **reliability logic** and **agent behavior**, not the UI.

```bash
pytest
```

You should see tests covering:

* Risk scoring and guardrails
* Heuristic fallbacks when LLM output is invalid
* End-to-end agent workflow shape
