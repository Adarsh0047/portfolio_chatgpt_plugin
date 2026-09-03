# Portfolio ChatGPT Plugin — Requirements Specification

Status: Draft — synthetic MVP approved in principle; production sources partially resolved

Version: 0.3

Last updated: 2026-09-02

## 1. Purpose

Build a ChatGPT plugin that lets an authorized user inspect and understand their investment portfolio through natural-language requests.

The first release will be read-only. It will retrieve factual portfolio data from an approved source and return structured, traceable results to ChatGPT. It will not place trades, rebalance investments, or modify an account.

## 2. Confirmed product decisions

| ID | Decision | Status |
|---|---|---|
| D-01 | The portfolio is held on the Smallcase investment platform. | Confirmed |
| D-02 | The Smallcase portfolio uses Groww as its broker. Groww's official API is the target source for stock-level holdings and valuation data; authenticated website scraping is excluded. | Confirmed |
| D-03 | Version 1 is for one user: the repository owner. | Confirmed |
| D-04 | The primary capabilities are portfolio P/L and individual-stock performance, discovery of other Smallcases, and information about owned and other Smallcases. | Confirmed |
| D-05 | The implementation language is Python. | Confirmed |
| D-06 | Development will begin with synthetic data before connecting a real account. | Confirmed |

Additional scope decisions:

- Version 1 is strictly read-only.
- The plugin provides factual analytics and educational explanations, not personalized financial advice.
- A custom embedded UI is not required for the first working version.

## 3. Intended user

### Primary user

An investor who owns or is authorized to view the portfolio and wants to query it conversationally.

### User needs

The user should be able to:

- Obtain a current portfolio summary.
- Inspect individual holdings and their weights.
- Understand allocation and concentration.
- Review gains and losses using values supplied by the portfolio data source.
- Discover Smallcases available on the platform.
- Inspect descriptive and performance information about owned and other Smallcases.
- Ask follow-up questions without manually copying portfolio data into ChatGPT.
- Understand when data was last updated and where it came from.

## 4. Product boundary

### The plugin will

- Expose a deliberately limited set of read-only portfolio operations.
- Authenticate the caller before returning private portfolio data.
- Authorize access to the requested portfolio on every relevant operation.
- Validate all tool inputs on the server.
- Return structured results with currency and `as_of` timestamps.
- Distinguish source data from calculations derived by the plugin.
- Produce understandable errors without leaking secrets or internal diagnostics.

### The plugin will not, in version 1

- Buy, sell, or rebalance securities.
- Place, modify, or cancel broker orders.
- Transfer money or securities.
- Change account, broker, or Smallcase settings.
- Store broker passwords.
- Promise returns or predict future performance as fact.
- Present generated text as regulated or personalized investment advice.
- Fill missing financial data with invented values.
- Scrape authenticated websites unless that approach is separately reviewed and explicitly approved.

## 5. Proposed MVP use-case inventory

Each use case will later map to one or more MCP tool contracts. Tool names in this document are provisional.

### UC-01: View portfolio summary

| Field | Requirement |
|---|---|
| User goal | Understand the portfolio's current high-level position. |
| Example requests | “Show my portfolio summary.” “How much have I invested?” “What is my current gain or loss?” |
| Expected result | Invested value, current value, absolute gain or loss, percentage return, currency, and data timestamp. |
| Required context | Authenticated user, authorized portfolio, current portfolio valuation. |
| Proposed capability | MCP tool: `get_portfolio_summary`. |
| Safety boundary | Private financial information; read-only. |
| Support decision | Proposed for MVP. |

### UC-02: List holdings

| Field | Requirement |
|---|---|
| User goal | See which securities are held and their current portfolio weights. |
| Example requests | “List my holdings.” “What are my five largest positions?” “Do I own INFY?” |
| Expected result | Stable instrument identifier, display name, symbol, quantity where available, current value, portfolio weight, currency, and timestamp. |
| Required context | Authenticated user, authorized portfolio, holding-level data. |
| Proposed capability | MCP tool: `list_holdings`, with server-side sorting and limiting. |
| Safety boundary | Private financial information; read-only. |
| Support decision | Proposed for MVP. |

### UC-03: Analyze allocation and concentration

