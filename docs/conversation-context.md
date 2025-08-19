# Conversation Context and ReACT Chain Management

## Overview

Codex maintains comprehensive conversation history that includes the complete chain of ReACT (Reasoning, Acting) components from each interaction. This document explains how context is accumulated, managed, and compressed to handle long conversations while preserving important information.

## ReACT Chain Components

Codex preserves the full ReACT chain in conversation history:

### **Reasoning/Thinking**
- **Type**: `ResponseItem::Reasoning`
- **Components**:
  - `summary`: High-level reasoning summary (always included)
  - `content`: Detailed reasoning content (optional, can be encrypted)
  - `encrypted_content`: Encrypted reasoning for sensitive operations
- **Purpose**: Captures the AI's thought process and decision-making

### **Actions**
- **Function Calls**: `ResponseItem::FunctionCall` - Tool/function invocations
- **Shell Commands**: `ResponseItem::LocalShellCall` - Terminal command executions
- **Tool Outputs**: `ResponseItem::FunctionCallOutput` - Results from function calls
- **Purpose**: Records all actions taken by the agent

### **Messages**
- **User Messages**: Input from the user
- **Assistant Messages**: Direct responses from the AI
- **Purpose**: Maintains the conversation flow

## Context Accumulation

### How Context is Built

For each turn, Codex constructs the full context by concatenating:

```rust
pub fn turn_input_with_history(&self, extra: Vec<ResponseItem>) -> Vec<ResponseItem> {
    [self.state.lock_unchecked().history.contents(), extra].concat()
}
```

This means:
1. **Complete History**: All previous ReACT chains are included
2. **New Input**: Current turn's input is appended
3. **Full Context**: The model receives the entire conversation history

### Context Growth Pattern

Context grows linearly with each turn:
- **Turn 1**: User input + AI response + reasoning + actions
- **Turn 2**: Turn 1 context + new user input + AI response + reasoning + actions
- **Turn N**: All previous turns + current turn

## Context Window Management

### Token Usage Tracking

Codex tracks token usage with special handling for reasoning:

```rust
pub fn tokens_in_context_window(&self) -> u64 {
    self.total_tokens
        .saturating_sub(self.reasoning_output_tokens.unwrap_or(0))
}
```

**Key Points**:
- Reasoning tokens are excluded from context window calculations
- This prevents reasoning from consuming the entire context budget
- Only user messages, assistant responses, and action results count toward limits

### Model Context Windows

Different models have varying context limits:

| Model | Context Window | Max Output |
|-------|---------------|------------|
| GPT-4 | 96,000 tokens | 32,000 tokens |
| GPT-4o | 200,000 tokens | 100,000 tokens |
| GPT-3.5 | 16,385 tokens | 4,096 tokens |
| Claude | 200,000 tokens | 100,000 tokens |

## Context Compression

### Manual Compression with `/compact`

When context limits are approached, users can manually compress the conversation:

#### **How `/compact` Works**

1. **Triggers Summarization Task**:
   ```rust
   let task = AgentTask::compact(
       sess.clone(),
       Arc::clone(&turn_context),
       sub.id,
       items,
       SUMMARIZATION_PROMPT.to_string(),
   );
   ```

2. **Sends Full History**: The entire conversation history is sent to the model with summarization instructions

3. **Generates Summary**: The model creates a structured summary including:
   - **Objective**: High-level goal being solved
   - **User Instructions**: Key requirements and decisions
   - **AI Actions**: Main code changes and behaviors
   - **Important Entities**: Functions, variables, files discussed
   - **Open Issues**: Unresolved questions or next steps

4. **Replaces History**: The conversation history is replaced with just the summary:
   ```rust
   state.history.keep_last_messages(1);
   ```

#### **Summarization Prompt**

The summarization uses a structured format:

```markdown
You are a summarization assistant. A conversation follows between a user and a coding-focused AI (Codex). Your task is to generate a clear summary capturing:

• High-level objective or problem being solved  
• Key instructions or design decisions given by the user  
• Main code actions or behaviors from the AI  
• Important variables, functions, modules, or outputs discussed  
• Any unresolved questions or next steps

Produce the summary in a structured format like:

**Objective:** …

**User instructions:** … (bulleted)

**AI actions / code behavior:** … (bulleted)

**Important entities:** … (e.g. function names, variables, files)

**Open issues / next steps:** … (if any)

**Summary (concise):** (one or two sentences)
```

