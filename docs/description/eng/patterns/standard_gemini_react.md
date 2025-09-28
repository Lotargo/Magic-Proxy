# Technical Documentation: The `standard_gemini_react` Pattern

## 1. Overview and Purpose

`standard_gemini_react.py` is a **baseline cognitive architecture** for the `react_driver`, optimized for universal application with models like Gemini Pro and similar ones.

This pattern implements a classic, "clean" **Think -> Action -> Observation** cycle, without the additional methodological complexities found in `basic_react` (persona emulation) or `deepseek_react` (forced planning).

**Main Purpose:**
*   **Universality**: To be flexible enough to solve most standard tasks requiring tool use, without excessive specialization.
*   **Reliability**: The simplicity and clarity of the instructions reduce the likelihood that the model will get "confused" or misinterpret its task.
*   **Baseline for Comparison**: To serve as a "control group" when developing new, more complex cognitive architectures. By comparing the results of a new pattern with this baseline, one can objectively assess whether the complications have brought real benefits.

## 2. Philosophy: Pure ReAct

Unlike specialized patterns, this template does not try to impose a complex "philosophy" of thinking on the model. Instead, it clearly and concisely teaches it the fundamental rules of the ReAct process. It is a "safe" and universal default choice.

## 3. Implementation in the Prompt

The pattern consists of clearly separated logical blocks that sequentially train the model.

### 3.1. Basic Instructions and Rules

*   **`SYSTEM_INSTRUCTION`**: Directly informs the model of its role ("a large language model enabled with tools") and its main directive ("follow the 'Think, Action, Observation' cycle").
*   **`SYSTEM_RULE`**: Contains a set of unambiguous, atomic rules regarding the response language and the syntax of the `<THOUGHT>` and `<ACTION>` tags.

### 3.2. Few-Shot Example

This is the core of the training. The example used is a multi-part query that requires **two consecutive tool calls** for a complete answer. This teaches the model key aspects of iterative thinking.

**Lesson 1: Decomposition and the First Step**
*   **Task**: The model learns to decompose a complex query into logical parts (e.g., "question 1: definition," "question 2: components").
*   **Action**: It performs the first logical step—searching for information on the first part of the query.

**Lesson 2: Iterative Refinement**
*   **Task**: This is a key lesson. After receiving the first result, the model analyzes its completeness (`"I have a clear definition now... but I should find a more complete list..."`).
*   **Action**: It recognizes the need for further action and performs a second, clarifying search to answer the second part of the query. This teaches the model **iterative and methodical information gathering**.

**Lesson 3: Completion of Data Gathering**
*   **Task**: The model learns to explicitly state that all necessary information has been fully gathered (`"I have successfully gathered information on both... I have enough information..."`).
*   **Action**: Only after this does it proceed to generate the final answer. This prevents "premature" or incomplete responses.

**Lesson 4: Synthesis**
*   **Task**: The model sees how two pieces of information, obtained from different tool calls, are synthesized into a single, structured, and comprehensive final answer.

## 4. Practical Application and Value

*   **Reliable "Workhorse"**: This pattern is well-suited for most tasks that require a simple "think -> find -> answer" logic.
*   **Excellent Starting Point**: When creating a new, custom ReAct pattern, `standard_gemini_react.py` is an ideal starting point. Its structure is logical, understandable, and easily modifiable.
*   **Diagnostics and Debugging**: If a more complex pattern behaves unpredictably, switching to `standard_gemini_react` can help localize the problem. If the model works correctly with this simple pattern, the problem is likely in the complexity of the custom prompt itself, not in the basic logic of the system.