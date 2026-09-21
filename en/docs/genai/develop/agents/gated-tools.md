---
sidebar_position: 5
title: Gated Tools
description: Gate a tool so a WSO2 Integrator AI agent pauses for approval before it runs, then approve or reject and resume the run.
keywords: [wso2 integrator, gated tools, tool approval, approval gate, ai agents, tools]
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Gated Tools

By default, an agent runs on autopilot. It reasons and calls tools in a loop until it produces an answer, with no opportunity for a person to step in. That is fine for read-only actions such as looking up an order, but risky for sensitive ones such as issuing a refund, deleting a record, or sending an email.

Gating a tool makes the agent pause immediately before it runs that tool, show what it proposes to do, and continue once a person has approved or rejected the call.

:::info When to use something else
A gated tool answers one question: may this call run? The answer is yes or no, and nothing about the decision is recorded. If you need a named approver, a deadline, a record of who decided, a person who supplies a value rather than permitting an action, or a repair after a step fails, use a human task in a durable workflow instead.
:::

## How it works

Gating is opt-in for each tool. When you gate a tool, the agent stops before invoking it and reports the calls waiting for a decision. A person reviews each proposed call and approves or rejects it, and the run continues from exactly where it paused.

| Stage | What happens |
|---|---|
| **Pause** | The agent decides to call a gated tool. Instead of running it, the agent saves its state and returns the pending approval requests. |
| **Decide** | A person reviews each request, including the tool name and the arguments the agent proposes to use, and approves or rejects it. |
| **Resume** | The decisions are passed back to the agent. Approved calls run, rejected calls do not, and the agent continues reasoning. |

Two properties are worth knowing before you build on it.

- **The agent does not hold a thread while it waits.** A pending approval can be resolved seconds or days later, from a different process or server replica, as long as it can reach the same memory store.
- **Nothing changes for agents that do not use it.** If no tool is gated, the agent never pauses and existing agents behave exactly as before.

Rejection is not simply a failure. The agent receives the rejection, along with any reason you supply, as feedback and replans. That makes the reason field a useful way to redirect the agent, for example, "Refunds above 100 USD need manager sign-off. Create a ticket instead."

## 1. Gate a tool

Gating is configured on the tool, not on the agent. Select the **AI Agent** node in the agent canvas and click the **+** button to open the **Add Tool** panel, then choose how you want to add the tool. For details on each option, see [Tools](tools.md).

![Add tool](/img/genai/develop/agents/29-tool.png)

In the tool configuration panel, expand **Advanced Configurations** and select **Requires Approval**.

<!-- TODO: screenshot /img/genai/develop/agents/32-requires-approval-field.png : tool configuration panel with Requires Approval selected and Approval Function empty -->

Configure the following fields.

