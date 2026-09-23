---
sidebar_position: 4
title: Gated Tools
description: Gate a tool so a WSO2 Integrator AI agent pauses for approval before it runs, then approve or reject and resume the run.
keywords: [wso2 integrator, gated tools, tool approval, approval gate, ai agents, tools]
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Gated Tools

By default, an agent runs on autopilot. It reasons and calls tools in a loop until it produces an answer, with no opportunity for a person to step in. That is fine for read-only actions such as looking up an order, but risky for sensitive ones such as issuing a refund, deleting a record, or sending an email.

Gating a tool makes the agent pause immediately before it runs that tool, show what it proposes to do, and continue once a person has approved or rejected the tool call.

:::info When to use something else
A gated tool answers one question: may this tool call run? The answer is yes or no, and nothing about the decision is recorded. If you need a named approver, a deadline, a record of who decided, a person who supplies a value rather than permitting an action, or a repair after a step fails, use a human task in a durable workflow instead.
:::

## How it works

Gating is opt-in for each tool. When you gate a tool, the agent stops before invoking it and reports the tool calls waiting for a decision. A person reviews each proposed tool call and approves or rejects it, and the run continues from exactly where it paused.

| Stage | What happens |
|---|---|
| **Pause** | The agent decides to call a gated tool. Instead of running it, the agent saves its state and returns the pending approval requests. |
| **Decide** | A person reviews each request, including the tool name and the arguments the agent proposes to use, and approves or rejects it. |
| **Resume** | The decisions are passed back to the agent. Approved tool calls run, rejected ones do not, and the agent continues reasoning. |

Two properties are worth knowing before you build on it.

- **The agent does not hold a thread while it waits.** A pending approval can be resolved seconds or days later, from a different process or server replica, as long as it can reach the same memory store.
- **Nothing changes for agents that do not use it.** If no tool is gated, the agent never pauses and existing agents behave exactly as before.

Rejection is not simply a failure. The agent receives the rejection, along with any reason you supply, as feedback and replans. That makes the reason field a useful way to redirect the agent, for example, "Refunds above 100 USD need manager sign-off. Create a ticket instead."

## 1. Gate a tool

Gating is configured on the tool, not on the agent. Select the **AI Agent** node in the agent canvas and click the **+** button to open the **Add Tool** panel, then choose how you want to add the tool. For details on each option, see [Tools](tools.md).

![Add tool](/img/genai/develop/agents/29-tool.png)

In the tool configuration panel, tick **Requires Approval**.

![Tool configuration panel with Requires Approval ticked and Approval Function empty](/img/genai/develop/agents/gated-tools/required-approval-field.png)

Configure the following fields.

