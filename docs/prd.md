# ShopPilot: Product Requirements Document

Status: draft v1 (Day 2). Targets marked "hypothesis" are replaced by measured baselines on eval days.

## 1. Problem

Small online merchants struggle with three things:

1. **Messy catalogs.** Product data arrives with abbreviations, typos, and missing attributes, so products are hard to find and hard to sell.
2. **Weak search.** Keyword-only search fails on natural queries like "warm jacket under 100 euros".
3. **Slow, repetitive support.** Customers ask the same shipping, return, and order-status questions all day.

Existing platforms offer AI as a bolt-on. ShopPilot treats AI as part of the core: it cleans catalogs, powers search, assists shoppers through a conversational agent, and answers support questions grounded in store policies.

## 2. Users

| User | Description | Main needs |
|---|---|---|
| **Merchant** | Owns one or more stores | Manage products and stock, see orders, trust AI suggestions, control costs |
| **Customer** | Shops in a store | Find products fast, buy safely, track orders, get answers |
| **Admin** | Platform operator | See usage, cost, and quality per store; handle abuse |

## 3. Goals

- G1. A complete multi-store commerce core: catalog, inventory, cart, orders, payments (Stripe test mode).
- G2. Strict **tenant isolation**: one store can never read or modify another store's data.
- G3. **Correctness under load**: no overselling, no duplicate orders from replayed webhooks.
- G4. AI features that are **measured**: each has a metric, a baseline, an eval set, and a failure mode.
- G5. AI features that **degrade gracefully**: if the LLM provider is down, the storefront still works.
- G6. **Cost visibility**: every LLM call records model, tokens, cost, latency, and store.
- G7. Deployed, observable, and documented well enough to demo and explain end to end.

## 4. Non-goals (v1)

- No multi-currency conversion, tax engine, or shipping-rate calculation (single currency, flat shipping).
- No marketplace features (cross-store carts, seller payouts).
- No native mobile apps.
- No custom domain management per store beyond the platform's own domain.
- No real-money payments (Stripe test mode only).
- No agent actions that spend money without explicit user approval.
- No training a foundation model from scratch (fine-tuning is an optional add-on).
- No microservices (see ADR 0001).

## 5. User stories and acceptance criteria

### Accounts and tenancy

**US-01 Register and log in.** As a user, I want to create an account and log in so that I can use the platform securely.
- Given valid details, when I register, then my account is created and my password is stored only as an argon2 hash.
- Given a protected endpoint, when I call it without a valid token, then I get 401.
- Given an expired access token and a valid refresh token, when I refresh, then I get a new access token.

**US-02 Create a store.** As a merchant, I want to create a store so that I can sell products.
- Given I am logged in as a merchant, when I create a store, then I become its owner.
- Given I am a customer, when I try to create a store, then I am refused.

**US-03 Tenant isolation.** As a merchant, I want my data invisible to other stores so that my business is private.
- Given stores A and B, when a user of A requests B's product, order, or inventory by ID, then the response is not found or forbidden.
- This holds at the database level (row-level security), proven by tests, not just at the API level.

### Catalog and inventory

**US-04 Manage products.** As a merchant, I want to create, edit, and list products with variants and images so that customers can browse them.
- Given a product with variants, when I list products, then results are paginated, filterable, and sortable.
- Given an image upload, when it completes, then the product shows the image.

**US-05 Track stock.** As a merchant, I want stock tracked per variant with an audit trail so that I know what is available and why it changed.
- Given a stock adjustment, when it is saved, then an audit record stores who, when, delta, and reason.

### Buying

**US-06 Cart.** As a customer, I want a cart so that I can collect items before buying.
- Given items in my cart, when totals are computed, then price calculation happens only on the server.

**US-07 Checkout without overselling.** As a customer, I want checkout to be reliable so that I never pay for something unavailable.
- Given 10 units in stock and 100 simultaneous checkouts, when they run, then exactly 10 succeed and stock never goes negative.
- Given a reservation that expires, when time passes, then the stock is released.

**US-08 Pay.** As a customer, I want to pay by card so that my order is confirmed.
- Given a Stripe test card, when I pay, then the order moves from pending to paid.
- Given the same Stripe webhook delivered twice, when it is processed, then exactly one order effect occurs.

