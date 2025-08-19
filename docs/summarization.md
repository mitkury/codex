# Codex Summarization System

## Overview

Codex includes a sophisticated summarization system that allows users to compress long conversations while preserving essential context. This document explains how the summarization process works, the prompts used, and how to effectively use the `/compact` command.

## How Summarization Works

### Triggering Summarization

Summarization is triggered by the `/compact` command in the Codex TUI. When executed:

1. **Creates a Special Task**: Codex creates a dedicated `AgentTask::compact` task
2. **Injects Summarization Instructions**: The system injects a special prompt designed for summarization
3. **Sends Full History**: The entire conversation history is sent to the model
4. **Generates Structured Summary**: The model produces a comprehensive summary
5. **Replaces History**: The conversation history is replaced with just the summary

### The Summarization Process

```rust
// When /compact is triggered
Op::Compact => {
    // Create a summarization request as user input
    const SUMMARIZATION_PROMPT: &str = include_str!("prompt_for_compact_command.md");

    // Attempt to inject input into current task
    if let Err(items) = sess.inject_input(vec![InputItem::Text {
        text: "Start Summarization".to_string(),
    }]) {
        let task = AgentTask::compact(
            sess.clone(),
            Arc::clone(&turn_context),
            sub.id,
            items,
            SUMMARIZATION_PROMPT.to_string(),
        );
        sess.set_task(task);
    }
}
```

## The Summarization Prompt

The core of Codex's summarization system is the structured prompt that guides the model to create comprehensive summaries.

### Full Summarization Prompt

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

### Prompt Analysis

The summarization prompt is designed to:

1. **Define the Role**: Establishes the AI as a "summarization assistant"
2. **Set Clear Scope**: Focuses on coding-focused conversations
3. **Specify Key Elements**: Lists exactly what should be captured
4. **Provide Structure**: Gives a clear format for the output
5. **Ensure Completeness**: Covers objectives, instructions, actions, entities, and next steps

## Summarization Task Execution

### Task Creation and Execution

```rust
async fn run_compact_task(
    sess: Arc<Session>,
    turn_context: &TurnContext,
    sub_id: String,
    input: Vec<InputItem>,
    compact_instructions: String,
) {
    // Build turn input with full conversation history
    let turn_input: Vec<ResponseItem> =
        sess.turn_input_with_history(vec![initial_input_for_turn.clone().into()]);

    // Create prompt with summarization instructions
    let prompt = Prompt {
        input: turn_input,
        store: !turn_context.disable_response_storage,
        tools: Vec::new(), // No tools needed for summarization
        base_instructions_override: Some(compact_instructions.clone()),
    };

    // Execute the summarization
    let attempt_result = drain_to_completed(&sess, turn_context, &sub_id, &prompt).await;
    
    // After completion, replace history with just the summary
    let mut state = sess.state.lock_unchecked();
    state.history.keep_last_messages(1);
}
```

### Key Characteristics

1. **No Tools Available**: Summarization runs without access to tools or file system
2. **Full Context**: The entire conversation history is provided
3. **Override Instructions**: The base system prompt is replaced with summarization instructions
4. **Single Turn**: Summarization completes in one turn
5. **History Replacement**: After completion, only the summary remains

## What Gets Summarized

### Input to Summarization

The summarization process receives the complete conversation history, including:

- **User Messages**: All user inputs and requests
- **Assistant Messages**: All AI responses and explanations
- **Reasoning**: All thinking and decision-making processes
- **Actions**: All tool calls, function executions, and shell commands
- **Results**: All outputs from tools and commands
- **Context**: File contents, search results, and other contextual information

### Output from Summarization

The summarization produces a structured summary containing:

#### **Objective**
- The high-level goal or problem being solved
- The main purpose of the conversation

#### **User Instructions**
- Key requirements and specifications
- Design decisions and preferences
- Important constraints or limitations

#### **AI Actions / Code Behavior**
- Major code changes and modifications
- File creations, updates, and deletions
- Tool usage and command executions
- Testing and validation steps

#### **Important Entities**
- Function names, variables, and classes
- File paths and module names
- Configuration settings and parameters
- Dependencies and imports

#### **Open Issues / Next Steps**
- Unresolved questions or problems
- Remaining tasks or improvements
- Known limitations or TODO items
- Suggested next actions

#### **Summary (Concise)**
- One or two sentence overview
- Captures the essence of the conversation

## Example Summarization Output

Here's an example of what a Codex summarization might look like:

