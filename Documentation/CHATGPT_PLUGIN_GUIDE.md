# Connecting a Small Case Portfolio to ChatGPT as a Plugin

This should be treated as an integration engineering project: define the data boundary and tool contracts first, then implement the smallest secure MCP server. We will not generate a large scaffold before understanding what each component does.

## 1. What “ChatGPT plugin” means today

OpenAI’s current plugin system has three possible components:

```text
Plugin
├── Skills
│   └── Instructions and repeatable workflows
│
└── MCP server
    ├── Tools ChatGPT can call
    ├── Authentication and authorization
    ├── Structured results
    └── Optional interactive UI
```

A plugin may contain:

- A skill only
- An MCP server only
- Skills plus an MCP server
- An MCP server with optional UI

For a portfolio containing live or private financial data, we will almost certainly need an **MCP server**. A skill alone cannot securely obtain account data from a broker or portfolio service.

OpenAI describes plugins as packages shared across supported ChatGPT and Codex surfaces. Public plugins use a common directory, although individual capabilities can still be product-specific. See the official [plugin architecture documentation](https://developers.openai.com/plugins/concepts/plugins).

## 2. The likely architecture

Assuming “small case portfolio” means your investment portfolio—possibly from the Smallcase platform—the system would look like this:

```text
You
 │
 ▼
ChatGPT
 │
 │ Selects an appropriate tool
 ▼
Your MCP server
 │
 ├── validates the request
 ├── verifies the current user
 ├── enforces permissions
 ├── retrieves only required data
 └── returns structured portfolio information
 │
 ▼
Portfolio data source
 ├── official Smallcase/broker API
 ├── your own database
 ├── periodically imported statements
 └── another approved source
```

The MCP server is a security boundary. ChatGPT should not receive broker credentials, API secrets, database credentials, or unrestricted access to an internal API.

An MCP tool might return:

```json
{
  "as_of": "2026-09-01T17:30:00+05:30",
  "currency": "INR",
  "invested_value": 250000,
  "current_value": 278500,
  "absolute_return": 28500,
  "absolute_return_percent": 11.4
}
```

ChatGPT can then explain or compare the result. It does not need direct access to the underlying account.

## 3. What the first version should do

We should begin with read-only capabilities.

| Tool | Purpose | Risk |
|---|---|---|
| `get_portfolio_summary` | Return invested value, current value, and returns | Low, but private |
| `list_holdings` | Return holdings and allocation | Private financial data |
| `get_allocation_breakdown` | Group by stock, sector, asset class, or smallcase | Private financial data |
| `get_portfolio_performance` | Return historical values over a period | Private financial data |
| `explain_portfolio` | Usually a skill using results from other tools | Interpretation risk |

Initially, I would explicitly exclude:

- Buying or selling securities
- Rebalancing a portfolio
- Placing broker orders
- Changing account settings
- Claiming personalized investment advice
- Inventing missing prices or historical values

Read and write tools should be separated because they have different authorization, confirmation, audit, and safety requirements. OpenAI’s [tool-planning guidance](https://developers.openai.com/plugins/plan/tools) makes the same distinction.

## 4. What we need to decide before coding

The most important question is not the programming language. It is the **source of truth**.

We need to identify:

1. What “small case” refers to:

   - Your Smallcase investment account
   - A custom portfolio application named Small Case
   - A spreadsheet or exported portfolio
   - Something else

2. Where the portfolio data currently lives:

   - Official API
   - Broker API
   - Database
   - CSV/Excel export
   - Scraped browser data
   - Only inside a website session

3. Who will use the plugin:

   - Only you
   - A private organization
   - Multiple customers
   - Anyone through the public plugin directory

4. What the first successful conversation should accomplish.

For example:

> Show my current portfolio allocation and identify the three largest concentration risks.

That single sentence determines the required data, tools, schemas, permissions, and test cases.

## 5. Authentication is not optional

There are two different identities:

- **ChatGPT user identity:** who is invoking the plugin?
- **Portfolio identity:** which investment account may that person access?

A production server must connect these identities securely.

A reasonable progression is:

1. Local prototype with synthetic portfolio data
2. Private test with your own account
3. Proper user authentication
4. Token storage and refresh handling
5. Multi-user authorization, if necessary
6. Public review only after privacy and reliability work

Secrets must remain server-side and encrypted. Logs should avoid portfolio contents and tokens. Responses should expose only the fields necessary for the current request.

Because this involves financial information, we will also distinguish carefully between:

- Factual portfolio analytics
- General educational explanations
- Personalized investment recommendations
- Actual trading actions

These are not interchangeable.

## 6. How ChatGPT discovers and uses tools

ChatGPT does not simply call arbitrary server endpoints. The MCP server publishes named tools with contracts.

Each tool needs:

- Stable name
- Clear user-oriented description
- Typed input schema
- Structured output schema
- Authorization rules
- Side-effect classification
- Predictable error behavior
- Safety annotations

Tool descriptions matter because the model uses them to decide whether and when to call a tool. We will write descriptions around user intent—not around internal endpoint names.

For example:

```text
get_portfolio_summary

Use when the user asks for the current value, invested amount,
overall gain or loss, or a high-level summary of their authorized
portfolio. This tool is read-only and does not provide trade execution.
```

## 7. Do we need a custom interface?

Not initially.

ChatGPT can explain structured tool results directly. An embedded interface becomes useful when users need to:

- Compare holdings
- Inspect allocation charts
- Change filters
- Navigate time periods
- Confirm an action
- Interact with a large structured result

OpenAI recommends that tools remain usable without UI. We can add a portfolio chart later without making the core tool dependent on it.

## 8. Development sequence

We will proceed in deliberate stages:

1. **Requirements**
   Define use cases, exclusions, data source, users, and privacy boundary.

2. **Tool design**
   Write contracts before implementation.

3. **Synthetic prototype**
   Implement a minimal MCP server using fake portfolio data.

4. **Protocol testing**
   Call tools directly and verify valid, invalid, missing, and unauthorized inputs.

5. **ChatGPT connection**
   Deploy or temporarily tunnel the MCP endpoint, enable Developer Mode, and create a personal plugin.

6. **Real-data integration**
   Add the approved portfolio data source and authentication.

7. **Security review**
   Test authorization, token handling, logging, data minimization, and failure behavior.

8. **Evaluation**
   Maintain a set of prompts that should call tools, should not call tools, and should be refused.

9. **Optional UI**
   Add visual portfolio exploration only if it improves the experience.

10. **Publication**
    Add privacy documentation, support information, listing metadata, and review readiness.

The official [plugin quickstart](https://developers.openai.com/plugins/quickstart) uses ChatGPT Developer Mode and a deployed MCP URL to create and test a personal plugin. Public publication is a later, separate step.

## 9. Our first concrete task

Place the existing portfolio code in this workspace—or identify where it is—and clarify whether “small case portfolio” means an account on the **Smallcase investment platform**.

Once that is established, we will create a written use-case inventory before writing server code. That inventory will become both the technical specification and the test plan.
