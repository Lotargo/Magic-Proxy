# Technical Documentation: The `deepseek_react` Pattern

## 1. Overview and Purpose

`deepseek_react.py` is a **highly structured and methodical cognitive architecture** for the `react_driver`. Unlike `basic_react`, which focuses on persona emulation, this pattern compels the model to act as an **expert AI researcher**.

**Main Goal**: To ensure maximum predictability, structure, and verifiability of the result for complex analytical and research tasks.

This pattern is ideal for generating reports, comparisons, technical explanations, and any other task where logical structure, accuracy, and step-by-step execution are important.

## 2. Philosophy: The Model as a Methodical Research Assistant

Where `basic_react` teaches the model to "feel" and interpret, `deepseek_react` teaches it to "work" like a methodical research assistant. The main idea is to decompose a complex task into manageable steps and prepare the answer in stages.

The pattern forces the model to sequentially play three roles:

1.  **The "Planner" Role**: At the very beginning, the model is required to create a clear, step-by-step action plan.
2.  **The "Executor" Role**: The model iteratively and sequentially executes the plan items, using tools to gather information.
3.  **The "Editor" Role**: Before giving the final answer, the model is required to synthesize all collected information into a structured **DRAFT** directly in its thoughts.

This approach dramatically improves the quality and reliability of answers to complex, factual queries.

## 3. Four-Stage Research Methodology

This is the core of the pattern. The model is trained to follow a strict, imperative four-step process:

1.  **Create a PLAN**
    *   **Task**: Decompose the user's query into a logical sequence of steps.
    *   **Result**: An explicit, numbered plan at the very beginning of the work.
2.  **EXECUTE the plan**
    *   **Task**: Sequentially execute each item of the plan, using tools to gather facts.
    *   **Result**: A set of "Observations" from the tools.
3.  **Create a DRAFT**
    *   **Task**: After gathering all information, synthesize it into a concise but structured draft. This draft is created in the model's "thoughts" (`<THOUGHT>`).
    *   **Result**: A "skeleton" for the final answer, ensuring that all key points are covered in a logical order.
4.  **Format the draft into the FINAL ANSWER**
    *   **Task**: Turn the structured but concise draft into a detailed, well-formatted, and easy-to-read final answer.
    *   **Result**: A high-quality final text.

## 4. Implementation in the Prompt

This process, as in other patterns, is implemented through system instructions and a **Few-Shot Example**.

*   **System Instructions**: Clearly define the role of "expert AI researcher" and imperatively define the four-stage methodology.
*   **Few-Shot Example**: Demonstrates the application of the methodology in practice.

**Key points in the few-shot example:**
1.  **Explicit Plan**: The very first "thought" (`<THOUGHT>`) of the model contains an explicit, numbered `PLAN`. This trains it to decompose the task *before* starting work.
2.  **Iterative Execution**: The model learns to refer to specific plan steps in its thoughts (`Executing Step 1...`, `Executing Step 2...`), which makes its work methodical.
3.  **Structured Draft**: This is the culmination of the training. In the final "thought," the model explicitly announces the transition to step 3, creates a structured `DRAFT`, and then announces the transition to step 4. This reinforces the link between preliminary preparation and a high-quality result.

## 5. Practical Application and Value

*   **Structured and Verifiable Answers**: The presence of an explicit plan and draft in the logs (`scratchpad`) makes it easy to trace and verify the model's logic, which is critically important for debugging.
*   **Performance on Complex Queries**: Decomposing the task helps the model not to get "lost" when processing multi-part queries.
*   **High Consistency**: The rigid structure leads to a more predictable and uniform format for answers.

**When to use this pattern**: Choose `deepseek_react` for tasks requiring an analytical, structured, and factual approach. It is less suitable for creative or conversational tasks, where `basic_react` would perform better.