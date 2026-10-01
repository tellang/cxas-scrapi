# Callback API Reference

The GECX sandbox auto-provides these globals. Do NOT import them. Use ONLY the methods listed here -- do not guess at additional attributes or constructors. The list includes the official `Part` constructors under "Official constructors" below (checked against the CX Agent Studio docs on 2026-10-01).

`typing` types (`Optional`, `Iterator`, `List`, `Dict`, etc.) are NOT auto-provided — they must be explicitly imported. Missing typing imports cause errors at push time.

## Part

**Creating Parts** -- always use factory methods, never raw constructors:

```python
# Text
Part.from_text(text="Hello, how can I help?")

# Tool/function call
Part.from_function_call(name="end_session", args={"session_escalated": True})
Part.from_function_call(name="lookup_benefits", args={"member_id": "H123", "plan_id": "PLN-001"})
```

**Official constructors** (from the CX Agent Studio best-practices page; all are returned via `LlmResponse.from_parts(...)`):

```python
# End the session deterministically (official pattern: "Gracefully terminate the session on tool failures")
Part.from_end_session(reason="Failure during user authentication.")

# Transfer to another agent deterministically (official pattern: "Transfer to another agent on tool failures")
Part.from_agent_transfer(agent="escalation agent")

# Voice only: speak fixed content with barge-in disabled (official pattern: "Disallow user barge-in for certain responses")
Part.from_customized_response(content="Please listen to the following legal disclaimer: ...", disable_barge_in=True)

# Custom JSON payload for the client; not visible to the LLM (official pattern: "Using custom payloads to drive your user interface")
Part.from_json(data=json.dumps(payload))
```

The official telephony page also ends a session from `before_model_callback` on the hang-up event:
`LlmResponse.from_parts([Part.from_function_call('sum', {'x': 1, 'y': 2}), Part.from_end_session(reason='...')])`
when `part.text == "<event>sys.remote-call-disconnected</event>"`.

**Reading Parts:**

```python
part.text                        # str | None -- text content (text mode only)
part.text_or_transcript()        # str | None -- text OR audio transcript (use this for audio-safe detection)
part.has_function_call("end_session")  # bool -- check if this part is a specific function call
part.function_call               # object | None -- the function call (has .name, .args)
part.function_response           # object | None -- the function response
```

**DO NOT USE:**
- `Part(text=...)` -- prefer `Part.from_text()`. Note: the official callback page examples do use `Part(text=...)` and `LlmResponse(content=Content(parts=[...]))`, so these are a cxas-scrapi convention, not a platform restriction.
- `part.custom_metadata` -- does not exist
- `part.inline_data` -- internal, use `text_or_transcript()` instead

## Content

```python
content.parts    # list[Part] -- the parts in this content
content.role     # str -- "model" or "user"
```

**DO NOT USE:**
- `Content(parts=[...])` -- use `LlmResponse.from_parts()` to build responses

## LlmResponse

**Creating responses** (in before_model callbacks):

```python
# 1. Deterministic Preemption (Bypasses LLM completely, speaks text, waits for user):
LlmResponse.from_parts(parts=[
    Part.from_text(text="What is your account number?"),
])

# 2. Tool Delegation with Audio Filler (Runs tool, feeds toolResponse to LLM, triggers LLM reasoning):
LlmResponse.from_parts(parts=[
    Part.from_text(text="Let me look that up for you.", partial=True),
    Part.from_function_call(name="lookup_account", args={"id": "123"}),
])
```

**LLM Lifecycle Decision Matrix:**
- **Text-Only Parts (`Part.from_text`)**: CXAS sends speech to user and closes bot turn. **LLM is bypassed.**
- **FunctionCall Parts (`Part.from_function_call`)**: CXAS executes tool and feeds `toolResponse` back to LLM. **LLM is invoked.**
- **Gotcha (`EMPTY_RESPONSE`)**: If you need to speak a prompt deterministically and wait for user speech, do NOT return a `FunctionCall` part in the same response. Execute the tool synchronously via `tools.<tool_name>()` in Python and return only `Part.from_text(...)`.

**Reading responses** (in after_model callbacks):

```python
llm_response.content        # Content -- the response content
llm_response.content.parts  # list[Part] -- iterate over parts
```

**DO NOT USE:**
- `LlmResponse(content=Content(parts=[...]))` -- use `LlmResponse.from_parts()` instead

## LlmRequest

**Reading requests** (in before_model callbacks):

```python
llm_request.contents    # list[Content] -- full conversation history
llm_request.contents[-1]  # Content -- last message (usually user input)
```

## CallbackContext

```python
callback_context.state              # dict -- session state (read/write)
callback_context.events             # list -- full session event history
callback_context.get_last_user_input()  # list[Part] -- parts from last user message
```

**Events** (for walking session history):

```python
for event in reversed(callback_context.events):
    event.is_user()    # bool -- is this a user event?
    event.is_agent()   # bool -- is this an agent event?
    event.parts()      # list[Part] -- parts in this event
```

## Common Patterns

### Detect session start

```python
for part in callback_context.get_last_user_input():
    if part.text == "<event>session start</event>":
        # First model call -- return greeting
```

### Detect silence / no-input

```python
import re

def is_user_inactive(contents: list) -> bool:
    pattern = r"<context>no user activity detected for \d+ seconds\.</context>"
    return len(contents) > 1 and any(
        re.search(pattern, p.text, re.IGNORECASE)
        for p in contents[-1].parts
        if p.text
    )
```

### Check if response has text (audio-safe)

```python
has_text = False
for part in llm_response.content.parts:
    content = part.text_or_transcript()
    if content and len(content.strip()) > 0:
        has_text = True
```

### Check for end_session in response

```python
has_end_session = any(
    part.has_function_call("end_session")
    for part in llm_response.content.parts
)
```

## Session State

- cxas-scrapi convention: store values as **strings**. This is NOT a platform limit. The official variables page lists Text, Number, Yes/No (boolean), Custom Object and List types, and handoff rules can only check text, number and boolean variables. Keep a variable numeric if a handoff rule must compare it (e.g. an age cut-off). Unverified: whether `callback_context.state` preserves non-string values at runtime has not been measured, so the string convention below stays the safe default inside callbacks.
- Booleans: use `"true"` / `"false"`, check with `str(state.get("flag", "false")).lower() == "true"`
- Counters: use `str(int(state.get("counter", "0")) + 1)`
- Access via `callback_context.state` in callbacks, `context.state` in tools