### What Gets Preserved vs. Lost

#### **After Compression**:
- ✅ **Summary**: High-level understanding of the conversation
- ✅ **Current Context**: The summary becomes the new conversation base
- ❌ **Detailed Reasoning**: All step-by-step thinking is lost
- ❌ **Action History**: Specific tool calls and their outputs are lost
- ❌ **Code Changes**: Individual edits and their context are lost

## Context Management Strategies

### **No Automatic Trimming**

Codex does **not** automatically trim context based on token limits. This design choice prioritizes:

1. **Complete Context**: Maximum continuity and understanding
2. **User Control**: Users decide when to compress
3. **No Surprise Loss**: Important information isn't unexpectedly dropped

### **Manual Intervention Required**

Users must proactively manage context through:

1. **Monitoring**: Watch the context percentage indicator in the UI
2. **Timing**: Compress before hitting limits
3. **Strategy**: Compress at natural break points in the conversation

### **Context Percentage Display**

The UI shows context usage:
```
75% context left
```

This helps users know when compression is needed.

## Best Practices

### **When to Use `/compact`**

- **Before Long Sessions**: Compress at the start of new major tasks
- **After Milestones**: Compress after completing significant features
- **When Context > 80%**: Proactive compression to avoid limits
- **At Natural Breaks**: Between different coding tasks or features

### **What to Expect After Compression**

1. **Loss of Detail**: The AI won't remember specific reasoning or actions
2. **Preserved Intent**: High-level goals and context remain
3. **Fresh Start**: The conversation continues with the summary as context
4. **Potential Re-work**: Some context may need to be re-established

### **Optimizing Context Usage**

1. **Concise Messages**: Keep user inputs focused and specific
2. **Batch Operations**: Group related changes in single turns
3. **Regular Compression**: Don't wait until context is nearly full
4. **Strategic Timing**: Compress at logical conversation boundaries

## Technical Implementation

### **Conversation History Structure**

```rust
pub(crate) struct ConversationHistory {
    /// The oldest items are at the beginning of the vector.
    items: Vec<ResponseItem>,
}
```

### **Recording Conversation Items**

```rust
async fn record_conversation_items(&self, items: &[ResponseItem]) {
    debug!("Recording items for conversation: {items:?}");
    self.record_state_snapshot(items).await;
    self.state.lock_unchecked().history.record_items(items);
}
```

### **Message Filtering**

Only certain items are recorded in conversation history:

```rust
fn is_api_message(message: &ResponseItem) -> bool {
    match message {
        ResponseItem::Message { role, .. } => role.as_str() != "system",
        ResponseItem::FunctionCallOutput { .. }
        | ResponseItem::FunctionCall { .. }
        | ResponseItem::LocalShellCall { .. }
        | ResponseItem::Reasoning { .. } => true,
        ResponseItem::Other => false,
    }
}
```

### **Adjacent Message Merging**

Assistant messages are merged to prevent duplicates:

```rust
match (&*item, self.items.last_mut()) {
    (
        ResponseItem::Message {
            role: new_role,
            content: new_content,
            ..
        },
        Some(ResponseItem::Message {
            role: last_role,
            content: last_content,
            ..
        }),
    ) if new_role == "assistant" && last_role == "assistant" => {
        append_text_content(last_content, new_content);
    }
    _ => {
        self.items.push(item.clone());
    }
}
```

## Limitations and Considerations

### **Context Loss Trade-offs**

- **Pros**: Enables very long conversations, prevents context overflow
- **Cons**: Loss of detailed reasoning and action history
- **Mitigation**: Strategic compression timing and clear summaries

### **Reasoning Token Accounting**

- Reasoning tokens are excluded from context window calculations
- This prevents reasoning from consuming the entire context budget
- However, reasoning is still included in the actual context sent to models

### **No Incremental Compression**

- Compression is all-or-nothing (full history → summary)
- No partial compression or selective retention
- Future versions may support more granular compression strategies

## Future Enhancements

Potential improvements to context management:

1. **Incremental Compression**: Selective retention of important details
2. **Automatic Compression**: Smart detection of when to compress
3. **Hierarchical Summaries**: Multi-level summaries with different detail levels
4. **Context Chunking**: Breaking long contexts into manageable chunks
5. **Semantic Compression**: AI-driven compression that preserves semantic meaning