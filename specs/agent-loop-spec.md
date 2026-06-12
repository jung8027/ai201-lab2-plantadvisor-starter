# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an *agent* rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's current message |
| `history` | `list` | Gradio conversation history — list of `[user_msg, assistant_msg]` pairs |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. Gradio history is a list of `[user, assistant]`
pairs — convert each pair to two API-format dicts:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for user_msg, assistant_msg in history:
    messages.append({"role": "user", "content": user_msg})
    if assistant_msg:
        messages.append({"role": "assistant", "content": assistant_msg})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

*The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case.*

```
(a) No tool calls: After each LLM call, check assistant_message.tool_calls. If it is
    falsy (None or empty list), the LLM has a final answer. Return
    assistant_message.content immediately, or a fallback string if content is None/empty.

(b) MAX_TOOL_ROUNDS reached: The for loop runs at most MAX_TOOL_ROUNDS iterations. If
    every iteration produced tool calls and the loop exhausts all rounds, execution falls
    through to the code below the loop. There, make one final LLM call without the tools
    parameter so the model is forced to produce a text response, then return that content.
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
response.choices[0].message.content

The response has a choices list; index 0 is the assistant turn. Its .message object has
a .content attribute that holds the final text string. Guard against None with a fallback:

    return assistant_message.content or "I'm sorry, I wasn't able to generate a response."
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: lookup_plant({"plant_name": "calathea"})
  → Returns found=True with full calathea care dict (watering, light, humidity, etc.)
Round 2 tool call: get_seasonal_conditions() [no season arg — auto-detects current month]
  → Returns summer seasonal advice with detected_season=True
Final response: Detailed calathea care advice citing the database entry, plus summer-specific
  tips (increase humidity, keep out of direct sun, water consistently).
```

**What happens when you ask about a plant that isn't in the database?**

```
lookup_plant returns {"found": False, "name": <input>, "message": "No plant matching '...'
was found in the database. Do not invent specific care parameters..."}.

The LLM reads the message field and follows its instructions: it tells the user the plant
isn't in its database, offers brief general care guidance based on what the user described,
and recommends consulting a trusted resource (e.g., the AHS or the plant's nursery tag)
for precise watering schedules, light levels, and temperatures.
```

**One thing about the tool call API that surprised you:**

```
The assistant message containing tool_calls must be appended to the messages list before
appending any tool result messages. The API enforces this ordering — a "tool" role message
must immediately follow the assistant message that requested it. Forgetting this step (e.g.,
appending only the tool results) causes an API error, even though conceptually the results
are what you're adding.
```