| Field | Description |
|---|---|
| **Requires Approval** | Optional. Pauses the tool before it runs and waits for approval. Off by default. |
| **Approval Function** | Optional. Decides for each tool call whether approval is needed. Available only when **Requires Approval** is ticked. Leave it empty to require approval for every call. For conditional gating, see [Gate a tool conditionally](#2-gate-a-tool-conditionally). |

Save the tool with **Create Tool** or **Save Tool**.

These fields combine to give three behaviours.

| Configuration | Behaviour |
|---|---|
| **Requires Approval** off | The tool runs freely. The agent never pauses for it. |
| **Requires Approval** ticked, **Approval Function** empty | Every call to this tool pauses for approval. |
| **Requires Approval** ticked, **Approval Function** set | Only the tool calls the function accepts pause. |

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

## 2. Gate a tool conditionally

Gating every call to a tool is often stricter than you need. A refund of 5 USD and a refund of 5000 USD are the same tool call, but only one of them needs a person's attention. **Approval Function** lets you decide for each call, based on the arguments the agent proposes.

Tick **Requires Approval**, then set **Approval Function** either by picking one of your project's own functions, or by typing a new name. Typing a new name has WSO2 Integrator generate a function next to the tool with the correct signature and a placeholder body. Generating a function this way is only available while you are creating the tool. If you edit a tool that already exists, **Approval Function** offers only your project's functions to pick from, so create the function first, then select it.

![Approval Function field with a new function name typed in](/img/genai/develop/agents/gated-tools/approval-function-field.png)

<!-- TODO: screenshot /img/genai/develop/agents/34-approval-predicate-stub.png : the generated function returning boolean with its TODO comment -->

Open the generated function and replace the placeholder body with the real condition. Returning `true` pauses that tool call for approval, and returning `false` lets it run.

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

- The function must take the same parameters as the tool it gates and return `boolean`, whether it is generated for you or picked from your project. **Approval Function** does not filter its picker by signature, so if you pick or edit a function so that it no longer matches, the project fails to build.
- The function runs synchronously, as part of the agent's reasoning, so keep it fast rather than doing something slow like a network call. Nothing caches its result, so it must return the same answer every time it is called with the same arguments.
- The function fails safe. If it panics or does not return a `boolean`, the tool call pauses for approval rather than running unreviewed.

## 3. See which tools are gated

Gated tools are marked with a badge in the bottom-right corner of the tool in the **AI Agent** node. Hover over the badge to see the **Requires Approval** tooltip. The badge is informational, and it gives you a way to confirm at a glance which tools can pause the agent.

![Agent node showing the approval badge and its tooltip](/img/genai/develop/agents/gated-tools/tool-approval-badge.png)

## 4. Approve or reject in the agent chat

Open the chat interface from the agent canvas and send a message that leads the agent to a gated tool. Instead of a text reply, the agent responds with an approval card headed **Approval required**, followed by the number of pending requests.

<!-- TODO: screenshot /img/genai/develop/agents/36-chat-approval-card.png : chat approval card with arguments expanded and the input reading "Waiting on your decision…" -->

Each pending request shows its position in the batch, such as `1/2`, the tool name, and the tool description. Click **Show arguments** to inspect the exact arguments the agent proposes to use, and **Hide arguments** to collapse them again.

Respond using the following controls.

| Control | Description |
|---|---|
| **Approve** | Runs the proposed tool call as it stands. |
| **Reject** | Blocks the tool call and opens a box for an optional reason. |
| **Confirm Reject** | Submits the rejection along with the reason. |
| **Cancel** | Discards the rejection and returns to the **Approve** and **Reject** buttons. |
| **Approve All** | Approves every pending request. Appears only when more than one request is pending. |
| **Reject All** | Rejects every pending request. Appears only when more than one request is pending. |

<!-- TODO: screenshot /img/genai/develop/agents/37-reject-reason.png : reject reason box with Cancel and Confirm Reject -->

The reason you type is shown to the agent, so use it to explain what to do instead rather than only why the tool call was blocked.

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
| The tool only reads data | Leave **Requires Approval** off. Gating read-only tools adds friction with no benefit. |
| The tool moves money, deletes data, or contacts a customer | Tick **Requires Approval** and leave **Approval Function** empty. |
| The action is sensitive only past a threshold, such as a large refund | Tick **Requires Approval** and set an **Approval Function** that tests the amount. |
| The action is sensitive only for certain records, such as production tenants | Tick **Requires Approval** and set an **Approval Function** that inspects the identifier. |
| The tool comes from an MCP server | Wrap the operation in your own function or custom tool and gate that. |
| A person may take hours to respond | Attach a durable memory store so the pause survives a restart. |

Two habits keep gating useful rather than tiring. Gate the smallest number of tools you can, because reviewers who approve everything by reflex provide no real oversight. And write specific tool descriptions, because the description is what the reviewer reads when deciding.

## What's next

- **[Tools](tools.md)** — Add functions, connectors, and integrations to your agents.
- **[Memory](memory.md)** — Configure conversational and persistent memory, including durable stores.
- **[Observability](observability.md)** — Trace which tools the agent selects and when it pauses.
