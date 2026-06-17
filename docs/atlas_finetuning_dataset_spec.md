# ATLAS Fine-Tuning Dataset Spec For Bamboo

Goal: prepare supervised data for a Bamboo-hosted ATLAS workflow LLM that
turns tool evidence into better operational answers. The model should learn
how to explain PanDA/ATLAS evidence, not become the source of truth for live
state.

## What The Dataset Should Contain

1. **Answer synthesis examples**
   - User question
   - Optional short conversation history
   - Bamboo tool evidence used to answer
   - Gold answer written in the desired ops style

2. **Tool-routing examples**
   - User question
   - Desired Bamboo tool plan
   - Optional short rationale

3. **Topic-guard examples**
   - In-domain ATLAS/PanDA questions
   - Out-of-domain questions
   - Content-free follow-ups like "tell me more"

4. **Failure-handling examples**
   - Missing IDs
   - Partial evidence
   - Tool errors
   - Conflicting evidence
   - Answers that state uncertainty cleanly

5. **Preference pairs**
   - Same question and evidence
   - Two candidate answers
   - Label which answer is better

## Recommended Record Shape

Use one JSON object per example. Keep the schema simple and stable.

```json
{
  "id": "atlas-task-status-0001",
  "task": "answer_synthesis",
  "question": "Why did task 12345 fail?",
  "history": [
    {"role": "user", "content": "What about the failed jobs?"},
    {"role": "assistant", "content": "Task 12345 had 17 failed jobs..."}
  ],
  "tool_calls": [
    {"name": "panda_task_status", "arguments": {"task_id": 12345}},
    {"name": "panda_log_analysis", "arguments": {"job_id": 67890}}
  ],
  "evidence": {
    "task_status": "...",
    "job_summary": "...",
    "log_summary": "..."
  },
  "target": "Concise expert answer here.",
  "labels": {
    "domain": "atlas",
    "difficulty": "normal",
    "routing": "FAST_PATH"
  }
}
```

## Quality Bar

- Answers should be grounded in the provided evidence.
- If evidence is insufficient, the target should say so explicitly.
- Keep answers in the style operators expect: summary, likely cause, next step.
- Do not encode live facts that will age quickly without the evidence payload.
- Avoid generic assistant prose; reward concise, operational language.

## Suggested Data Mix

- 60-70% answer synthesis
- 10-20% tool-routing
- 10-15% failure-handling
- 5-10% preference pairs

## Evaluation Split

Hold back a small, fixed eval set that is never used for training. Include:

- Common task/job/log triage
- Site and queue questions
- Follow-up questions
- Missing/ambiguous input
- Out-of-domain questions

## Non-Goals

- Do not train the model to query live PanDA directly.
- Do not train on raw answers without the underlying evidence.
- Do not mix ATLAS and other experiment data unless the experiment is labeled.
- Do not treat the model as a replacement for Bamboo tools.

## Practical Next Step

Start with a few hundred curated examples from real Bamboo interactions, then
grow the set with edge cases and preference pairs once the format is stable.