```markdown
**Objective:** Create a new React component for user authentication with form validation and error handling.

**User instructions:** 
• Build a login form with email and password fields
• Include client-side validation for required fields
• Add error handling for failed authentication attempts
• Use modern React hooks and functional components
• Follow the existing project's styling conventions

**AI actions / code behavior:**
• Created `LoginForm.jsx` component with useState hooks
• Implemented form validation using custom validation functions
• Added error state management and user feedback
• Integrated with existing authentication service
• Added unit tests for validation logic
• Updated component documentation

**Important entities:**
• `LoginForm.jsx` - Main component file
• `useAuth` hook - Authentication context
• `validateEmail`, `validatePassword` - Validation functions
• `authService.login()` - Authentication service method
• `LoginForm.test.js` - Test file

**Open issues / next steps:**
• Need to implement password strength requirements
• Consider adding "Remember me" functionality
• Integration tests for the complete authentication flow
• Accessibility improvements (ARIA labels, keyboard navigation)

**Summary (concise):** Built a React login form component with validation and error handling, following project conventions and including comprehensive testing.
```

## What Gets Preserved vs. Lost

### ✅ **Preserved After Summarization**

- **High-level Understanding**: The overall goal and context
- **Key Decisions**: Important design choices and requirements
- **Major Changes**: Significant code modifications and file changes
- **Important Entities**: Function names, file paths, and key variables
- **Next Steps**: Remaining tasks and open issues

### ❌ **Lost After Summarization**

- **Detailed Reasoning**: Step-by-step thinking and decision processes
- **Specific Actions**: Individual tool calls and their exact outputs
- **Code Details**: Specific implementation details and line-by-line changes
- **Context Information**: File contents, search results, and intermediate data
- **Error Details**: Specific error messages and debugging information
- **Timing Information**: When actions occurred and their sequence

## Best Practices for Effective Summarization

### **When to Summarize**

1. **At Natural Break Points**: Between major features or tasks
2. **Before Context Limits**: When approaching token limits (80%+ usage)
3. **After Milestones**: When completing significant work
4. **Before Long Sessions**: At the start of new major tasks

### **Preparing for Summarization**

1. **Complete Current Tasks**: Finish any in-progress work
2. **Resolve Open Issues**: Address any immediate problems
3. **Document Key Decisions**: Ensure important choices are clear
4. **Clean Up Context**: Remove any irrelevant information

### **After Summarization**

1. **Review the Summary**: Ensure it captures the essential information
2. **Note Important Details**: Remember any critical specifics you might need
3. **Plan Next Steps**: Use the summary to guide future work
4. **Be Ready to Re-establish Context**: Some details may need to be re-provided

## Technical Implementation Details

### **Prompt Override Mechanism**

The summarization uses a prompt override to replace the normal system instructions:

```rust
let prompt = Prompt {
    input: turn_input,
    store: !turn_context.disable_response_storage,
    tools: Vec::new(), // No tools for summarization
    base_instructions_override: Some(compact_instructions.clone()),
};
```

### **History Replacement**

After summarization, the conversation history is replaced with just the summary:

```rust
state.history.keep_last_messages(1);
```

This keeps only the last message (the summary) and discards all previous conversation history.

### **Error Handling**

The summarization process includes retry logic for reliability:

```rust
let max_retries = turn_context.client.get_provider().stream_max_retries();
let mut retries = 0;

loop {
    let attempt_result = drain_to_completed(&sess, turn_context, &sub_id, &prompt).await;
    
    match attempt_result {
        Ok(()) => break,
        Err(CodexErr::Interrupted) => return,
        Err(e) => {
            if retries < max_retries {
                retries += 1;
                // Retry with exponential backoff
                continue;
            } else {
                // Report error and return
                return;
            }
        }
    }
}
```

## Limitations and Considerations

### **Context Loss**

- **Inevitable Detail Loss**: Summarization inherently loses specific details
- **Reasoning Loss**: The AI's step-by-step thinking is not preserved
- **Action History**: Specific tool calls and their outputs are lost
- **Code Context**: File contents and search results are not preserved

### **Summary Quality**

- **Model Dependent**: Quality depends on the underlying model's summarization capabilities
- **Structure Dependent**: The structured format helps but may not capture all nuances
- **Length Dependent**: Very long conversations may be harder to summarize effectively

### **No Incremental Summarization**

- **All-or-Nothing**: The entire conversation is summarized at once
- **No Selective Preservation**: Cannot choose what to keep vs. summarize
- **No Hierarchical Summaries**: No multi-level summaries with different detail levels

## Future Enhancements

Potential improvements to the summarization system:

1. **Incremental Summarization**: Summarize portions of conversations selectively
2. **Hierarchical Summaries**: Create multi-level summaries with different detail levels
3. **Semantic Compression**: Use AI to preserve semantic meaning while reducing tokens
4. **Context Chunking**: Break long contexts into manageable, related chunks
5. **Automatic Summarization**: Trigger summarization automatically based on context size
6. **Summary Quality Metrics**: Evaluate and improve summary quality
7. **Custom Summarization Prompts**: Allow users to customize summarization focus
8. **Summary Versioning**: Track different versions of summaries over time