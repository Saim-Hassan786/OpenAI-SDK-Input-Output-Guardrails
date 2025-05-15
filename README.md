# OpenAI Agents SDK: Input/Output Guardrails

## Overview  
Input and Output Guardrails in the OpenAI Agents SDK are lightweight, parallel checks that validate user inputs or model outputs against custom rules before and after the core agent logic, enabling safe, compliant, and cost-efficient AI agent behavior

## Concepts

### Input Guardrails  
- **Function**: Validate or transform incoming user requests before they reach the agent’s main model call   
- **Flow**:  
  1. Receive the same user input as the agent.  
  2. Run checks (e.g., content filtering, schema validation) in parallel.  
  3. Trigger a tripwire and halt execution if any rule fails 

### Output Guardrails  
- **Function**: Examine the agent’s response to ensure compliance with domain rules or policies before returning to the user 
- **Flow**:  
  1. Capture the agent’s generated output.  
  2. Apply validation rules (e.g., format enforcement, policy checks).  
  3. Trigger a tripwire and raise an exception if validation fails  

### Tripwires  
- A **tripwire** is a boolean flag in a guardrail result indicating a violation.  
- When `tripwire_triggered == true`, execution halts immediately, raising either `InputGuardrailTripwireTriggered` or `OutputGuardrailTripwireTriggered`

## Benefits  
- **Safety & Compliance**: Decouple policy enforcement from model prompts, making it easier to update rules without retraining  
- **Cost Efficiency**: Early rejection of malicious or irrelevant inputs prevents unnecessary calls to expensive LLMs .
- **Maintainability**: Modular guardrail functions can be tested, logged, and maintained separately from core agent logic .

## Challenges  
- **False Positives**: Overly strict rules may block valid interactions, degrading user experience .
- **Latency Overhead**: Additional parallel checks introduce compute overhead; guardrail logic must be optimized for performance.  
- **Complexity Management**: As the number of rules grows, interdependencies can become harder to debug and maintain .

## Best Practices  
- **Modular Rules**: Define each guardrail function for a single concern (e.g., profanity filter, schema validation) to improve clarity and reuse .  
- **Fail Fast**: Place cheap Input Guardrails early in the flow to avoid wasted computation on invalid requests.  
- **Observability**: Log tripwire activations and guardrail metrics to refine rules over time and detect edge cases in production.  
- **Extensive Testing**: Use adversarial and edge-case inputs in unit tests to minimize false positives and negatives .

## Summary  
This document provides a concise theoretical overview of **Input/Output Guardrails** in the OpenAI Agents SDK. It covers their purpose, execution flow, implementation patterns, and key trade-offs when deploying guardrails to ensure safe, compliant, and cost-effective AI agent behavior.

## 1. What Are Guardrails?  
- **Guardrails** are programmable checks that run before and/or after an agent invokes a language model, allowing developers to intercept, validate, or block unsafe or unwanted interactions. 
- There are two main types:  
  1. **Input Guardrails** – examine and (optionally) block or transform user inputs before hitting the model.  
  2. **Output Guardrails** – validate or filter model responses, raising exceptions on rule violations. 

## 2. Why Use Guardrails?  
- **Safety & Compliance**: Prevent models from producing disallowed content (e.g., hate speech, PII leaks). 
- **Cost-Control**: Early-detect malicious or irrelevant inputs to avoid unnecessary API calls. 
- **Business Logic Enforcement**: Ensure outputs conform to domain-specific schemas or policies.

## 3. Theoretical Execution Flow  
### 3.1 Input Guardrails  
1. **Receive user input** in the agent runner.  
2. **Run guardrail function** (`@input_guardrail`) producing a `GuardrailFunctionOutput`.  
3. **Check tripwire**: if `tripwire_triggered == true`, raise `InputGuardrailTripwireTriggered` to halt execution. 

### 3.2 Output Guardrails  
1. **Receive agent output** after model invocation.  
2. **Run guardrail function** (`@output_guardrail`) producing a `GuardrailFunctionOutput`.  
3. **Check tripwire**: if triggered, raise `OutputGuardrailTripwireTriggered` before returning to user. 

## 4. Implementing Guardrails (Pseudo-Code)  
```python
from agents import (
    Agent, Runner,
    input_guardrail, output_guardrail,
    GuardrailFunctionOutput,
    InputGuardrailTripwireTriggered,
    OutputGuardrailTripwireTriggered
)

# 1. Define a guardrail agent or logic
@input_guardrail
async def profanity_filter(ctx, agent, user_input):
    # Detect prohibited words (theoretical)
    is_bad = any(word in user_input.lower() for word in ["badword"])
    return GuardrailFunctionOutput(
        output_info={"flagged": is_bad},
        tripwire_triggered=is_bad
    )

@output_guardrail
async def schema_validator(ctx, agent, agent_output):
    # Ensure JSON schema compliance
    valid = validate_schema(agent_output)
    return GuardrailFunctionOutput(
        output_info={"schema_ok": valid},
        tripwire_triggered=not valid
    )

# 2. Attach guardrails to your agent
my_agent = Agent(
    name="MySafeAgent",
    instructions="Provide product recommendations in JSON.",
    guardrails=[profanity_filter, schema_validator]
)

# 3. Run the agent with guardrails in place
try:
    result = await Runner.run(my_agent, "Show me product X")
    print(result.final_output)
except (InputGuardrailTripwireTriggered, OutputGuardrailTripwireTriggered) as e:
    handle_exception(e)

