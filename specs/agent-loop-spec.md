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
Loop up to MAX_TOOL_ROUNDS times. Stop the loop when the assistant message has no tool calls (when assistant.message.tool_calls is falsy) and return the assistant message as the agent's response.

If the loop exits because MAX_TOOL_ROUNDS was reached, return a fallback message: "Sorry, I couldn't produce an answer. Please try again or provide more detail."
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
Take the assistant message at response.choices[0].message. The user-facing text to return will be assistant.message.content. If content is a string, return as it is. If it is structured, normalize by concatenating the text. If the content is empty, return the fallback message.
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: Tool call: lookup_plant({'plant_name': 'pothos'})
Round 2 tool call: Tool call: get_seasonal_conditions({})
Final response: 

According to the care data for your pothos, it prefers to have the top inch of soil to dry out between waterings. It's also an easy plant to care for and can thrive in low to bright indirect light. The ideal temperature for your pothos is between 65-85°F (18-29°C), and it's tolerant of average household humidity (40-60%).

As we are currently in summer, it's essential to water your pothos more frequently, as the hot weather and air conditioning can dry out the soil faster. You should also continue to fertilize your pothos regularly, as it's the peak growing season. However, be cautious not to over-fertilize, as this can harm your plant.

Additionally, be on the lookout for pests like spider mites and fungus gnats, which can thrive in the hot and dry conditions of summer. Check your plant weekly for any signs of infection, and isolate it immediately if you notice any issues.

By following these care tips, you should be able to keep your pothos happy and healthy throughout the summer season.
```

**What happens when you ask about a plant that isn't in the database?**

```
It returns the fallback message and also provides some general guidance.
```

**One thing about the tool call API that surprised you:**

```

I was surprised that the tool response has to come immediately after the assistant message that requested it, with a matching tool_call_id.
I 
```