| Field | Required | Description |
|---|---|---|
| **Requires Approval** | No | Pauses this tool before it runs and waits for approval. Cleared by default. |
| **Approval Function** | No | Decides for each call whether approval is needed. Available only when **Requires Approval** is selected. Leave it empty to require approval for every call. See [Gate only some calls](#2-gate-only-some-calls). |

Save the tool with **Create Tool** or **Save Tool**.

These fields combine to give three behaviours.

| Configuration | Behaviour |
|---|---|
| **Requires Approval** cleared | The tool runs freely. The agent never pauses for it. |
| **Requires Approval** selected, **Approval Function** empty | Every call to this tool pauses for approval. |
| **Requires Approval** selected, **Approval Function** set | Only the calls the function accepts pause. |

### Where the field appears

**Requires Approval** is available for the following tool options.

| Tool option | Supports Requires Approval |
|---|---|
| **Use Function** | Yes |
| **Use Connection** | Yes |
| **Create Custom Tool** | Yes |
| **Use Agent** | Yes |
| **Use MCP Server** | No |

Tools discovered from an MCP server are generated from the remote server's tool list, so there is no local declaration on which to set the field. To gate an action provided by an MCP server, wrap it in a function or a custom tool and gate that instead.

## 2. Gate only some calls

Gating every call to a tool is often stricter than you need. A refund of 5 USD and a refund of 5000 USD are the same tool call, but only one of them warrants a person's attention. **Approval Function** lets you decide for each call, based on the arguments the agent proposes.

Select **Requires Approval**, then type a name in **Approval Function**. WSO2 Integrator generates a function next to the tool with the correct signature and a placeholder body.

<!-- TODO: screenshot /img/genai/develop/agents/33-approval-function-field.png : Approval Function field with a new function name typed in -->

<!-- TODO: screenshot /img/genai/develop/agents/34-approval-predicate-stub.png : the generated function returning boolean with its TODO comment -->

Open the generated function and replace the placeholder body with the real condition. Returning `true` pauses that call for approval, and returning `false` lets it run.

<Tabs>
<TabItem value="code" label="Ballerina Code">

```ballerina
// Generated for you. Pause only when the refund is large.
isolated function refundNeedsReview(string orderId, decimal amount) returns boolean =>
    amount > 100d;
```

</TabItem>
</Tabs>

Keep the following constraints in mind.

- The function takes the same parameters as the tool it gates and returns `boolean`. WSO2 Integrator generates the signature for you. If you edit it so that it no longer matches, compilation fails with diagnostic `AI_111`.
- The function must be deterministic for a given set of arguments. It runs in line with the agent's reasoning.
- The function fails safe. If it panics or does not return a `boolean`, the call pauses for approval rather than running unreviewed.

:::info
When you edit a tool that already exists, **Approval Function** accepts only functions already defined in your project. Create the function first, then select it. When you create a new tool, you can type a new name and WSO2 Integrator generates the function for you.
:::

## 3. See which tools are gated

Gated tools are marked with a badge in the top-right corner of the tool in the **AI Agent** node. Hover over the badge to see the **Requires Approval** tooltip. The badge is informational, and it gives you a way to confirm at a glance which tools can pause the agent.

<!-- TODO: screenshot /img/genai/develop/agents/35-tool-approval-badge.png : agent node showing the approval badge and its tooltip -->

## 4. Approve or reject in the agent chat

Open the chat interface from the agent canvas and send a message that leads the agent to a gated tool. Instead of a text reply, the agent responds with an approval card headed **Approval required**, followed by the number of pending requests.

<!-- TODO: screenshot /img/genai/develop/agents/36-chat-approval-card.png : chat approval card with arguments expanded and the input reading "Waiting on your decision…" -->

Each pending request shows its position in the batch, such as `1/2`, the tool name, and the tool description. Click **Show arguments** to inspect the exact arguments the agent proposes to use, and **Hide arguments** to collapse them again.

Respond using the following controls.

| Control | Description |
|---|---|
| **Approve** | Runs the proposed tool call as it stands. |
| **Reject** | Blocks the call and opens a box for an optional reason. |
| **Confirm Reject** | Submits the rejection along with the reason. |
| **Cancel** | Discards the rejection and returns to the **Approve** and **Reject** buttons. |
| **Approve All** | Approves every pending request. Appears only when more than one request is pending. |
| **Reject All** | Rejects every pending request. Appears only when more than one request is pending. |

<!-- TODO: screenshot /img/genai/develop/agents/37-reject-reason.png : reject reason box with Cancel and Confirm Reject -->

The reason you type is shown to the agent, so use it to explain what to do instead rather than only why the call was blocked.

While a decision is outstanding, the chat input is disabled and its placeholder reads **Waiting on your decision…**. Once every request is decided, the card collapses to a one-line summary of what was approved or rejected, and the agent continues its run.

<!-- TODO: screenshot /img/genai/develop/agents/38-approval-card-collapsed.png : collapsed approval card summarising what was approved and rejected -->

An agent can pause more than once in a single turn, so you may see several cards before you get a final answer.

## 5. Make pauses survive a restart {#make-pauses-survive-a-restart}

A paused run is stored as a checkpoint in the agent's memory store, keyed by the session ID. Where that store keeps its data determines whether a pending approval survives.

| Memory configuration | Pauses survive a restart? |
|---|---|
| Default in-memory store | No. Pending approvals are lost when the process stops, and another replica cannot see them. |
| A durable store, such as a database-backed store | Yes. A pause can be resolved after a restart or by another replica. |

The default is fine while you develop and test. For production, where a person may take hours to respond, attach a durable store to the agent's memory. See [Memory](memory.md#add-memory-store).

<!-- TODO: screenshot /img/genai/develop/agents/39-memory-store-attached.png : agent node with a database-backed short-term memory store attached -->

## Choose what to gate

Use the following table as a starting point.

| Situation | Recommended setup |
|---|---|
| The tool only reads data | Leave **Requires Approval** cleared. Gating read-only tools adds friction with no benefit. |
| The tool moves money, deletes data, or contacts a customer | Select **Requires Approval** and leave **Approval Function** empty. |
| The action is sensitive only past a threshold, such as a large refund | Select **Requires Approval** and set an **Approval Function** that tests the amount. |
| The action is sensitive only for certain records, such as production tenants | Select **Requires Approval** and set an **Approval Function** that inspects the identifier. |
| The tool comes from an MCP server | Wrap the call in your own function or custom tool and gate that. |
| A person may take hours to respond | Attach a durable memory store so the pause survives a restart. |

Two habits keep gating useful rather than tiring. Gate the smallest number of tools you can, because reviewers who approve everything by reflex provide no real oversight. And write specific tool descriptions, because the description is what the reviewer reads when deciding.

## Handle approvals in your own application

Everything above runs in the agent chat. If you are building your own front end instead, the same pauses are available to your code.

### Over HTTP

An agent exposed as a chat service handles approvals across separate stateless requests, correlated by the session ID. Add a `decision` resource alongside the usual `chat` resource.

<Tabs>
<TabItem value="code" label="Ballerina Code">

```ballerina
import ballerina/ai;
import ballerina/http;

listener ai:Listener chatListener = new (listenOn = check http:getDefaultListener());

service /support on chatListener {

    resource function post chat(@http:Payload ai:ChatReqMessage request)
            returns ai:ChatRespMessage|error {
        string|ai:Error result = supportAgent.run(request.message, request.sessionId);
        return result is ai:Error ? result : {message: result};
    }

    resource function post decision(@http:Payload ai:DecisionMessage request)
            returns ai:ChatRespMessage|error {
        string|ai:Error result = supportAgent.run({decisions: request.decisions},
            request.sessionId);
        return result is ai:Error ? result : {message: result};
    }
}
```

</TabItem>
</Tabs>

The service performs no error-to-HTTP mapping of its own. It returns the pause, and the listener converts it into a response.

| Situation | Status | Response body |
|---|---|---|
| The run paused for approval | `403` | `{"requests": [ ... ]}`, one entry for each pending request |
| A decision arrived but nothing is pending for the session | `404` | `{"errorType": "ApprovalNotFoundError", "message": "..."}` |
| A decision named an ID that is not pending | `400` | `{"errorType": "UnknownApprovalIdError", "message": "..."}` |
| The run completed | `200` | `{"message": "..."}` |

Branch on `errorType`, which is a stable discriminator. The `message` field is free text for humans and may be reworded, so do not depend on it.

:::warning Secure the decision resource outside the service
The listener attaches an internal dispatcher rather than your service, and it invokes your resource with the payload only. Your resource never sees request headers, so it cannot authenticate the caller itself, and declarative HTTP authentication declared on the service does not apply to it.

Anyone who can reach the endpoint and knows a session ID and a request ID can approve a pending call. Terminate authentication at an API gateway in front of the integration, and restrict the endpoint by network policy so that only your own front end can reach it.
:::

### In code

`run` returns an `ai:ApprovalRequiredError` when the agent pauses. Passing an `ai:Resume` back to `run`, rather than a query, is what continues the paused run. There is no separate resume operation, and the input type is what distinguishes a new turn from a continuation.

<Tabs>
<TabItem value="code" label="Ballerina Code">

```ballerina
string|ai:Error result = supportAgent.run(userInput, sessionId);

// A single turn may propose several gated calls, and the agent can pause more
// than once, so loop until the run is no longer waiting on a person.
while result is ai:ApprovalRequiredError {
    map<ai:HumanResponse> decisions = {};
    foreach ai:ApprovalRequest req in result.detail().requests {
        io:println(string `Approval needed for '${req.toolName}' ` +
            string `with arguments ${req.arguments.toString()}`);
        string decision = io:readln("Approve? (yes / no): ");
        if decision == "yes" {
            decisions[req.id] = {decision: ai:APPROVE};
        } else {
            string reason = io:readln("Reason for rejection: ");
            decisions[req.id] = {decision: ai:REJECT, reason};
        }
    }

    // `ai:Resume` is a readonly record, so freeze the map before passing it.
    result = supportAgent.run({decisions: decisions.cloneReadOnly()}, sessionId);
}
```

</TabItem>
</Tabs>

The types involved are as follows.

| Type | Purpose |
|---|---|
| `ai:ApprovalRequiredError` | Returned by `run` when the agent pauses. Its detail carries an `ai:ApprovalRequest[]` in the `requests` field. |
| `ai:ApprovalRequest` | One proposed tool call awaiting a decision. Carries `id`, `sessionId`, `toolName`, `toolDescription`, `arguments`, and `batchIndex`. |
| `ai:HumanResponse` | A decision, made up of `decision` and an optional `reason`. |
| `ai:ApprovalDecision` | The enum `ai:APPROVE` or `ai:REJECT`. |
| `ai:Resume` | A readonly record passed to `run` to continue a paused run. Holds `decisions`, keyed by each request's `id`. |
| `ai:ApprovalNotFoundError` | Returned when a resume arrives for a session with nothing pending. |
| `ai:UnknownApprovalIdError` | Returned when a resume names an ID that is not currently pending. |

Two behaviours help keep real applications correct.

- **Partial decisions are allowed.** If you supply decisions for only some of the pending requests, the rest stay pending and `run` returns a fresh `ai:ApprovalRequiredError` listing only those still undecided.
- **A pending approval blocks new turns.** Calling `run` with a new query while an approval is outstanding returns the same `ai:ApprovalRequiredError` instead of starting an unrelated turn. Resolve the pending decision first.

### Gating in source

Selecting **Requires Approval** sets the `requiresApproval` field of the `@ai:AgentTool` annotation. You can also set it directly in source view.

<Tabs>
<TabItem value="code" label="Ballerina Code">

```ballerina
// A read-only tool. The agent calls this freely.
@ai:AgentTool
isolated function lookupOrder(string orderId) returns Order|error {
    // Return the order for the given ID.
}

// A gated tool. The agent pauses before every call.
@ai:AgentTool {requiresApproval: true}
isolated function issueRefund(string orderId, decimal amount) returns string|error {
    // Issue the refund.
}

// A gated tool that pauses only above a threshold.
@ai:AgentTool {requiresApproval: refundNeedsReview}
isolated function issueLargeRefund(string orderId, decimal amount) returns string|error {
    // Issue the refund.
}
```

</TabItem>
</Tabs>

`@ai:AgentTool` can only be applied to functions you declare. A toolkit that builds its `ai:ToolConfig` values by hand sets the same field directly on the config. Both routes feed the same set of rules, so the agent behaves identically regardless of how a tool was gated.

### Custom memory stores

If you implement a custom `ai:ShortTermMemoryStore`, it must provide four checkpoint methods in addition to the message methods: `putCheckpoint`, `getCheckpoint`, `removeCheckpoint`, and `takeCheckpoint`. `takeCheckpoint` fetches and removes the pending approval atomically, so that a duplicate resume for the same session cannot execute the same call twice.

:::warning
This is a breaking change introduced with gated tool support. A custom `ai:ShortTermMemoryStore` written before this release must add these four methods to keep conforming. See [Custom memory](memory.md#custom-memory).
:::

## What's next

- **[Tools](tools.md)** — Add functions, connectors, and integrations to your agents.
- **[Memory](memory.md)** — Configure conversational and persistent memory, including durable stores.
- **[Observability](observability.md)** — Trace which tools the agent selects and when it pauses.