**US-09 Track orders.** As a customer, I want to see my order status so that I know what is happening.
- Given an order, when its state changes, then I see the new state; illegal transitions (for example cancelled to fulfilled) are rejected.

### AI features

**US-10 Catalog enrichment.** As a merchant, I want AI to clean and complete messy product data so that products are easier to find.
- Given a messy product, when enrichment runs, then I get schema-valid structured output with a confidence score.
- Given low confidence, when enrichment finishes, then the item goes to a human-review queue instead of going live.
- Given my approval or edit, when I save, then the change goes live and my edit is stored as feedback data.

**US-11 Smart search.** As a customer, I want to search in natural language so that I find what I mean, not just what I typed.
- Given "warm jacket under 100 euros", when I search, then filters (price) and semantic intent (warm jacket) are both applied.
- Given the LLM provider is unavailable, when I search, then keyword search still returns results.

**US-12 Shopping assistant.** As a customer, I want a chat assistant that can find, compare, and check stock so that shopping is faster.
- Given a question, when the agent answers, then product facts come from tool results, not from model memory.
- Given a request to check out, when the agent proceeds, then it requires my explicit confirmation first.
- Given a request for another store's data, when the agent tries a tool, then the tool refuses regardless of what the model says.

**US-13 Review summaries.** As a customer, I want pros and cons summarized from reviews so that I can decide quickly.
- Given reviews, when a summary is generated, then each claim cites review IDs and is checked for faithfulness.

**US-14 Support bot.** As a customer, I want answers about returns, shipping, and my order so that I don't wait for a human.
- Given a policy question, when the bot answers, then it cites the policy source or refuses.
- Given a case it cannot resolve, when it detects this, then it escalates to a human.

### Operations

**US-15 Cost control.** As a merchant or admin, I want AI usage budgets so that costs cannot run away.
- Given a store over its token budget, when it makes an AI request, then it receives a graceful error, not a crash.

**US-16 Observability.** As the operator, I want traces, metrics, and alerts so that I can find and fix problems.
- Given a chat request, when it completes, then a trace shows the prompt version, retrieved items, tool calls, and cost.

## 6. Success metrics

Targets are **hypotheses** until a baseline is measured on the day listed.

| Area | Metric | Initial target | Measured on |
|---|---|---|---|
| Correctness | Oversold units under concurrent checkout | 0 | Day 19 |
| Correctness | Duplicate order effects from replayed webhooks | 0 | Day 21 |
| Security | Cross-tenant reads in isolation test suite | 0 | Days 10-11 |
| Enrichment | Field-level accuracy vs gold set | beat baseline by a clear margin | Days 35-36 |
| Enrichment | Structured-output parse success rate | >= 99% (with retry) | Day 34 |
| Search | NDCG@10 on labeled queries | hybrid > keyword; reranked > hybrid | Days 41-44 |
| Search | recall@k | track alongside NDCG | Day 41 |
| Search | p95 latency | < 800 ms (hypothesis) | Day 45 |
| Agent | Task success rate on scenario suite | to be set from baseline | Day 54 |
| Agent | Tool-selection accuracy | to be set from baseline | Day 54 |
| Safety | Adversarial suite pass rate | 100% on injection and cross-tenant cases | Day 55 |
| Reviews | Faithfulness of summaries | to be set from baseline | Day 57 |
| Support | Correct-escalation rate | to be set from baseline | Day 60 |
| Cost | Cost per search query, per chat session, per 1,000 enriched products | recorded and tracked | Days 32, 37, 62 |
| Reliability | Storefront availability with AI provider down | 100% of core flows work | Days 45, 66 |

## 7. Constraints and assumptions

- Solo builder, roughly 2-4 hours per day, 84 days.
- Stack is fixed in the project guide (FastAPI, Postgres + pgvector, Redis + Arq, Next.js, OpenAI behind our own wrapper).
- Models, pricing, and SDK syntax are checked against current official docs when used; model names live in env vars.

## 8. Risks

| Risk | Mitigation |
|---|---|
| Scope creep | Non-goals list; carry-over notes in `docs/progress.md` |
| LLM cost surprises | Per-call cost logging (Day 32), budgets (Day 62) |
| Prompt injection via reviews or chat | Guardrails and adversarial suite (Day 55) |
| Cross-tenant data leak | Row-level security plus tests (Days 10-11) |
| Provider outage | Fallbacks and circuit breaker (Days 45, 66) |