| Field | Requirement |
|---|---|
| User goal | Understand how concentrated or diversified the portfolio is. |
| Example requests | “Show allocation by holding.” “Which positions dominate my portfolio?” “What percentage is in the top three holdings?” |
| Expected result | Allocation groups, weights, calculation basis, timestamp, and clearly identified concentration metrics. |
| Required context | Authorized holdings and reliable classifications when grouping beyond individual securities. |
| Proposed capability | MCP tool: `get_allocation_breakdown`; a skill may provide a consistent educational explanation. |
| Safety boundary | Analytics must not be framed as personalized buy or sell advice. |
| Support decision | Proposed for MVP for holding-level allocation; sector or asset-class grouping depends on source data. |

### UC-04: Review holding-level gains and losses

| Field | Requirement |
|---|---|
| User goal | Identify which holdings contribute most to current gains or losses. |
| Example requests | “Which holdings gained the most?” “Show my largest unrealized losses.” |
| Expected result | Holding-level invested value, current value, gain or loss, percentage return, currency, and timestamp where the source supports those fields. |
| Required context | Authorized cost-basis and valuation data. |
| Proposed capability | Either filters on `list_holdings` or a focused read-only tool, decided during tool design. |
| Safety boundary | Cost basis is sensitive; calculations must identify missing or partial data. |
| Support decision | Conditional on data-source support. |

### UC-05: Review historical performance

| Field | Requirement |
|---|---|
| User goal | Understand how portfolio value or returns changed over a selected period. |
| Example requests | “How did my portfolio perform this year?” “Compare the last three months.” |
| Expected result | A dated series, selected period, return methodology, currency, and source timestamp. |
| Required context | Reliable historical snapshots or an upstream performance API. |
| Proposed capability | MCP tool: `get_portfolio_performance`. |
| Safety boundary | Results must distinguish deposits and withdrawals from investment performance when possible. |
| Support decision | Deferred until historical-data availability is confirmed. |

### UC-06: Discover other Smallcases

| Field | Requirement |
|---|---|
| User goal | Find Smallcases available on the platform using supported discovery criteria. |
| Example requests | “Show me other Smallcases.” “Find Smallcases related to technology.” “Which Smallcases match this theme?” |
| Expected result | Smallcase identifier, name, description, manager or publisher where available, category or theme, image or detail link where allowed, source timestamp, and supported summary statistics. |
| Required context | Access to Smallcase discovery data through an approved API; the request may not require access to the user's portfolio. |
| Proposed capability | MCP tool: `discover_smallcases`, with explicit search, filter, sorting, and pagination inputs supported by the upstream API. |
| Safety boundary | Discovery must not be presented as a recommendation or guarantee of suitability or returns. Sponsored or promoted ordering must not be misrepresented as an objective ranking. |
| Support decision | Proposed for MVP synthetic data; real-data support depends on Smallcase Gateway access and permitted discovery fields. |

### UC-07: View information about a Smallcase

| Field | Requirement |
|---|---|
| User goal | Understand the composition, description, strategy, manager, statistics, and available performance information for an owned or discovered Smallcase. |
| Example requests | “Tell me about my Smallcase.” “What does this Smallcase invest in?” “Show details for this Smallcase.” |
| Expected result | Stable Smallcase identifier, ownership status when known, descriptive metadata, constituent or configuration information when permitted, performance information with its methodology and period, and source timestamp. |
| Required context | A resolvable Smallcase identifier plus either user-investment access or public discovery/detail access. |
| Proposed capability | MCP tool: `get_smallcase_details`; ownership-specific fields may be enriched from the authorized user's investments. |
| Safety boundary | Historical performance must not be represented as a promise of future returns. Information and ownership-specific results must remain distinguishable. |
| Support decision | Proposed for MVP synthetic data; real-data fields depend on Smallcase Gateway access. |

## 6. External data-source findings

### 6.1 Groww API for personal holdings

The owner uses Groww as the broker connected to Smallcase. Groww's official Trading API is therefore the intended production source for stock-level portfolio data.

The official Groww documentation states that individual Groww account holders can use the API with an active Trading API subscription. It provides a Python SDK and documents:

- Holdings with ISIN, trading symbol, quantity, and average price.
- Positions, including realized P/L where applicable.
- Instrument reference data.
- Live market data, including latest traded price.
- Historical market-price data.

