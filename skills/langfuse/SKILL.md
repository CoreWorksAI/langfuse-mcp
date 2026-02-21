---
name: langfuse
version: 1.1.0
description: Debug AI traces, find exceptions, analyze sessions, and manage prompts via Langfuse MCP. Multi-environment support (dev/prod/local).
metadata:
  short-description: Langfuse observability via MCP (multi-env)
  compatibility: claude-code, codex-cli
---

# Langfuse Skill

Debug your AI systems through Langfuse observability. Supports multiple environments via `env` parameter.

**Triggers:** langfuse, traces, debug AI, find exceptions, set up langfuse, what went wrong, why is it slow, datasets, evaluation sets

## Multi-Environment

Every tool accepts an optional `env` parameter. Omit to use the default environment.

```
fetch_traces(age=60)                  # uses default env
fetch_traces(age=60, env="prod")      # target prod
fetch_trace(trace_id="...", env="local")  # target local
```

Available environments are configured in `config.json`.

---

## CRITICAL: Context Management Rules

MCP responses can be very large (10k+ tokens). **Always drill down progressively** to avoid filling up context.

### Rule 1: Start with aggregates and small result sets
```
get_error_count(age=1440)                              # just a number
find_exceptions(age=60, group_by="file")               # aggregated counts
fetch_traces(age=60, limit=5)                          # small limit
```

### Rule 2: Fetch traces WITHOUT observations first
```
fetch_trace(trace_id="...")                            # metadata only
get_session_details(session_id="...")                   # overview only
```

### Rule 3: Only add include_observations when you need it
```
fetch_trace(trace_id="...", include_observations=true)  # ONLY for specific traces
fetch_observation(observation_id="...")                  # single observation
```

### Rule 4: Use full_json_file for large responses
```
fetch_trace(trace_id="...", include_observations=true, output_mode="full_json_file")
```
Then use the Read tool on the returned file path to inspect specific parts.

### NEVER do this:
- `fetch_traces(age=1440, include_observations=true)` — fetches everything
- `get_session_details(..., include_observations=true)` as a first call
- `output_mode="full_json_string"` on large traces — puts entire JSON inline

---

## Playbooks

### "Where are the errors?"

```
get_error_count(age=1440)
```
→ Quick count. If non-zero, drill down:

```
find_exceptions(age=1440, group_by="file")
```
→ Error counts by file. Pick the worst offender.

```
find_exceptions_in_file(filepath="src/ai/chat.py", age=1440)
```
→ Lists specific exceptions. Grab a trace_id.

```
get_exception_details(trace_id="...")
```
→ Full stacktrace and context.

---

### "What happened in this interaction?"

```
fetch_traces(age=60, user_id="...", limit=10)
```
→ Find the trace. Note the trace_id.

```
fetch_trace(trace_id="...")
```
→ Get trace metadata first (without observations).

```
fetch_trace(trace_id="...", include_observations=true)
```
→ Only if you need to see all LLM calls in the trace.

```
fetch_observation(observation_id="...")
```
→ Inspect a specific generation's input/output.

---

### "Why is it slow?"

```
fetch_observations(age=60, type="GENERATION", limit=10)
```
→ Find recent LLM calls. Look for high latency.

```
fetch_observation(observation_id="...")
```
→ Check token counts, model, timing.

---

### "What's this user experiencing?"

```
get_user_sessions(user_id="...", age=1440)
```
→ List their sessions.

```
get_session_details(session_id="...")
```
→ See all traces in the session (without observations first).

---

### "Manage datasets"

```
list_datasets()
```
→ See all datasets.

```
get_dataset(name="evaluation-set-v1")
```
→ Get dataset details.

```
list_dataset_items(dataset_name="evaluation-set-v1", page=1, limit=10)
```
→ Browse items in the dataset.

```
create_dataset(name="qa-test-cases", description="QA evaluation set")
```
→ Create a new dataset.

```
create_dataset_item(
  dataset_name="qa-test-cases",
  input={"question": "What is 2+2?"},
  expected_output={"answer": "4"}
)
```
→ Add test cases.

```
create_dataset_item(
  dataset_name="qa-test-cases",
  item_id="item_123",
  input={"question": "What is 3+3?"},
  expected_output={"answer": "6"}
)
```
→ Upsert: updates existing item by id or creates if missing.

---

### "Manage prompts"

```
list_prompts()
```
→ See all prompts with labels.

```
get_prompt(name="...", label="production")
```
→ Fetch current production version.

```
create_text_prompt(name="...", prompt="...", labels=["staging"])
```
→ Create new version in staging.

```
update_prompt_labels(name="...", version=N, labels=["production"])
```
→ Promote to production. (Rollback = re-apply label to older version)

---

## Quick Reference

| Task | Tool |
|------|------|
| List traces | `fetch_traces(age=N)` |
| Get trace details | `fetch_trace(trace_id="...", include_observations=true)` |
| List LLM calls | `fetch_observations(age=N, type="GENERATION")` |
| Get observation | `fetch_observation(observation_id="...")` |
| Error count | `get_error_count(age=N)` |
| Find exceptions | `find_exceptions(age=N, group_by="file")` |
| List sessions | `fetch_sessions(age=N)` |
| User sessions | `get_user_sessions(user_id="...", age=N)` |
| List prompts | `list_prompts()` |
| Get prompt | `get_prompt(name="...", label="production")` |
| List datasets | `list_datasets()` |
| Get dataset | `get_dataset(name="...")` |
| List dataset items | `list_dataset_items(dataset_name="...", limit=N)` |
| Create/update dataset item | `create_dataset_item(dataset_name="...", item_id="...")` |

`age` = minutes to look back (max 10080 = 7 days)

---

## References

- `references/tool-reference.md` — Full parameter docs, filter semantics, response schemas
- `references/setup.md` — Manual setup, troubleshooting, advanced configuration