For long-term holdings, the plugin can derive an indicative unrealized P/L from the Groww-provided quantity and average price combined with a Groww-provided current price. The exact formula, timestamp, rounding, treatment of unavailable prices, and interpretation of special quantities must be documented and tested before production use.

Groww's documented holdings schema does not include Smallcase membership, Smallcase identifiers, strategy metadata, or Smallcase-level return attribution. Therefore:

- Groww can support stock-level holdings, valuation, and P/L.
- Groww cannot be assumed to identify which holdings or quantities belong to a particular Smallcase.
- Groww cannot supply the Smallcase discovery catalog or descriptive Smallcase metadata.
- The plugin must not infer Smallcase membership merely by comparing a user's stocks with a model portfolio.

Authentication remains server-side. Tokens and API secrets must never be exposed to ChatGPT, stored in synthetic fixtures, or committed to Git. Token lifetime and renewal behavior will be specified during production-adapter design.

References:

- [Groww Trading API introduction and authentication](https://groww.in/trade-api/docs/curl)
- [Groww Python SDK portfolio API](https://groww.in/trade-api/docs/python-sdk/portfolio)
- [Groww live-data API](https://groww.in/trade-api/docs/curl/live-data)
- [Groww instrument data](https://groww.in/trade-api/docs/curl/instruments)

### 6.2 Smallcase Gateway for Smallcase-specific data

Smallcase Gateway would be the natural source for Smallcase-specific data, but it is not a self-service individual-user API. It remains unavailable to this project unless Smallcase grants partner integration access.

Official Smallcase documentation currently describes APIs for:

- Importing stock, ETF, and Smallcase holdings.
- Fetching a user's Smallcase investments.
- Discovering Smallcases.
- Fetching Smallcase details and historical returns.

The documented Gateway integration requires credentials issued through Smallcase's business onboarding:

- Gateway name.
- Shared secret used for JWT signing and verification.
- API secret used for server-side API authentication.

Fetching holdings also requires a connected-user token and one-time user consent through the Holdings Import flow. A normal consumer Smallcase login must not be assumed to provide these developer credentials.

Consequences for the build:

1. The synthetic adapter can model both Groww portfolio data and Smallcase catalog data without credentials.
2. The production Groww adapter can provide stock-level portfolio information after the owner subscribes and configures credentials.
3. Real Smallcase grouping, owned-Smallcase details, and platform discovery remain blocked without an authorized source.
4. Authenticated website scraping is not an accepted fallback.
5. A user-maintained or exported data file may later be evaluated for Smallcase-specific metadata, but it must be treated as a separate, timestamped source.

References:

- [Smallcase Gateway: Getting started](https://developers.gateway.smallcase.com/docs/getting-started)
- [Smallcase Gateway: List of APIs](https://developers.gateway.smallcase.com/docs/list-of-apis)
- [Smallcase Gateway: Holdings Import](https://developers.gateway.smallcase.com/docs/holdings-import)
- [Smallcase Gateway: Fetch holdings](https://developers.gateway.smallcase.com/reference/fetch-holdings)
- [Smallcase Gateway: Fetch Smallcase investments](https://developers.gateway.smallcase.com/reference/user-investments)

## 7. Requests that must not invoke portfolio tools

The evaluation suite must include requests for which portfolio access is unnecessary or inappropriate:

- General definitions such as “What is diversification?”
- Requests about another person's portfolio without authorization.
- Requests for broker passwords, access tokens, or internal credentials.
- Requests to fabricate unavailable prices, returns, or holdings.
- Trade execution requests in the read-only version.
- Unrelated conversation.

## 8. Data requirements

Every financial response must state or structurally include:

- The portfolio identifier in a non-secret form.
- The valuation currency.
- The time at which the data was valid (`as_of`).
- The upstream source or source category.
- Whether each result is source-provided or plugin-derived.
- Missing, stale, partial, or unavailable fields.

The implementation must not assume that:

- All holdings use the same exchange.
- All instruments are equities.
- Quantity is an integer.
- Cost basis is always available.
- Current value is real-time.
- Percentage values and monetary values can be mixed without an explicit currency or basis.

## 9. Authentication and authorization requirements

Before real portfolio data is connected:

- The authentication method must be documented.
- Access tokens must remain server-side.
- Secrets must be loaded from an approved secret store or environment, never committed to Git.
- The server must validate authorization for the requested portfolio, not merely authenticate the caller.
- Tokens and sensitive portfolio payloads must not appear in normal application logs.
- Authentication failures must not reveal whether another user's portfolio exists.
- Any cached data must have a documented lifetime and deletion policy.

For a single-user prototype, authentication may be simplified only if the server remains private and the limitation is documented. Synthetic local data requires no account authentication but must be unmistakably labelled as synthetic.

## 10. Financial-safety requirements

- Calculations must be deterministic and testable.
- Monetary calculations must avoid binary floating-point where precision affects correctness.
- Return methodology must be named or documented.
- The plugin must not turn incomplete data into confident conclusions.
- Educational observations must remain distinguishable from personalized financial advice.
- The plugin must never claim that portfolio data is current unless the source timestamp supports that claim.
- Any future state-changing operation requires a separate requirements and threat review.

## 11. Non-functional requirements

### Privacy

Return only the data needed for the request. Do not expose credentials, internal identifiers, or unnecessary personal information to ChatGPT.

### Reliability

The server must return explicit, machine-readable errors for invalid input, unavailable upstream services, stale data, authentication failure, and authorization failure.

### Observability

Log operational events using correlation identifiers while excluding secrets and raw portfolio contents. Metrics should cover latency, failure category, and tool usage without identifying holdings.

### Maintainability

Business calculations, data-source adapters, MCP transport, and authentication should be separable so each can be tested independently.

### Portability

Core tools should return useful structured and textual results without requiring a custom UI.

## 12. MVP acceptance criteria

The first read-only prototype is successful when:

1. It runs against a documented synthetic portfolio dataset.
2. ChatGPT can request a portfolio summary through an MCP tool.
3. ChatGPT can list and rank holdings through an MCP tool.
4. ChatGPT can explain holding-level concentration using returned data.
5. ChatGPT can discover synthetic Smallcases using documented filters.
6. ChatGPT can retrieve synthetic details for both owned and non-owned Smallcases.
7. Ownership-specific data is not returned for a non-owned Smallcase.
8. Every financial result includes currency where applicable and an `as_of` timestamp.
9. Invalid inputs return stable, understandable errors.
10. Requests outside the supported scope do not trigger misleading tool calls.
11. No secret or real account data is stored in the repository.
12. Automated tests verify calculations and tool schemas.
13. The documented synthetic examples reproduce the expected values exactly.

Connection to a real portfolio is a separate acceptance milestone and will not be considered complete until authentication, authorization, and data-source terms have been reviewed.

## 13. Remaining decisions and dependencies

| ID | Decision or dependency | Status |
|---|---|---|
| R-01 | Activate or confirm an active Groww Trading API subscription before production integration. | Open; does not block the synthetic prototype. |
| R-02 | Choose the Groww authentication flow and document token renewal. | Deferred to production-adapter design. |
| R-03 | Identify an authorized source for owned-Smallcase grouping and Smallcase catalog/details. | Open; no suitable individual API is currently confirmed. |
| R-04 | Decide freshness thresholds for holdings, current prices, and imported Smallcase metadata. | Deferred to production-adapter design. |
| R-05 | Decide deployment and private authentication after the local MCP prototype works. | Deferred. |

## 14. Requirements-to-test traceability

Each supported use case will produce four categories of evaluation prompts:

- **Direct:** explicitly names the plugin or requested portfolio operation.
- **Indirect:** states the user goal without naming a tool.
- **Edge case:** supplies missing, invalid, ambiguous, or unsupported parameters.
- **Out of scope:** should not call a portfolio tool or should explain the limitation.

The use-case identifiers in this document will be referenced from tool contracts and tests so that every implemented capability has a documented reason to exist.

## 15. Step-completion rule

Requirements work is complete only when:

- Decisions D-01 through D-06 have answers.
- Supported, deferred, and excluded use cases are approved.
- The source of truth and its freshness behavior are documented.
- The privacy boundary and intended users are explicit.
- MVP acceptance criteria are agreed upon.

The synthetic MVP requirements may be approved before R-01 through R-05 are resolved. Production integration requirements cannot be finalized until those dependencies are resolved.

Only then should Step 2—detailed MCP tool design—begin.
