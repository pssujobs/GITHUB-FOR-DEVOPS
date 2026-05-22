# Kong AI Gateway — End-to-End Guide

> Kong AI Gateway is a high-performance connectivity and governance layer for AI-native applications. It sits between your clients and any LLM provider, enforcing authentication, routing, cost control, guardrails, and observability — without changing your application code.

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Concepts](#core-concepts)
4. [Prerequisites](#prerequisites)
5. [Quick Start](#quick-start)
6. [Supported LLM Providers](#supported-llm-providers)
7. [Capabilities Reference](#capabilities-reference)
8. [Recipes](#recipes)
   - [Recipe 1 — Basic LLM Routing](#recipe-1--basic-llm-routing)
   - [Recipe 2 — LLM Cost Optimization](#recipe-2--llm-cost-optimization)
   - [Recipe 3 — Claude Code SSO](#recipe-3--claude-code-sso)
   - [Recipe 4 — Secure External MCP Gateway](#recipe-4--secure-external-mcp-gateway)
   - [Recipe 5 — Secure Internal MCP Gateway](#recipe-5--secure-internal-mcp-gateway)
   - [Recipe 6 — Voice AI Pipeline Observability](#recipe-6--voice-ai-pipeline-observability)
9. [Plugin Reference](#plugin-reference)
10. [Observability & Monitoring](#observability--monitoring)
11. [Hybrid Mode: Separate Control Plane and Data Plane](#hybrid-mode-separate-control-plane-and-data-plane)
12. [AWS EKS Deployment with Kong Ingress Controller](#aws-eks-deployment-with-kong-ingress-controller)
13. [Deployment Checklist](#deployment-checklist)
14. [Troubleshooting & FAQ](#troubleshooting--faq)
15. [Further Reading](#further-reading)

---

## Overview

Modern AI applications are evolving from simple LLM calls into complex, multi-actor systems involving user apps, agents, orchestration layers, and context servers (MCP, A2A). Each component calls foundation models in real time.

Without a dedicated AI gateway layer, engineering teams face recurring pain points:

- **Credential sprawl** — every service embeds provider API keys; a single leaked key affects the entire org.
- **No client identity** — no way to attribute usage to a tenant, enforce per-user quotas, or revoke access for one app.
- **Provider lock-in** — each provider uses different auth (Bearer, x-api-key, SigV4) and different request formats; switching providers requires rewriting application code.
- **Cost unpredictability** — token-based rate limits don't map to dollars; no per-tier budget enforcement.
- **Zero observability** — no centralized audit trail, no trace spanning multiple AI hops.

Kong AI Gateway solves all of this with a single control plane that secures, governs, and observes AI-native systems end-to-end.

> Source: [Kong AI Gateway](https://developer.konghq.com/ai-gateway/)

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Client Applications                          │
│          (OpenAI SDK / Claude Code / Voice Agent / MCP Client)       │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  OpenAI-format request + API Key
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       Kong AI Gateway (Data Plane)                   │
│                                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │  Auth Layer │  │ Routing Layer│  │   Policy / Plugin Layer   │  │
│  │ ─────────── │  │ ──────────── │  │ ─────────────────────────  │  │
│  │ Key Auth    │  │ AI Proxy     │  │ Rate Limiting (token/cost) │  │
│  │ OIDC / JWT  │  │ AI Proxy Adv │  │ Semantic Cache            │  │
│  │ OAuth 2.1   │  │ Semantic     │  │ Prompt Guard / PII Strip  │  │
│  │ MCP OAuth2  │  │  Routing     │  │ Prompt Compressor         │  │
│  └─────────────┘  └──────────────┘  │ RAG Injector              │  │
│                                     │ Request/Response Transform │  │
│                                     │ OpenTelemetry              │  │
│                                     └───────────────────────────┘  │
└──────────────────────────┬───────────────────────────────────────────┘
                           │  Translated, authenticated, governed request
          ┌────────────────┼──────────────────────────────┐
          ▼                ▼                              ▼
   ┌────────────┐  ┌──────────────┐             ┌──────────────────┐
   │   OpenAI   │  │  Anthropic   │   . . .     │  AWS Bedrock /   │
   │ (GPT-4o)   │  │  (Claude 3.5)│             │  Azure / Gemini  │
   └────────────┘  └──────────────┘             └──────────────────┘

         ┌──────────────────────────────────────┐
         │       Konnect Control Plane          │
         │  (Config, Analytics, AI Manager UI)  │
         └──────────────────────────────────────┘

         ┌──────────────────────────────────────┐
         │    Observability Stack               │
         │  Langfuse / Datadog / Prometheus /   │
         │  Konnect Advanced Analytics          │
         └──────────────────────────────────────┘
```

| Component | Responsibility |
|-----------|---------------|
| **Client Application** | Sends OpenAI-format requests using the same SDK regardless of actual provider |
| **Auth Layer** | Validates per-Consumer credentials (API key, OIDC JWT, OAuth 2.1 token, MCP OAuth2) |
| **Routing Layer** | AI Proxy / AI Proxy Advanced — matches model alias, injects provider credentials, translates request/response format |
| **Policy Layer** | Enforces rate limits, caches semantically similar queries, compresses prompts, guards content, strips PII |
| **LLM Provider** | Receives the translated, authenticated request; returns native response |
| **Konnect Control Plane** | Manages gateway configuration, consumer groups, access tiers, and usage analytics |
| **Observability Stack** | Receives OpenTelemetry spans, token metrics, and audit logs |

> Source: [Kong AI Gateway](https://developer.konghq.com/ai-gateway/)

---

## Core Concepts

### Universal API
Kong's Universal API (delivered via the AI Proxy and AI Proxy Advanced plugins) provides a single, standardized OpenAI-compatible interface. Clients send OpenAI-format requests; Kong translates them to the target provider's native format, injects provider credentials, and translates responses back. Applications never hold a provider key and never change code when switching providers.

### Consumers & Consumer Groups
A **Consumer** is a named entity (e.g. an application or user) that authenticates to Kong. **Consumer Groups** (e.g. `free-tier`, `enterprise-tier`) let you apply different plugins — rate limits, model routing — to different tiers of clients. Consumers are identified by API keys, OIDC tokens, or OAuth 2.1 tokens.

### Routes & Services
A **Route** defines how incoming requests are matched (by path, method, host). A **Service** points at the upstream. For AI, you typically define one Service per LLM provider and attach the AI Proxy plugin to it, or use a single Service with AI Proxy Advanced for multi-provider routing.

### Plugins
Kong capabilities are delivered as **plugins** attached to Routes, Services, or globally. AI plugins understand the prompt body, not just HTTP headers — enabling prompt inspection, semantic routing, content moderation, and token-level rate limiting.

### Semantic Routing & Caching
AI Proxy Advanced can route requests to different models based on **semantic similarity** to pre-configured model descriptions, using a Redis vector database. The **AI Semantic Cache** plugin does the same for responses: if an incoming prompt is semantically close to a cached one, Kong returns the cached response without calling the LLM at all.

### Model Context Protocol (MCP)
MCP is a protocol for AI agents to discover and call "tools" (REST APIs, data sources, functions). Kong's **AI MCP Proxy** plugin converts your REST APIs into MCP tools and exposes them via a single MCP endpoint. The **AI MCP OAuth2** plugin enforces OAuth 2.1 at that endpoint, mapping token claims to per-tool ACLs via Consumer Groups.

### decK & kongctl
**decK** is a declarative configuration tool for Kong Gateway — you describe your desired state in YAML and decK syncs it. **kongctl** is a CLI for managing Konnect resources. Most recipes use both.

---

## Prerequisites

The following are required for any Kong AI Gateway recipe. Recipe-specific prerequisites are listed in each recipe section.

**Accounts**
- [Kong Konnect account](https://konghq.com/products/kong-konnect/register) (free tier available)
- A Konnect Personal Access Token (PAT) — create one at https://cloud.konghq.com/global/account/tokens

**Tools — install these first**

```bash
# Install decK (v1.43+)
# macOS
brew tap kong/deck && brew install deck

# Linux
curl -sL https://github.com/Kong/deck/releases/latest/download/deck_linux_amd64.tar.gz | tar -xz
sudo mv deck /usr/local/bin/

# Verify
deck version
```

```bash
# Install kongctl
# Visit https://developer.konghq.com/kongctl/ for platform-specific instructions
kongctl version
```

**Runtime**
- Docker (for local Data Plane and auxiliary services)
- Python 3.11+ (for demo scripts)

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install 'openai>=1.0.0'
```

**Export your Konnect token**
```bash
export KONNECT_TOKEN='YOUR_KONNECT_PAT'
```

**Minimum Kong Gateway version:** 3.14  
**Supported deployment modes:** Konnect (SaaS), self-hosted hybrid, DB-less (on-prem not supported for AI recipes)

---

## Quick Start

The fastest way to run Kong AI Gateway locally:

```bash
# 1. One-liner: spin up a Konnect-connected Data Plane with AI plugins
curl -Ls https://get.konghq.com/ai | bash

# -- OR -- provision a named Control Plane for a specific recipe:
export KONNECT_CONTROL_PLANE_NAME='my-ai-gateway'
curl -Ls https://get.konghq.com/quickstart | bash -s -- -k $KONNECT_TOKEN --deck-output

# 2. Export your OpenAI key
export DECK_OPENAI_TOKEN='Bearer sk-YOUR-KEY'

# 3. Apply a minimal AI Proxy config via decK (save to ai-proxy.yaml first)
deck gateway sync ai-proxy.yaml

# 4. Test it — OpenAI-format request through Kong
curl -X POST http://localhost:8000/ai/chat \
  -H "Content-Type: application/json" \
  -H "apikey: your-consumer-key" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello, Kong AI Gateway!"}]
  }'
```

> Source: [Kong AI Gateway Quickstart](https://developer.konghq.com/ai-gateway/get-started/)

---

## Supported LLM Providers

Kong AI Gateway routes requests to any of the following providers through its Universal API. Clients always send OpenAI-format requests; Kong translates credentials and request format automatically.

| Provider | Auth Method | Notes |
|----------|-------------|-------|
| OpenAI | `Authorization: Bearer sk-...` | GPT-4o, GPT-4o-mini, o1, etc. |
| Anthropic | `x-api-key: ...` | Claude 3.5, Claude 3 Haiku, etc. |
| AWS Bedrock | SigV4 request signing | Requires AWS credentials |
| Azure OpenAI | `api-key: ...` + instance endpoint | Instance-specific endpoint required |
| Google Gemini | OAuth2 token | Gemini 1.5 Pro, Flash, etc. |
| Mistral | `Authorization: Bearer ...` | Mistral 7B, Mixtral, etc. |
| Llama (self-hosted) | Configurable | Works with Ollama, vLLM, etc. |

> Source: [AI Gateway Providers](https://developer.konghq.com/ai-gateway/ai-providers/)

---

## Capabilities Reference

| Capability | Plugin(s) | What It Does |
|------------|-----------|-------------|
| **LLM Routing** | AI Proxy, AI Proxy Advanced | Route to any provider with credential injection and format translation |
| **Semantic Routing** | AI Proxy Advanced | Route requests to the cheapest adequate model based on semantic similarity |
| **Load Balancing** | AI Proxy Advanced | Round-robin, lowest-latency, consistent-hash, usage-based, semantic |
| **Retry & Fallback** | AI Proxy Advanced | Automatic retry on failure; fallback to alternate model |
| **Semantic Caching** | AI Semantic Cache | Return cached responses for semantically similar prompts (Redis-backed) |
| **Token Rate Limiting** | AI Rate Limiting Advanced | Enforce per-consumer token budgets by model, tier, or cost |
| **Prompt Compression** | AI Prompt Compressor | LLMLingua-powered prompt compression to reduce token counts 20-40% |
| **Prompt Guard** | AI Prompt Guard | Allow/deny list for prompt content |
| **Semantic Prompt Guard** | AI Semantic Prompt Guard | Topic-based allow/deny using semantic similarity |
| **PII Sanitization** | AI PII Sanitizer | Detects and redacts PII in 20 categories across 9 languages |
| **Content Safety** | AI Azure Content Safety, AI AWS Guardrails, AI GCP Model Armor, AI Lakera Guard | Third-party content moderation |
| **Prompt Decoration** | AI Prompt Decorator | Prepend/append system messages to user prompts |
| **Prompt Templates** | AI Prompt Template | Fill-in-the-blank prompt scaffolding for end users |
| **RAG Injection** | AI RAG Injector | Auto-inject vector DB content into prompts without app changes |
| **Request Transform** | AI Request Transformer | Use LLM to transform request body before upstream |
| **Response Transform** | AI Response Transformer | Use LLM to transform response body before client |
| **MCP Proxy** | AI MCP Proxy | Convert REST APIs into MCP tools; proxy external MCP servers |
| **MCP Auth** | AI MCP OAuth2 | OAuth 2.1 enforcement + per-tool ACLs on MCP endpoints |
| **Streaming** | Built-in | Server-Sent Events streaming for chat responses |
| **OpenTelemetry** | OpenTelemetry Plugin | Export `gen_ai.*` spans per request hop to any OTLP-compatible backend |
| **Audit Logging** | Built-in | Full request/response audit trail |
| **Secrets Management** | Konnect Config Store | Store LLM API keys in vaults; reference via `{vault://...}` |

---

## Recipes

All recipes require Kong Gateway 3.14+ and are designed for **Konnect** (SaaS). On-prem deployments are not supported for these AI cookbook recipes.

---

### Recipe 1 — Basic LLM Routing

**What it solves:** Route any client request to any LLM provider through a single endpoint, with per-app consumer authentication and per-request model selection — without embedding provider keys in application code.

**Plugins used:** Key Auth, AI Proxy Advanced

**Architecture:**
```
Client → POST /basic-llm-routing (apikey, model: "fast" | "smart")
  └─▶ Kong: key-auth validates apikey, attaches Consumer
      └─▶ Kong: ai-proxy-advanced matches model alias, injects provider auth
          └─▶ LLM Provider (OpenAI / Anthropic / etc.)
              └─▶ Kong: translates response to OpenAI format
                  └─▶ Client (+ X-Kong-LLM-Model header)
```

**Setup steps:**

```bash
# 1. Provision Control Plane and Data Plane
export KONNECT_CONTROL_PLANE_NAME='basic-llm-routing-recipe'
curl -Ls https://get.konghq.com/quickstart | bash -s -- -k $KONNECT_TOKEN --deck-output

# 2. Export your provider key
export DECK_OPENAI_TOKEN='Bearer sk-YOUR-KEY'

# 3. Apply configuration via decK
# (decK YAML defines: Service, Route /basic-llm-routing, key-auth plugin,
#  ai-proxy-advanced plugin with two model aliases: "fast" → gpt-4o-mini, "smart" → gpt-4o,
#  Consumer with API key credential)
deck gateway sync basic-llm-routing.yaml --konnect-token $KONNECT_TOKEN
```

**Test it:**
```bash
# Route to the "fast" model
curl -X POST http://localhost:8000/basic-llm-routing \
  -H "Content-Type: application/json" \
  -H "apikey: YOUR_CONSUMER_KEY" \
  -d '{"model": "fast", "messages": [{"role":"user","content":"What is 2+2?"}]}'

# Route to the "smart" model
curl -X POST http://localhost:8000/basic-llm-routing \
  -H "Content-Type: application/json" \
  -H "apikey: YOUR_CONSUMER_KEY" \
  -d '{"model": "smart", "messages": [{"role":"user","content":"Design a distributed database."}]}'
```

**Key benefits:**
- One endpoint, any provider, any model — zero application code changes to switch providers.
- Provider credentials never leave Kong — no key rotation across services.
- Per-Consumer API keys allow per-app revocation and quota enforcement.

> Source: [Basic LLM Routing Cookbook](https://developer.konghq.com/cookbooks/basic-llm-routing/)

**Full decK configuration — `basic-llm-routing.yaml`:**

```yaml
_format_version: '3.0'
_info:
  select_tags:
  - basic-llm-routing-recipe

services:
- name: basic-llm-routing
  url: http://localhost
  routes:
  - name: basic-llm-routing
    paths:
    - /basic-llm-routing
    protocols:
    - http
    - https
    methods:
    - POST
    - OPTIONS
    strip_path: true
  plugins:
  - name: key-auth
    instance_name: basic-llm-routing-auth
    config:
      key_names:
      - apikey
      hide_credentials: true
  - name: ai-proxy-advanced
    instance_name: basic-llm-routing-proxy
    config:
      max_request_body_size: 10485760
      response_streaming: allow
      targets:
      - route_type: llm/v1/chat
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        logging:
          log_statistics: true
          log_payloads: true
        model:
          model_alias: fast
          provider: openai
          name: ${{ env "DECK_CHAT_MODEL_1" }}
      - route_type: llm/v1/chat
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        logging:
          log_statistics: true
          log_payloads: true
        model:
          model_alias: smart
          provider: openai
          name: ${{ env "DECK_CHAT_MODEL_2" }}

consumers:
- username: demo-app
  keyauth_credentials:
  - key: demo-api-key
```

Apply:

```bash
export DECK_CHAT_MODEL_1='gpt-4o-mini'   # fast alias
export DECK_CHAT_MODEL_2='gpt-4o'        # smart alias
deck gateway sync basic-llm-routing.yaml \
  --konnect-token $KONNECT_TOKEN \
  --konnect-control-plane-name $KONNECT_CONTROL_PLANE_NAME
```

---

### Recipe 2 — LLM Cost Optimization

**What it solves:** Four stackable cost-reduction techniques applied at the gateway — semantic routing to cheapest adequate model, semantic caching to eliminate repeat LLM calls, prompt compression to reduce token counts, and dollar-based rate limiting per consumer tier.

**Plugins used:** Key Auth, AI Proxy Advanced, AI Semantic Cache, AI Prompt Compressor, AI Rate Limiting Advanced

**Architecture:**
```
Client → POST /llm-cost-optimization (apikey + prompt)
  └─▶ key-auth: identify Consumer → attach Consumer Group (free/enterprise)
  └─▶ ai-rate-limiting-advanced: check dollar budget for this Consumer Group
  └─▶ ai-prompt-compressor: compress prompt via LLMLingua (↓ 20-40% tokens)
  └─▶ ai-semantic-cache: check Redis for semantically similar cached response
      ├─▶ Cache HIT → return cached response (no LLM call, $0 cost)
      └─▶ Cache MISS →
          └─▶ ai-proxy-advanced: semantic routing → cheapest adequate model
              └─▶ LLM Provider → response cached in Redis for future hits
```

**Additional prerequisites:**
```bash
# Redis Stack (for semantic cache + semantic routing vector DB)
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack-server:latest
export DECK_REDIS_HOST='host.docker.internal'

# LLMLingua Prompt Compressor (contact Kong for registry credentials)
docker login docker.cloudsmith.io
docker run -d \
  --name llmlingua-compressor \
  -p 8080:8080 \
  -e LLMLINGUA_DEVICE_MAP=cpu \
  --memory=2g \
  docker.cloudsmith.io/kong/ai-compress/service:v0.0.3
export DECK_COMPRESSOR_URL='http://host.docker.internal:8080'

# Python deps for demo script
pip install 'openai>=1.0.0' 'tiktoken>=0.7.0'
```

**Setup:**
```bash
export KONNECT_CONTROL_PLANE_NAME='llm-cost-optimization-recipe'
curl -Ls https://get.konghq.com/quickstart | \
  bash -s -- -k $KONNECT_TOKEN \
    -e KONG_NGINX_WORKER_PROCESSES=1 \
    --deck-output

export DECK_OPENAI_TOKEN='Bearer sk-YOUR-KEY'
deck gateway sync llm-cost-optimization.yaml --konnect-token $KONNECT_TOKEN
```

**Cost control levers:**

| Technique | Savings mechanism | Kong plugin |
|-----------|------------------|-------------|
| Semantic routing | Routes cheap questions to a budget model (e.g. gpt-4o-mini instead of gpt-4o) | AI Proxy Advanced |
| Semantic caching | Eliminates duplicate LLM calls for near-identical prompts | AI Semantic Cache |
| Prompt compression | Reduces token count before billing by 20-40% | AI Prompt Compressor |
| Cost-based rate limiting | Enforces `$` budgets per consumer tier per hour | AI Rate Limiting Advanced |

> Source: [LLM Cost Optimization Cookbook](https://developer.konghq.com/cookbooks/llm-cost-optimization/)

**Full decK configuration — `llm-cost-optimization.yaml`:**

```yaml
_format_version: '3.0'
_info:
  select_tags:
  - llm-cost-optimization-recipe

services:
- name: llm-cost-optimization
  url: http://localhost
  routes:
  - name: llm-cost-optimization
    paths:
    - /llm-cost-optimization
    protocols: [http, https]
    methods: [POST, OPTIONS]
    strip_path: true
  plugins:
  - name: key-auth
    instance_name: llm-cost-optimization-auth
    config:
      key_names: [apikey]
      hide_credentials: true

  # 1. Semantic Cache — returns cached response for near-identical prompts
  - name: ai-semantic-cache
    instance_name: llm-cost-optimization-cache
    config:
      cache_ttl: 300
      message_countback: 1
      stop_on_failure: false
      embeddings:
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        model:
          provider: openai
          name: ${{ env "DECK_EMBEDDINGS_MODEL" }}
      vectordb:
        dimensions: ${{ env "DECK_EMBEDDINGS_DIMENSIONS" }}
        distance_metric: cosine
        strategy: redis
        threshold: 0.3
        redis:
          host: ${{ env "DECK_REDIS_HOST" }}
          port: 6379

  # 2. Prompt Compressor — reduces token count 20-40% before billing
  - name: ai-prompt-compressor
    instance_name: llm-cost-optimization-compressor
    ordering:
      before:
        access: [ai-proxy-advanced]
    config:
      compressor_type: rate
      compressor_url: ${{ env "DECK_COMPRESSOR_URL" }}
      stop_on_error: false
      compression_ranges:
      - min_tokens: 100
        max_tokens: 1000000
        value: 0.6

  # 3. AI Proxy Advanced — semantic routing to cheapest adequate model
  - name: ai-proxy-advanced
    instance_name: llm-cost-optimization-proxy
    config:
      max_request_body_size: 10485760
      response_streaming: allow
      balancer:
        algorithm: semantic
      embeddings:
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        model:
          provider: openai
          name: ${{ env "DECK_EMBEDDINGS_MODEL" }}
      vectordb:
        dimensions: ${{ env "DECK_EMBEDDINGS_DIMENSIONS" }}
        distance_metric: cosine
        strategy: redis
        threshold: ${{ env "DECK_PROXY_THRESHOLD" }}
        redis:
          host: ${{ env "DECK_REDIS_HOST" }}
          port: 6379
      targets:
      - route_type: llm/v1/chat
        description: CATCHALL
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        model:
          model_alias: auto
          provider: openai
          name: ${{ env "DECK_CHAT_MODEL_1" }}
          options:
            input_cost: ${{ env "DECK_INPUT_COST_1" }}
            output_cost: ${{ env "DECK_OUTPUT_COST_1" }}
        logging:
          log_payloads: true
          log_statistics: true
      - route_type: llm/v1/chat
        description: Software architecture, system design, and complex code generation.
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        model:
          model_alias: auto
          provider: openai
          name: ${{ env "DECK_CHAT_MODEL_2" }}
          options:
            input_cost: ${{ env "DECK_INPUT_COST_2" }}
            output_cost: ${{ env "DECK_OUTPUT_COST_2" }}
        logging:
          log_payloads: true
          log_statistics: true

consumers:
- username: standard-user
  keyauth_credentials:
  - key: standard-api-key
- username: premium-user
  keyauth_credentials:
  - key: premium-api-key

consumer_groups:
- name: standard-tier
  consumers:
  - username: standard-user
  plugins:
  # 4. Cost-based Rate Limiting — $1/hour for standard tier
  - name: ai-rate-limiting-advanced
    config:
      tokens_count_strategy: cost
      llm_providers:
      - name: openai
        limit: [1]        # $1 per hour
        window_size: [3600]
      window_type: sliding
      strategy: local

- name: premium-tier
  consumers:
  - username: premium-user
  plugins:
  - name: ai-rate-limiting-advanced
    config:
      tokens_count_strategy: cost
      llm_providers:
      - name: openai
        limit: [5]        # $5 per hour
        window_size: [3600]
      window_type: sliding
      strategy: local
```

Apply:

```bash
export DECK_CHAT_MODEL_1='gpt-4o-mini'
export DECK_CHAT_MODEL_2='gpt-4o'
export DECK_EMBEDDINGS_MODEL='text-embedding-3-small'
export DECK_EMBEDDINGS_DIMENSIONS='1536'
export DECK_REDIS_HOST='host.docker.internal'
export DECK_COMPRESSOR_URL='http://host.docker.internal:8080'
export DECK_PROXY_THRESHOLD='0.8'
export DECK_INPUT_COST_1='0.00000015'
export DECK_OUTPUT_COST_1='0.0000006'
export DECK_INPUT_COST_2='0.0000025'
export DECK_OUTPUT_COST_2='0.00001'
deck gateway sync llm-cost-optimization.yaml \
  --konnect-token $KONNECT_TOKEN \
  --konnect-control-plane-name $KONNECT_CONTROL_PLANE_NAME
```

---

### Recipe 3 — Claude Code SSO

**What it solves:** Put Kong AI Gateway in front of Anthropic/AWS Bedrock so every Claude Code request from your engineering team is authenticated via Okta SSO, routed to a model tier by Okta group membership, and subject to per-tier token rate limits. No provider credentials sit on developer machines.

**Plugins used:** OpenID Connect, AI Proxy Advanced, AI Rate Limiting Advanced

**Architecture:**
```
Developer (Claude Code) → POST /claude-code (Okta JWT)
  └─▶ openid-connect: validate JWT against Okta JWKS endpoint
      └─▶ map Okta group claim → Kong Consumer Group
          ├─▶ claude-standard-users → ai-proxy-advanced → claude-3-5-haiku (budget model)
          │     └─▶ ai-rate-limiting-advanced: 100k tokens/hour
          └─▶ claude-power-users → ai-proxy-advanced → claude-3-5-sonnet (premium model)
                └─▶ ai-rate-limiting-advanced: 500k tokens/hour
```

**Okta setup (key steps):**
1. Create a **Native Application** in Okta (public client, PKCE enabled).
2. Set redirect URI to `http://localhost:9876/callback`.
3. Enable **Refresh Token** grant type.
4. Add a `groups` claim to your authorization server (Claims tab → Value type: Groups).
5. Create groups `claude-standard-users` and `claude-power-users`; assign test users.

**Setup:**
```bash
export KONNECT_CONTROL_PLANE_NAME='claude-code-sso-recipe'
curl -Ls https://get.konghq.com/quickstart | \
  bash -s -- -k $KONNECT_TOKEN \
    -e KONG_NGINX_HTTP_CLIENT_BODY_BUFFER_SIZE=16m \
    -e KONG_NGINX_HTTP_CLIENT_MAX_BODY_SIZE=16m \
    --deck-output

# Export Anthropic key
export DECK_ANTHROPIC_TOKEN='YOUR_ANTHROPIC_API_KEY'
# Export Okta OIDC values
export DECK_OKTA_ISSUER='https://your-org.okta.com/oauth2/default'
export DECK_OKTA_CLIENT_ID='0oa...'

deck gateway sync claude-code-sso.yaml --konnect-token $KONNECT_TOKEN
```

**Configure Claude Code to use Kong:**
```bash
# In Claude Code settings or environment:
export ANTHROPIC_BASE_URL='http://localhost:8000/claude-code'
export ANTHROPIC_AUTH_TOKEN='<okta-access-token-from-helper-script>'
```

**Key benefits:**
- Zero provider credentials on developer machines.
- Model access controlled by Okta group membership — onboarding/offboarding in one place.
- Per-tier token budgets enforced at the gateway without app changes.

> Source: [Claude Code SSO Cookbook](https://developer.konghq.com/cookbooks/claude-code-sso/)

#### `claude-code-sso.yaml` — Full decK Configuration

```yaml
# =============================================================================
# Claude Code SSO — decK Configuration
# =============================================================================
# Apply with:
#   export KONNECT_TOKEN='YOUR_PAT'
#   export KONNECT_CONTROL_PLANE_NAME='claude-code-sso-recipe'
#   export DECK_ANTHROPIC_TOKEN='YOUR_ANTHROPIC_API_KEY'
#   export DECK_OKTA_ISSUER='https://your-org.okta.com/oauth2/default'
#   export DECK_OKTA_AUDIENCE='api://default'
#   export DECK_OIDC_CACHE_TOKENS_SALT='$(openssl rand -hex 16)'
#   export DECK_CHAT_MODEL_1='claude-3-5-haiku-20241022'
#   export DECK_CHAT_MODEL_2='claude-3-5-sonnet-20241022'
#   export DECK_SONNET_ALIAS='claude-sonnet-4-5'
#   export DECK_OPUS_ALIAS='claude-opus-4-5'
#   export DECK_ANTHROPIC_VERSION='2023-06-01'
#
#   deck gateway sync claude-code-sso.yaml \
#     --konnect-token $KONNECT_TOKEN \
#     --konnect-control-plane-name $KONNECT_CONTROL_PLANE_NAME
#
# Configure Claude Code to use this gateway:
#   export ANTHROPIC_BASE_URL='http://localhost:8000/claude-code-sso'
#   export ANTHROPIC_AUTH_TOKEN='<okta-access-token>'
# =============================================================================

_format_version: '3.0'
_info:
  select_tags:
  - claude-code-sso-recipe

consumer_groups:
- name: claude-standard-users
- name: claude-power-users

services:
- name: claude-code-sso
  url: http://localhost
  routes:
  - name: claude-code-sso
    paths:
    - /claude-code-sso
    protocols: [http, https]
    methods: [POST, OPTIONS]
    strip_path: true
  plugins:
  # OIDC: validate Okta JWT and map 'groups' claim → Kong Consumer Groups
  - name: openid-connect
    instance_name: claude-code-sso-oidc
    config:
      issuer: ${{ env "DECK_OKTA_ISSUER" }}
      auth_methods: [bearer]
      bearer_token_param_type: [header]
      audience_claim: [aud]
      audience_required:
      - ${{ env "DECK_OKTA_AUDIENCE" }}
      consumer_groups_claim: [groups]
      consumer_optional: true
      consumer_groups_optional: false
      upstream_headers_claims: [sub, email]
      upstream_headers_names: [X-Authenticated-User, X-User-Email]
      ssl_verify: true
      hide_credentials: true
      cache_tokens_salt: ${{ env "DECK_OIDC_CACHE_TOKENS_SALT" }}

plugins:
# Standard tier: Haiku model, 20k tokens/min
- name: ai-proxy-advanced
  instance_name: claude-code-sso-standard-proxy
  service: claude-code-sso
  consumer_group: claude-standard-users
  config:
    llm_format: anthropic
    max_request_body_size: 10485760
    response_streaming: allow
    targets:
    - route_type: llm/v1/chat
      auth:
        header_name: x-api-key
        header_value: ${{ env "DECK_ANTHROPIC_TOKEN" }}
      logging:
        log_statistics: true
        log_payloads: true
      model:
        model_alias: ${{ env "DECK_SONNET_ALIAS" }}
        provider: anthropic
        name: ${{ env "DECK_CHAT_MODEL_1" }}
        options:
          anthropic_version: ${{ env "DECK_ANTHROPIC_VERSION" }}

# Power tier: Haiku + Opus models, 100k tokens/min
- name: ai-proxy-advanced
  instance_name: claude-code-sso-power-proxy
  service: claude-code-sso
  consumer_group: claude-power-users
  config:
    llm_format: anthropic
    max_request_body_size: 10485760
    response_streaming: allow
    targets:
    - route_type: llm/v1/chat
      auth:
        header_name: x-api-key
        header_value: ${{ env "DECK_ANTHROPIC_TOKEN" }}
      logging:
        log_statistics: true
        log_payloads: true
      model:
        model_alias: ${{ env "DECK_SONNET_ALIAS" }}
        provider: anthropic
        name: ${{ env "DECK_CHAT_MODEL_1" }}
        options:
          anthropic_version: ${{ env "DECK_ANTHROPIC_VERSION" }}
    - route_type: llm/v1/chat
      auth:
        header_name: x-api-key
        header_value: ${{ env "DECK_ANTHROPIC_TOKEN" }}
      logging:
        log_statistics: true
        log_payloads: true
      model:
        model_alias: ${{ env "DECK_OPUS_ALIAS" }}
        provider: anthropic
        name: ${{ env "DECK_CHAT_MODEL_2" }}
        options:
          anthropic_version: ${{ env "DECK_ANTHROPIC_VERSION" }}

# Standard tier rate limit: 20,000 tokens per minute
- name: ai-rate-limiting-advanced
  instance_name: claude-code-sso-standard-ratelimit
  service: claude-code-sso
  consumer_group: claude-standard-users
  config:
    policies:
    - limits:
      - limit: 20000
        window_size: 60
      window_type: sliding
    identifier: consumer-group
    tokens_count_strategy: total_tokens
    strategy: local
    llm_format: anthropic

# Power tier rate limit: 100,000 tokens per minute
- name: ai-rate-limiting-advanced
  instance_name: claude-code-sso-power-ratelimit
  service: claude-code-sso
  consumer_group: claude-power-users
  config:
    policies:
    - limits:
      - limit: 100000
        window_size: 60
      window_type: sliding
    identifier: consumer-group
    tokens_count_strategy: total_tokens
    strategy: local
    llm_format: anthropic
```

---

### Recipe 4 — Secure External MCP Gateway

**What it solves:** Organizations connecting AI agents to third-party MCP servers (GitHub, Slack, Figma, Konnect) end up with credential sprawl and no audit trail. Putting Kong in front provides a single boundary for ACL enforcement, observability, and credential management.

**Plugins used:** AI MCP Proxy, AI MCP OAuth2, Datakit, Key Auth, Response/Request Transformer Advanced, CORS

**Two patterns for external MCP:**

```
Pattern A — Passthrough (e.g. GitHub MCP)
  MCP Client → Kong /github-mcp (Kong API Key)
    └─▶ key-auth: identify Consumer → apply ACLs
        └─▶ ai-mcp-proxy: pass user's GitHub OAuth token through
            └─▶ GitHub MCP Server (user's own GitHub token)

Pattern B — Token Swap (e.g. Konnect MCP)
  MCP Client → Kong /konnect-mcp (Okta JWT)
    └─▶ ai-mcp-oauth2: validate Okta token → map to Consumer
        └─▶ ai-mcp-proxy: swap user token for stored Konnect Service Account Token
            └─▶ Konnect MCP Server (service account token)
```

**Setup:**
```bash
export KONNECT_CONTROL_PLANE_NAME='secure-external-mcp-gateway-recipe'
export DECK_KONNECT_MCP_TOKEN="${KONNECT_TOKEN}"  # use SAT in production
curl -Ls https://get.konghq.com/quickstart | bash -s -- -k $KONNECT_TOKEN --deck-output

# GitHub OAuth App: register at github.com/settings/developers
# Set callback URL to your MCP client's OAuth redirect URL
# Export GitHub OAuth App Client ID and Secret (used by MCP client at runtime)

deck gateway sync secure-external-mcp-gateway.yaml --konnect-token $KONNECT_TOKEN
```

**Security notes:**
- In production, replace PATs with **System Account Tokens** (least-privilege, not tied to individual users).
- Store secrets in a [Kong Vault](https://developer.konghq.com/gateway/secrets-management/) and reference them via `{vault://backend/key}`.
- GitHub OAuth Apps support only one callback URL — create a separate app per MCP client type.

> Source: [Secure External MCP Gateway Cookbook](https://developer.konghq.com/cookbooks/secure-external-mcp-gateway/)

#### `secure-external-mcp-gateway.yaml` — Full decK Configuration

```yaml
# =============================================================================
# Secure External MCP Gateway — decK Configuration
# =============================================================================
# Two patterns demonstrated:
#   Pattern A — Passthrough (GitHub MCP): Kong adds key-auth + tool-level ACLs;
#               user's own GitHub OAuth token passes through to GitHub MCP server.
#   Pattern B — Token Swap (Konnect MCP): Kong validates user's Okta/IdP token,
#               then swaps it for a stored Konnect Service Account Token upstream.
#
# Apply with:
#   export KONNECT_TOKEN='YOUR_PAT'
#   export KONNECT_CONTROL_PLANE_NAME='secure-external-mcp-gateway-recipe'
#   export DECK_KONNECT_MCP_TOKEN="${KONNECT_TOKEN}"   # use SAT in production
#   export DECK_KONNECT_MCP_HOST='us.api.konghq.com'
#   export DECK_OAUTH_AUTH_SERVER='https://your-org.okta.com/oauth2/default'
#   export DECK_OAUTH_INTROSPECTION_URL='https://your-org.okta.com/oauth2/default/v1/introspect'
#   export DECK_OAUTH_CLIENT_ID='0oa...'
#   export DECK_OAUTH_CLIENT_SECRET='YOUR_CLIENT_SECRET'
#
#   deck gateway sync secure-external-mcp-gateway.yaml \
#     --konnect-token $KONNECT_TOKEN \
#     --konnect-control-plane-name $KONNECT_CONTROL_PLANE_NAME
# =============================================================================

_format_version: '3.0'
_info:
  select_tags:
  - secure-external-mcp-gateway-recipe

consumer_groups:
- name: admin
- name: developer

consumers:
- username: admin-user
  keyauth_credentials:
  - key: admin-api-key
  groups:
  - name: admin
- username: dev-user
  keyauth_credentials:
  - key: dev-api-key
  groups:
  - name: developer

services:
# Pattern A — GitHub MCP (Passthrough)
# User authenticates directly with GitHub; Kong adds key-auth + tool ACLs
- name: github-mcp-service
  host: api.githubcopilot.com
  port: 443
  protocol: https
  path: /mcp
  routes:
  - name: github-mcp
    paths:
    - /github-mcp
    protocols: [http, https]
    strip_path: true
    request_buffering: false
    response_buffering: false
    plugins:
    # Datakit: challenge unauthenticated requests before key-auth runs
    - name: datakit
      instance_name: github-mcp-prm-challenge
      tags: [secure-external-mcp-gateway-recipe]
      ordering:
        before:
          access: [ai-mcp-proxy, key-auth]
      config:
        nodes:
        - name: CHECK_AUTH
          type: jq
          inputs:
            headers: request.headers
          jq: .headers.Authorization? != null or .headers.authorization? != null
        - name: AUTH_BRANCH
          type: branch
          input: CHECK_AUTH
          else: [UNAUTH_RESPONSE, UNAUTH_EXIT]
        - name: UNAUTH_RESPONSE
          type: static
          values:
            content-type: application/json
            www-authenticate: >-
              Bearer
              resource_metadata="https://localhost:8443/.well-known/oauth-protected-resource/github-mcp",
              error="invalid_token"
            BODY: '{"message": "Unauthorized"}'
        - name: UNAUTH_EXIT
          type: exit
          status: 401
          inputs:
            body: UNAUTH_RESPONSE.BODY
            headers: UNAUTH_RESPONSE

    - name: key-auth
      instance_name: github-mcp-key-auth
      tags: [secure-external-mcp-gateway-recipe]
      config:
        key_names: [apikey]
        hide_credentials: true

    - name: ai-mcp-proxy
      instance_name: github-mcp-listener
      tags: [secure-external-mcp-gateway-recipe]
      config:
        mode: passthrough-listener
        consumer_identifier: username
        include_consumer_groups: true
        default_acl:
        - deny:
          - developer       # developers cannot use write tools by default
          scope: tools
        logging:
          log_payloads: true
          log_statistics: true
        tools:
        # Read-only tools — both groups
        - name: search_repositories
          description: Search GitHub repositories
          acl:
            allow: [admin, developer]
        - name: get_file_contents
          description: Get file contents from a repository
          acl:
            allow: [admin, developer]
        - name: list_issues
          description: List issues in a repository
          acl:
            allow: [admin, developer]
        - name: list_pull_requests
          description: List pull requests in a repository
          acl:
            allow: [admin, developer]
        - name: search_users
          description: Search GitHub users
          acl:
            allow: [admin, developer]
        - name: search_code
          description: Search code across repositories
          acl:
            allow: [admin, developer]
        # Write tools — admin only
        - name: create_issue
          description: Create an issue in a repository
          acl:
            allow: [admin]
        - name: push_files
          description: Push files to a repository
          acl:
            allow: [admin]
        - name: create_pull_request
          description: Create a pull request
          acl:
            allow: [admin]
        - name: merge_pull_request
          description: Merge a pull request
          acl:
            allow: [admin]
        - name: create_repository
          description: Create a new repository
          acl:
            allow: [admin]

    - name: cors
      instance_name: github-mcp-cors
      tags: [secure-external-mcp-gateway-recipe]
      config:
        origins: ['*']
        methods: [GET, HEAD, PUT, PATCH, POST, DELETE, OPTIONS]
        credentials: false
        preflight_continue: false

# Protected Resource Metadata endpoint for GitHub MCP (RFC 9728)
- name: github-mcp-prm-service
  host: api.githubcopilot.com
  port: 443
  protocol: https
  path: /.well-known/oauth-protected-resource/mcp
  routes:
  - name: github-mcp-prm
    paths:
    - /.well-known/oauth-protected-resource/github-mcp
    protocols: [http, https]
    methods: [GET, POST]
    strip_path: true
    preserve_host: false
    plugins:
    - name: response-transformer-advanced
      instance_name: github-mcp-prm-rewrite
      tags: [secure-external-mcp-gateway-recipe]
      config:
        replace:
          json:
          - 'resource:https://localhost:8443/github-mcp'
          json_types: [string]

# Pattern B — Konnect MCP (Token Swap)
# Kong validates user's Okta token, then swaps for stored Konnect SAT
- name: konnect-mcp-service
  host: ${{ env "DECK_KONNECT_MCP_HOST" }}
  port: 443
  protocol: https
  routes:
  - name: konnect-mcp
    paths:
    - /konnect-mcp
    - /.well-known/oauth-protected-resource/konnect-mcp
    protocols: [http, https]
    request_buffering: false
    response_buffering: false
    plugins:
    - name: ai-mcp-oauth2
      instance_name: konnect-mcp-oauth2
      tags: [secure-external-mcp-gateway-recipe]
      config:
        resource: https://localhost:8443/konnect-mcp
        authorization_servers:
        - ${{ env "DECK_OAUTH_AUTH_SERVER" }}
        introspection_endpoint: ${{ env "DECK_OAUTH_INTROSPECTION_URL" }}
        client_id: ${{ env "DECK_OAUTH_CLIENT_ID" }}
        client_secret: ${{ env "DECK_OAUTH_CLIENT_SECRET" }}
        metadata_endpoint: /.well-known/oauth-protected-resource/konnect-mcp
        scopes_supported: [openid]
        insecure_relaxed_audience_validation: true
        ssl_verify: true
        cache_introspection: true
        passthrough_credentials: false    # swap token, not pass through
        consumer_claim: [sub]
        consumer_by: [username, custom_id]
        consumer_optional: true
        consumer_groups_claim: [groups]
        consumer_groups_optional: true
```

---

### Recipe 5 — Secure Internal MCP Gateway

**What it solves:** Multiple API teams each exposing tools through their own MCP servers, with no central control over discovery or access. Kong collapses them into a single aggregated MCP endpoint with one place to enforce OAuth 2.1 and per-tool ACLs.

**Plugins used:** AI MCP Proxy, AI MCP OAuth2, CORS

**Architecture:**
```
MCP Client (Insomnia / Claude Code / VS Code)
  └─▶ OAuth 2.1 Authorization Code + PKCE with your IdP (Okta / Keycloak)
      └─▶ POST /ecommerce-mcp (access token)
          └─▶ ai-mcp-oauth2: validate token → extract groups claim
              └─▶ map groups → Kong Consumer Groups (read-only / full-access)
                  └─▶ ai-mcp-proxy: per-tool ACL enforcement
                      ├─▶ GET /products   (available to all groups)
                      ├─▶ GET /orders     (available to all groups)
                      └─▶ POST /fulfilment (restricted to full-access only)
```

**Okta setup (key steps):**
1. Create a **Kong Gateway application** (API Services type) — this is Kong's introspection client.
2. Create an **MCP Client application** (SPA type, PKCE) — this is the developer tool.
3. Add a `groups` claim to the authorization server; set value type to Groups.
4. Create groups (e.g. `mcp-read-only`, `mcp-full-access`); assign test users.
5. Export introspection credentials:
```bash
export DECK_MCP_RESOURCE_URL='http://localhost:8000/ecommerce-mcp'
export DECK_OAUTH_CLIENT_ID='0oa...'
export DECK_OAUTH_CLIENT_SECRET='YOUR_CLIENT_SECRET'
export DECK_OAUTH_ISSUER='https://your-org.okta.com/oauth2/default'
```

**Setup:**
```bash
export KONNECT_CONTROL_PLANE_NAME='secure-internal-mcp-gateway-recipe'
curl -Ls https://get.konghq.com/quickstart | bash -s -- -k $KONNECT_TOKEN --deck-output
deck gateway sync secure-internal-mcp-gateway.yaml --konnect-token $KONNECT_TOKEN
```

**Two modes for internal MCP servers:**
- **REST-to-MCP conversion** (no MCP server needed): AI MCP Proxy reads your OpenAPI spec and auto-generates MCP tools from your REST endpoints.
- **Passthrough mode**: AI MCP Proxy proxies an existing standalone MCP server your team already runs.

> Source: [Secure Internal MCP Gateway Cookbook](https://developer.konghq.com/cookbooks/secure-internal-mcp-gateway/)

#### `secure-internal-mcp-gateway.yaml` — Full decK Configuration

```yaml
# =============================================================================
# Secure Internal MCP Gateway — decK Configuration
# =============================================================================
# Aggregates 3 internal ecommerce APIs (orders, inventory, customers) into a
# single MCP endpoint with OAuth 2.1 authentication and per-tool ACLs.
#
# Two Consumer Groups:
#   warehouse-ops    → can read + restock inventory, read orders/customers
#   customer-support → can read orders + customers, cancel orders
#
# Apply with:
#   export KONNECT_TOKEN='YOUR_PAT'
#   export KONNECT_CONTROL_PLANE_NAME='secure-internal-mcp-gateway-recipe'
#   export DECK_MCP_RESOURCE_URL='http://localhost:8000/ecommerce-mcp'
#   export DECK_OAUTH_CLIENT_ID='0oa...'
#   export DECK_OAUTH_CLIENT_SECRET='YOUR_SECRET'
#   export DECK_OAUTH_ISSUER='https://your-org.okta.com/oauth2/default'
#   export DECK_OAUTH_INTROSPECTION_URL='https://your-org.okta.com/oauth2/default/v1/introspect'
#
#   deck gateway sync secure-internal-mcp-gateway.yaml \
#     --konnect-token $KONNECT_TOKEN \
#     --konnect-control-plane-name $KONNECT_CONTROL_PLANE_NAME
# =============================================================================

_format_version: '3.0'
_info:
  select_tags:
  - secure-internal-mcp-gateway-recipe

consumer_groups:
- name: warehouse-ops
- name: customer-support

services:
# Orders API → MCP tools: list-orders, get-order, cancel-order
- name: orders-api
  host: httpbin.konghq.com   # replace with your real orders API host
  port: 443
  protocol: https
  path: /anything
  routes:
  - name: orders-mcp
    paths:
    - /orders-mcp
    protocols: [http, https]
    plugins:
    - name: ai-mcp-proxy
      instance_name: orders-mcp-conversion
      tags: [secure-internal-mcp-gateway-recipe, ecom-mcp]
      config:
        mode: conversion-only
        consumer_identifier: username
        include_consumer_groups: true
        max_request_body_size: 1048576
        logging:
          log_payloads: true
          log_statistics: true
        tools:
        - name: list-orders
          description: List recent orders with optional filters for status and date.
          method: GET
          path: /orders
          annotations:
            read_only_hint: true
            title: List orders
          acl:
            allow: [warehouse-ops, customer-support]
          parameters:
          - name: status
            in: query
            required: false
            description: Filter by order status (pending, shipped, delivered, cancelled)
            schema:
              type: string
          - name: date
            in: query
            required: false
            description: Filter by order date (YYYY-MM-DD)
            schema:
              type: string

        - name: get-order
          description: Get full details for a specific order including line items and shipping.
          method: GET
          path: /orders/{orderId}
          annotations:
            read_only_hint: true
            title: Get order details
          acl:
            allow: [warehouse-ops, customer-support]
          parameters:
          - name: orderId
            in: path
            required: true
            description: The order ID
            schema:
              type: string

        - name: cancel-order
          description: Cancel a pending order. This action cannot be undone.
          method: POST
          path: /orders/{orderId}/cancel
          annotations:
            destructive_hint: true
            title: Cancel order
          acl:
            allow: [customer-support]    # warehouse-ops cannot cancel
          parameters:
          - name: orderId
            in: path
            required: true
            description: The order ID to cancel
            schema:
              type: string

# Inventory API → MCP tools: list-inventory, check-inventory, restock-item
- name: inventory-api
  host: httpbin.konghq.com
  port: 443
  protocol: https
  path: /anything
  routes:
  - name: inventory-mcp
    paths:
    - /inventory-mcp
    protocols: [http, https]
    plugins:
    - name: ai-mcp-proxy
      instance_name: inventory-mcp-conversion
      tags: [secure-internal-mcp-gateway-recipe, ecom-mcp]
      config:
        mode: conversion-only
        consumer_identifier: username
        include_consumer_groups: true
        max_request_body_size: 1048576
        logging:
          log_payloads: true
          log_statistics: true
        tools:
        - name: list-inventory
          description: List inventory items with optional filters for category or low stock.
          method: GET
          path: /inventory
          annotations:
            read_only_hint: true
            title: List inventory
          acl:
            allow: [warehouse-ops, customer-support]
          parameters:
          - name: category
            in: query
            required: false
            description: Filter by product category
            schema:
              type: string
          - name: low_stock
            in: query
            required: false
            description: Set to true to show only items below restock threshold
            schema:
              type: string

        - name: check-inventory
          description: Check current stock level and details for a specific SKU.
          method: GET
          path: /inventory/{sku}
          annotations:
            read_only_hint: true
            title: Check inventory for SKU
          acl:
            allow: [warehouse-ops, customer-support]
          parameters:
          - name: sku
            in: path
            required: true
            description: The product SKU
            schema:
              type: string

        - name: restock-item
          description: Add units to a product's inventory.
          method: POST
          path: /inventory/{sku}/restock
          annotations:
            destructive_hint: true
            title: Restock inventory item
          acl:
            allow: [warehouse-ops]       # customer-support cannot restock
          parameters:
          - name: sku
            in: path
            required: true
            description: The product SKU to restock
            schema:
              type: string
          - name: quantity
            in: query
            required: true
            description: Number of units to add
            schema:
              type: integer

# Customers API → MCP tools: get-customer, list-customers
- name: customers-api
  host: httpbin.konghq.com
  port: 443
  protocol: https
  path: /anything
  routes:
  - name: customers-mcp
    paths:
    - /customers-mcp
    protocols: [http, https]
    plugins:
    - name: ai-mcp-proxy
      instance_name: customers-mcp-conversion
      tags: [secure-internal-mcp-gateway-recipe, ecom-mcp]
      config:
        mode: conversion-only
        consumer_identifier: username
        include_consumer_groups: true
        max_request_body_size: 1048576
        logging:
          log_payloads: true
          log_statistics: true
        tools:
        - name: get-customer
          description: Retrieve customer profile and contact information.
          method: GET
          path: /customers/{customerId}
          annotations:
            read_only_hint: true
            title: Get customer profile
          acl:
            allow: [warehouse-ops, customer-support]
          parameters:
          - name: customerId
            in: path
            required: true
            description: The customer ID
            schema:
              type: string

        - name: list-customers
          description: List customers with optional search filters.
          method: GET
          path: /customers
          annotations:
            read_only_hint: true
            title: List customers
          acl:
            allow: [customer-support]
          parameters:
          - name: search
            in: query
            required: false
            description: Search by name or email
            schema:
              type: string

# Global plugins — OAuth 2.1 + CORS applied to all ecom-mcp routes
plugins:
- name: ai-mcp-oauth2
  instance_name: ecommerce-mcp-oauth2
  tags: [secure-internal-mcp-gateway-recipe, ecom-mcp]
  config:
    resource: ${{ env "DECK_MCP_RESOURCE_URL" }}
    authorization_servers:
    - ${{ env "DECK_OAUTH_ISSUER" }}
    introspection_endpoint: ${{ env "DECK_OAUTH_INTROSPECTION_URL" }}
    client_id: ${{ env "DECK_OAUTH_CLIENT_ID" }}
    client_secret: ${{ env "DECK_OAUTH_CLIENT_SECRET" }}
    scopes_supported: [openid, profile, email]
    consumer_groups_claim: [groups]
    consumer_groups_optional: false
    ssl_verify: true
    cache_introspection: true

- name: cors
  instance_name: ecommerce-mcp-cors
  tags: [secure-internal-mcp-gateway-recipe, ecom-mcp]
  config:
    origins: ['*']
    methods: [GET, HEAD, PUT, PATCH, POST, DELETE, OPTIONS]
    credentials: false
    preflight_continue: false
```

---

### Recipe 6 — Voice AI Pipeline Observability

**What it solves:** A production voice AI system runs STT → LLM → TTS in sequence for every conversational turn. Each hop has its own latency budget, error modes, and cost profile. This recipe governs all three hops through Kong and exports per-hop OpenTelemetry traces to Langfuse for full-turn analysis.

**Plugins used:** AI Proxy Advanced, OpenTelemetry, Key Auth

**Architecture:**
```
Voice Agent (Python client)
  │
  ├─▶ POST /stt  (apikey, audio file) → AI Proxy Advanced → OpenAI Whisper (STT)
  │     └─▶ OpenTelemetry: gen_ai.* span (latency, tokens, model)
  │
  ├─▶ POST /llm  (apikey, transcript)  → AI Proxy Advanced → LLM (GPT-4o / Claude / etc.)
  │     └─▶ OpenTelemetry: gen_ai.* span (latency, input/output tokens, cost)
  │
  └─▶ POST /tts  (apikey, LLM response) → AI Proxy Advanced → OpenAI TTS-1
        └─▶ OpenTelemetry: gen_ai.* span (latency, character count)

All spans share conversation-level trace ID → appear as single trace in Langfuse
```

**Setup:**
```bash
# Tracing must be enabled at Data Plane startup
export KONNECT_CONTROL_PLANE_NAME='voice-ai-observability-recipe'
curl -Ls https://get.konghq.com/quickstart | \
  bash -s -- -k $KONNECT_TOKEN \
    -e KONG_TRACING_INSTRUMENTATIONS=all \
    -e KONG_TRACING_SAMPLING_RATE=1.0 \
    --deck-output

export DECK_OPENAI_TOKEN='Bearer sk-YOUR-KEY'
```

**Langfuse setup:**
```bash
export DECK_LANGFUSE_PUBLIC_KEY='pk-lf-YOUR-PUBLIC-KEY'
export DECK_LANGFUSE_SECRET_KEY='sk-lf-YOUR-SECRET-KEY'

# Pick your region endpoint:
# US:    https://us.cloud.langfuse.com/api/public/otel/v1/traces
# EU:    https://cloud.langfuse.com/api/public/otel/v1/traces
# HIPAA: https://hipaa.cloud.langfuse.com/api/public/otel/v1/traces
export DECK_LANGFUSE_OTLP_ENDPOINT='https://us.cloud.langfuse.com/api/public/otel/v1/traces'

export DECK_LANGFUSE_AUTH_HEADER="Basic $(printf '%s:%s' \
  "$DECK_LANGFUSE_PUBLIC_KEY" "$DECK_LANGFUSE_SECRET_KEY" | base64 | tr -d '\n')"

deck gateway sync voice-ai-observability.yaml --konnect-token $KONNECT_TOKEN
```

**Python client (minimal):**
```python
from openai import OpenAI
import os

client = OpenAI(base_url="http://localhost:8000", api_key="ignored")
headers = {"apikey": os.environ["VOICE_AGENT_KEY"]}

# Hop 1: STT
with open("audio.wav", "rb") as f:
    transcript = client.audio.transcriptions.create(
        model="whisper-1", file=f, extra_headers=headers
    )

# Hop 2: LLM
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": transcript.text}],
    extra_headers=headers
)

# Hop 3: TTS
speech = client.audio.speech.create(
    model="tts-1", voice="alloy",
    input=response.choices[0].message.content,
    extra_headers=headers
)
speech.stream_to_file("response.mp3")
```

**Key insight:** Set `KONG_TRACING_SAMPLING_RATE=1.0` at Data Plane startup — the default rate of `0.01` drops 99% of spans before the OpenTelemetry plugin sees them.

> Source: [Voice AI Pipeline Observability Cookbook](https://developer.konghq.com/cookbooks/voice-ai-observability/)

#### `voice-ai-observability.yaml` — Full decK Configuration

```yaml
# =============================================================================
# Voice AI Pipeline Observability — decK Configuration
# =============================================================================
# Three Kong routes — one per pipeline hop:
#   /voice-ai-observability/stt  → OpenAI Whisper (Speech-to-Text)
#   /voice-ai-observability/llm  → Any LLM (GPT-4o, Claude, etc.)
#   /voice-ai-observability/tts  → OpenAI TTS-1 (Text-to-Speech)
#
# Apply with:
#   export KONNECT_TOKEN='YOUR_PAT'
#   export KONNECT_CONTROL_PLANE_NAME='voice-ai-observability-recipe'
#   export DECK_OPENAI_TOKEN='Bearer sk-YOUR-KEY'
#   export DECK_CHAT_MODEL='gpt-4o'
#   export DECK_LANGFUSE_PUBLIC_KEY='pk-lf-...'
#   export DECK_LANGFUSE_SECRET_KEY='sk-lf-...'
#   export DECK_LANGFUSE_OTLP_ENDPOINT='https://us.cloud.langfuse.com/api/public/otel/v1/traces'
#   export DECK_LANGFUSE_AUTH_HEADER="Basic $(printf '%s:%s' \
#     "$DECK_LANGFUSE_PUBLIC_KEY" "$DECK_LANGFUSE_SECRET_KEY" | base64 | tr -d '\n')"
#
# IMPORTANT — tracing must be enabled at Data Plane startup:
#   curl -Ls https://get.konghq.com/quickstart | \
#     bash -s -- -k $KONNECT_TOKEN \
#       -e KONG_TRACING_INSTRUMENTATIONS=all \
#       -e KONG_TRACING_SAMPLING_RATE=1.0 \
#       --deck-output
#   (Default sampling rate 0.01 drops 99% of spans — always set to 1.0 for dev)
#
#   deck gateway sync voice-ai-observability.yaml \
#     --konnect-token $KONNECT_TOKEN \
#     --konnect-control-plane-name $KONNECT_CONTROL_PLANE_NAME
# =============================================================================

_format_version: '3.0'
_info:
  select_tags:
  - voice-ai-observability-recipe

services:
# Hop 1 — Speech-to-Text (STT)
- name: voice-ai-stt
  url: http://localhost
  routes:
  - name: voice-ai-stt
    paths:
    - /voice-ai-observability/stt
    protocols: [http, https]
    methods: [POST, OPTIONS]
    strip_path: true
  plugins:
  - name: ai-proxy-advanced
    instance_name: voice-ai-stt-proxy
    config:
      genai_category: audio/transcription
      max_request_body_size: 26214400    # 25MB for audio files
      response_streaming: deny
      targets:
      - route_type: audio/v1/audio/transcriptions
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        logging:
          log_payloads: true
        model:
          provider: openai
          name: whisper-1

# Hop 2 — LLM Reasoning
- name: voice-ai-llm
  url: http://localhost
  routes:
  - name: voice-ai-llm
    paths:
    - /voice-ai-observability/llm
    protocols: [http, https]
    methods: [POST, OPTIONS]
    strip_path: true
  plugins:
  - name: ai-proxy-advanced
    instance_name: voice-ai-llm-proxy
    config:
      max_request_body_size: 8388608
      response_streaming: allow
      targets:
      - route_type: llm/v1/chat
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        logging:
          log_statistics: true
          log_payloads: true
        model:
          provider: openai
          name: ${{ env "DECK_CHAT_MODEL" }}

# Hop 3 — Text-to-Speech (TTS)
- name: voice-ai-tts
  url: http://localhost
  routes:
  - name: voice-ai-tts
    paths:
    - /voice-ai-observability/tts
    protocols: [http, https]
    methods: [POST, OPTIONS]
    strip_path: true
  plugins:
  - name: ai-proxy-advanced
    instance_name: voice-ai-tts-proxy
    config:
      genai_category: audio/speech
      max_request_body_size: 1048576
      response_streaming: allow
      targets:
      - route_type: audio/v1/audio/speech
        auth:
          header_name: Authorization
          header_value: ${{ env "DECK_OPENAI_TOKEN" }}
        logging:
          log_payloads: true
        model:
          provider: openai
          name: tts-1

# Global plugins — applied to all three hops
plugins:
- name: key-auth
  instance_name: voice-ai-observability-auth
  config:
    key_names: [apikey]
    hide_credentials: true

- name: opentelemetry
  instance_name: voice-ai-observability-otel
  config:
    traces_endpoint: ${{ env "DECK_LANGFUSE_OTLP_ENDPOINT" }}
    headers:
      Authorization: ${{ env "DECK_LANGFUSE_AUTH_HEADER" }}
      x-langfuse-ingestion-version: '4'
    sampling_rate: 1      # 1 = 100% — required for dev; tune for production
    propagation:
      default_format: w3c  # W3C traceparent header links hops into one trace

consumers:
- username: voice-agent
  keyauth_credentials:
  - key: voice-demo-key
```

---

## Plugin Reference

| Plugin | Category | Purpose |
|--------|----------|---------|
| AI Proxy | Routing | Single-provider LLM proxy with format translation |
| AI Proxy Advanced | Routing | Multi-provider, load balancing, semantic routing, retry/fallback |
| Key Auth | Auth | API key authentication for Consumers |
| OpenID Connect | Auth | JWT/OIDC validation against any IdP (Okta, Keycloak, Entra, etc.) |
| AI MCP OAuth2 | Auth | OAuth 2.1 enforcement for MCP endpoints with group-based ACLs |
| AI MCP Proxy | MCP | REST-to-MCP conversion; external MCP server proxy |
| AI Semantic Cache | Cost | Redis-backed embedding-based response cache |
| AI Prompt Compressor | Cost | LLMLingua-powered prompt token reduction |
| AI Rate Limiting Advanced | Governance | Token/cost-based rate limits per Consumer Group |
| AI Prompt Guard | Safety | Allow/deny prompt content by string match |
| AI Semantic Prompt Guard | Safety | Topic-based allow/deny by semantic similarity |
| AI Semantic Response Guard | Safety | Block responses similar to known unsafe outputs |
| AI PII Sanitizer | Safety | Auto-redact PII across 20 categories and 9 languages |
| AI Azure Content Safety | Safety | Azure AI moderation on prompts and responses |
| AI AWS Guardrails | Safety | AWS Bedrock Guardrails integration |
| AI GCP Model Armor | Safety | Google Cloud Model Armor integration |
| AI Lakera Guard | Safety | Lakera Guard policy enforcement |
| AI Prompt Decorator | Prompts | Prepend/append system messages to chat history |
| AI Prompt Template | Prompts | Fill-in-the-blank prompt templates for end users |
| AI RAG Injector | Context | Auto-inject vector DB content into prompts |
| AI Request Transformer | Transform | Use LLM to transform request body before upstream |
| AI Response Transformer | Transform | Use LLM to transform response body before client |
| OpenTelemetry | Observability | Export `gen_ai.*` spans to any OTLP-compatible backend |
| Canary | Traffic | Gradual model rollout to a subset of users |

> Source: [All AI Plugins](https://developer.konghq.com/plugins/?category=ai)

---

## Observability & Monitoring

### OpenTelemetry (per-request spans)
Kong exports `gen_ai.*` OTLP spans for every LLM, STT, TTS, and MCP request. Spans include: model name, provider, input/output token counts, latency, and status.

```yaml
# OpenTelemetry plugin config
- name: opentelemetry
  config:
    endpoint: "https://us.cloud.langfuse.com/api/public/otel/v1/traces"
    headers:
      Authorization: "{vault://env/langfuse-auth-header}"
    resource_attributes:
      service.name: "kong-ai-gateway"
```

**Important:** Enable tracing at Data Plane startup:
```bash
-e KONG_TRACING_INSTRUMENTATIONS=all
-e KONG_TRACING_SAMPLING_RATE=1.0
```

### Konnect Advanced Analytics
The [Konnect AI Manager](https://cloud.konghq.com/ai-manager) provides pre-built dashboards showing:
- Request and token volumes per model
- Error rates and average latency
- Historical comparisons
- Visual traffic maps (real-time request flows between clients and models)

### Audit Logging
Kong logs full request/response bodies (with optional PII stripping) to standard logging backends. See [AI Audit Log Reference](https://developer.konghq.com/ai-gateway/ai-audit-log-reference/).

### LLM Metrics (Prometheus/Grafana)
Kong exposes LLM metrics at `/metrics` in Prometheus format. Key metrics include token usage, request counts by model, latency percentiles, and cache hit rates. See [Monitor AI LLM Metrics](https://developer.konghq.com/ai-gateway/monitor-ai-llm-metrics/).

### Metering & Billing
Konnect can meter LLM token usage across models and prompt types for accurate billing. Create pricing plans based on input/output/system token consumption and automate invoicing with Stripe or ERP integrations. See [Metering & Billing](https://developer.konghq.com/metering-and-billing/).

---

## Hybrid Mode: Separate Control Plane and Data Plane

### Why Hybrid Mode?

In classic Kong deployments every node connects to the same database. Hybrid Mode decouples the **Control Plane (CP)** — which owns configuration and the database — from the **Data Plane (DP)**, which serves traffic with no database connection at all.

| Aspect | Classic Mode | Hybrid Mode |
|--------|-------------|-------------|
| Database access | Every node | CP only |
| Config delivery | All nodes poll DB | CP pushes to DP over mTLS |
| DP availability during CP outage | ✗ (DB required) | ✓ (LMDB cache on disk) |
| Deployment flexibility | Single topology | CP central, DPs at edge/cloud |
| DB traffic on DP nodes | Yes | None |

**Key benefits:**
- **Reliability** — each DP caches the last known config in `dbless.lmdb`; it keeps serving requests even if CP is unreachable for hours.
- **Security** — DB credentials never exist on DP nodes. A compromised DP cannot reach the database.
- **Edge deployments** — DPs run in remote regions or on-prem; one central CP manages them all.
- **Reduced DB load** — only CP nodes read/write Postgres.

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│                   Control Plane (CP)                      │
│                                                           │
│  ┌───────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │  Kong CP  │←→ │  PostgreSQL  │   │  Admin API     │  │
│  │  process  │   │  Database    │   │  :8001 / :8444 │  │
│  └───────────┘   └──────────────┘   └────────────────┘  │
│        │                                                  │
│   Port 8005 (cluster_listen) — mTLS config push          │
└────────┼─────────────────────────────────────────────────┘
         │  mTLS  (cluster_cert / cluster_cert_key)
┌────────▼─────────────────────────────────────────────────┐
│                   Data Plane (DP)                         │
│                                                           │
│  ┌───────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │  Kong DP  │   │  LMDB cache  │   │  Proxy         │  │
│  │  process  │   │ dbless.lmdb  │   │  :8000 / :8443 │  │
│  └───────────┘   └──────────────┘   └────────────────┘  │
│  No DB connection — config received from CP only          │
└──────────────────────────────────────────────────────────┘
```

### Key Configuration Parameters

**Control Plane `kong.conf` / environment variables:**
```bash
KONG_ROLE=control_plane
KONG_CLUSTER_LISTEN=0.0.0.0:8005          # DPs connect here
KONG_CLUSTER_TELEMETRY_LISTEN=0.0.0.0:8006
KONG_DATABASE=postgres
KONG_PG_HOST=your-postgres-host
KONG_CLUSTER_MTLS=shared                  # or 'pki' for PKI-based mTLS
KONG_CLUSTER_CERT=/path/to/cluster.crt
KONG_CLUSTER_CERT_KEY=/path/to/cluster.key
```

**Data Plane `kong.conf` / environment variables:**
```bash
KONG_ROLE=data_plane
KONG_DATABASE=off                         # DP is always DB-less
KONG_CLUSTER_CONTROL_PLANE=cp-host:8005  # CP address
KONG_CLUSTER_TELEMETRY_ENDPOINT=cp-host:8006
KONG_CLUSTER_MTLS=shared
KONG_CLUSTER_CERT=/path/to/cluster.crt
KONG_CLUSTER_CERT_KEY=/path/to/cluster.key
KONG_LUA_SSL_TRUSTED_CERTIFICATE=/path/to/cluster.crt
# AI Gateway observability — required for gen_ai.* spans
KONG_TRACING_INSTRUMENTATIONS=all
KONG_TRACING_SAMPLING_RATE=1.0
```

### Step 1 — Generate the Cluster Certificate

For `shared` mTLS both CP and DP use the same certificate:

```bash
openssl req -new -x509 -nodes \
  -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout cluster.key -out cluster.crt \
  -days 1095 -subj "/CN=kong_clustering"
```

For **PKI** mode (`KONG_CLUSTER_MTLS=pki`), your internal CA issues separate certs for CP and DP. Set `KONG_CLUSTER_CA_CERT` on the CP to the CA cert that signed all DP certs.

### Step 2 — Start Control Plane (Docker)

```bash
# Create shared Docker network
docker network create kong-net

# Start Postgres
docker run -d --name kong-database --network kong-net \
  -e POSTGRES_USER=kong -e POSTGRES_PASSWORD=kongpass -e POSTGRES_DB=kong \
  postgres:15

# Bootstrap DB
docker run --rm --network kong-net \
  -e KONG_DATABASE=postgres -e KONG_PG_HOST=kong-database \
  -e KONG_PG_USER=kong -e KONG_PG_PASSWORD=kongpass \
  kong/kong-gateway:latest kong migrations bootstrap

# Start CP
docker run -d --name kong-cp --network kong-net \
  -e KONG_ROLE=control_plane \
  -e KONG_DATABASE=postgres \
  -e KONG_PG_HOST=kong-database \
  -e KONG_PG_USER=kong \
  -e KONG_PG_PASSWORD=kongpass \
  -e KONG_CLUSTER_LISTEN=0.0.0.0:8005 \
  -e KONG_CLUSTER_TELEMETRY_LISTEN=0.0.0.0:8006 \
  -e KONG_CLUSTER_MTLS=shared \
  -e KONG_CLUSTER_CERT=/certs/cluster.crt \
  -e KONG_CLUSTER_CERT_KEY=/certs/cluster.key \
  -e KONG_ADMIN_LISTEN=0.0.0.0:8001 \
  -e KONG_ADMIN_GUI_LISTEN=0.0.0.0:8002 \
  -v $(pwd)/certs:/certs \
  -p 8001:8001 -p 8002:8002 -p 8005:8005 -p 8006:8006 \
  kong/kong-gateway:latest
```

### Step 3 — Start Data Plane (Docker)

```bash
docker run -d --name kong-dp --network kong-net \
  -e KONG_ROLE=data_plane \
  -e KONG_DATABASE=off \
  -e KONG_CLUSTER_CONTROL_PLANE=kong-cp:8005 \
  -e KONG_CLUSTER_TELEMETRY_ENDPOINT=kong-cp:8006 \
  -e KONG_CLUSTER_MTLS=shared \
  -e KONG_CLUSTER_CERT=/certs/cluster.crt \
  -e KONG_CLUSTER_CERT_KEY=/certs/cluster.key \
  -e KONG_LUA_SSL_TRUSTED_CERTIFICATE=/certs/cluster.crt \
  -e KONG_TRACING_INSTRUMENTATIONS=all \
  -e KONG_TRACING_SAMPLING_RATE=1.0 \
  -v $(pwd)/certs:/certs \
  -p 8000:8000 -p 8443:8443 \
  kong/kong-gateway:latest

# Verify DP connected to CP
curl -s http://localhost:8001/clustering/data-planes | \
  jq '.data[] | {ip, version, config_hash, last_seen}'
```

### Step 4 — Apply AI Gateway Config

Once CP and DP are running, use decK to apply any recipe:

```bash
export KONNECT_TOKEN='YOUR_PAT'
# OR target a self-managed CP directly:
export DECK_KONG_ADDR='http://localhost:8001'

deck gateway sync claude-code-sso.yaml --kong-addr $DECK_KONG_ADDR
# Config is pushed from CP to DP automatically within seconds
```

### Konnect (SaaS Control Plane)

With Konnect, Kong manages the CP for you. You only run Data Planes:

```bash
# 1. Create a Hybrid CP in Konnect
CP_RESPONSE=$(curl -s -X POST https://us.api.konghq.com/v2/control-planes \
  -H "Authorization: Bearer $KONNECT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-ai-gateway",
    "cluster_type": "CLUSTER_TYPE_HYBRID",
    "auth_type": "pinned_client_certs"
  }')
CP_ID=$(echo $CP_RESPONSE | jq -r .id)
CP_ENDPOINT=$(echo $CP_RESPONSE | jq -r '.config.control_plane_endpoint' | sed 's|https://||')

# 2. Generate DP certificate
openssl req -new -x509 -nodes \
  -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout dp.key -out dp.crt -days 365 -subj "/CN=dp-node"

# 3. Upload certificate to Konnect
curl -s -X POST https://us.api.konghq.com/v2/control-planes/$CP_ID/dp-client-certificates \
  -H "Authorization: Bearer $KONNECT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"cert\":\"$(awk 'NF {sub(/\r/,""); printf "%s\\n",$0}' dp.crt)\"}"

# 4. Start DP connecting to Konnect
docker run -d --name kong-dp \
  -e KONG_ROLE=data_plane \
  -e KONG_DATABASE=off \
  -e KONG_CLUSTER_CONTROL_PLANE=${CP_ENDPOINT}:443 \
  -e KONG_CLUSTER_TELEMETRY_ENDPOINT=$(echo $CP_ENDPOINT | sed 's/cp0/tp0'):443 \
  -e KONG_CLUSTER_MTLS=pki \
  -e KONG_CLUSTER_CERT=/certs/dp.crt \
  -e KONG_CLUSTER_CERT_KEY=/certs/dp.key \
  -e KONG_CLUSTER_SERVER_NAME=$CP_ENDPOINT \
  -e KONG_LUA_SSL_TRUSTED_CERTIFICATE=system \
  -e KONG_TRACING_INSTRUMENTATIONS=all \
  -e KONG_TRACING_SAMPLING_RATE=1.0 \
  -v $(pwd)/certs:/certs \
  -p 8000:8000 -p 8443:8443 \
  kong/kong-gateway:latest
```

### Version Compatibility Rules

| Rule | Detail |
|------|--------|
| Same major version required | CP 3.x cannot manage DP 2.x |
| DP minor ≤ CP minor | CP 3.9 can manage DPs at 3.7, 3.8, or 3.9 |
| Never DP > CP | Running a newer DP than CP is unsupported |

```bash
# Check CP version
curl -s http://localhost:8001/ | jq .version

# Check all connected DPs
curl -s http://localhost:8001/clustering/data-planes | \
  jq '.data[] | {id, version, config_hash}'
```

### Plugin Behavior in Hybrid / DB-less Mode

| Plugin / Feature | Classic (DB) | Hybrid DP (DB-less) |
|-----------------|-------------|---------------------|
| AI Proxy Advanced | Full support | Full support |
| AI Rate Limiting Advanced — `strategy: cluster` | ✓ shared counter | ✗ not supported |
| AI Rate Limiting Advanced — `strategy: redis` | ✓ | ✓ recommended |
| AI Rate Limiting Advanced — `strategy: local` | ✓ | ✓ (per-DP counter) |
| AI Semantic Cache (Redis) | ✓ | ✓ |
| OpenID Connect | ✓ | ✓ |
| AI MCP Proxy / AI MCP OAuth2 | ✓ | ✓ |
| Legacy OAuth 2.0 plugin | ✓ (DB-backed tokens) | ✗ |
| Session plugin | DB-backed | Requires Redis storage |

For AI Gateway production deployments, use `strategy: redis` for `ai-rate-limiting-advanced` to get shared counters across all DP nodes without needing a CP/DB connection.

---

## AWS EKS Deployment with Kong Ingress Controller

### Architecture Overview

Kong Ingress Controller (KIC) runs inside your EKS cluster and translates Kubernetes-native resources (Ingress, HTTPRoute, KongPlugin CRDs) into Kong Gateway config. No separate Admin API calls needed — everything is declarative via `kubectl`.

```
┌─────────────────────────────────────────────────────────────┐
│                      AWS EKS Cluster                         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                   kong namespace                      │   │
│  │                                                       │   │
│  │  ┌─────────────────────┐   ┌──────────────────────┐  │   │
│  │  │  KIC (controller)   │──▶│  Kong Gateway (proxy) │  │   │
│  │  │  Watches K8s API    │   │  :80 / :443           │  │   │
│  │  │  Converts → config  │   │  LoadBalancer SVC     │  │   │
│  │  └─────────────────────┘   └──────────┬───────────┘  │   │
│  │                                       │               │   │
│  └───────────────────────────────────────┼───────────────┘   │
└──────────────────────────────────────────┼─────────────────── ┘
                                           │
                               ┌───────────▼──────────┐
                               │   AWS NLB / ALB       │
                               └───────────┬───────────┘
                                           │
                                     AI Clients
```

When paired with Konnect, KIC syncs config up to the SaaS Control Plane:

```
KIC (in EKS) ──mTLS──▶ Konnect CP ──▶ AI Manager / Analytics
```

### Prerequisites

```bash
# Required tools
brew install helm kubectl awscli

# Configure EKS cluster access
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Verify
kubectl get nodes
```

### Step 1 — Install Kong via Helm

```bash
# Add Kong chart repo
helm repo add kong https://charts.konghq.com
helm repo update

# Basic install (local CP, no Konnect)
helm install kong kong/ingress \
  --namespace kong \
  --create-namespace \
  --wait

# Verify
kubectl get pods -n kong
# kong-controller-xxx   1/1  Running
# kong-gateway-xxx      1/1  Running

# Get external endpoint
kubectl get svc -n kong kong-gateway-proxy
```

### Step 2 — Install Gateway API CRDs

Gateway API gives finer-grained control than legacy Ingress:

```bash
# Install standard Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# GatewayClass — tells KIC to manage gateways of this class
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: kong
  annotations:
    konghq.com/gatewayclass-unmanaged: 'true'
spec:
  controllerName: konghq.com/kic-gateway-controller
EOF

# Gateway — defines the proxy listener
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kong
  namespace: kong
spec:
  gatewayClassName: kong
  listeners:
  - name: proxy
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
EOF
```

### Step 3 — Connect KIC to Konnect (SaaS CP)

```bash
# Create a KIC-type Control Plane in Konnect
CP_RESPONSE=$(curl -s -X POST https://us.api.konghq.com/v2/control-planes \
  -H "Authorization: Bearer $KONNECT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "eks-ai-gateway",
    "cluster_type": "CLUSTER_TYPE_K8S_INGRESS_CONTROLLER",
    "auth_type": "pinned_client_certs"
  }')
CP_ID=$(echo $CP_RESPONSE | jq -r .id)

# Generate mTLS cert for KIC
openssl req -new -x509 -nodes \
  -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout tls.key -out tls.crt \
  -days 365 -subj "/CN=kic-eks"

# Upload cert to Konnect
curl -s -X POST https://us.api.konghq.com/v2/control-planes/$CP_ID/dp-client-certificates \
  -H "Authorization: Bearer $KONNECT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"cert\":\"$(awk 'NF {sub(/\r/,""); printf "%s\\n",$0}' tls.crt)\"}"

# Store cert as Kubernetes Secret
kubectl create secret tls konnect-client-tls \
  --cert=tls.crt --key=tls.key \
  --namespace kong
```

### Step 4 — Helm values for Konnect integration

Create `values-konnect.yaml`:

```yaml
# values-konnect.yaml
controller:
  ingressController:
    enabled: true
    konnect:
      enabled: true
      runtimeGroupID: "<CP_ID from Step 3>"
      apiHostname: "us.api.konghq.com"
      tlsClientCertificateSecretName: konnect-client-tls

gateway:
  enabled: true
  env:
    # Required for AI Gateway observability
    kong_tracing_instrumentations: "all"
    kong_tracing_sampling_rate: "1.0"

  # Expose via AWS NLB
  proxy:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"

  # Admin API internal only
  admin:
    enabled: true
    type: ClusterIP
```

```bash
helm upgrade --install kong kong/ingress \
  --namespace kong \
  --create-namespace \
  --values values-konnect.yaml \
  --wait

# Confirm KIC connected to Konnect
kubectl logs -n kong -l app.kubernetes.io/component=controller | grep -i konnect
```

### Step 5 — Deploy an AI Gateway Route

With KIC, AI Gateway plugins attach to routes via CRD annotations:

```yaml
# ai-llm-route.yaml
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-openai
  namespace: default
plugin: ai-proxy-advanced
config:
  response_streaming: allow
  targets:
  - route_type: llm/v1/chat
    auth:
      header_name: Authorization
      header_value: "Bearer sk-YOUR-OPENAI-KEY"  # use a K8s secret in production
    model:
      provider: openai
      name: gpt-4o

---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: key-auth
  namespace: default
plugin: key-auth
config:
  key_names: [apikey]

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ai-llm-route
  namespace: default
  annotations:
    konghq.com/plugins: "key-auth,ai-proxy-openai"
spec:
  parentRefs:
  - name: kong
    namespace: kong
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /llm
    backendRefs:
    - name: any-placeholder-svc   # ai-proxy-advanced rewrites the upstream
      port: 80
```

```bash
kubectl apply -f ai-llm-route.yaml

# Test
PROXY_IP=$(kubectl get svc -n kong kong-gateway-proxy \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

curl -s http://$PROXY_IP/llm/v1/chat/completions \
  -H "apikey: my-test-key" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello from EKS!"}]}' | jq .
```

### Step 6 — Apply decK Recipes on EKS

The same `.yaml` recipe files work identically on EKS — just point decK at the Konnect CP:

```bash
export KONNECT_TOKEN='YOUR_PAT'

deck gateway sync claude-code-sso.yaml \
  --konnect-token $KONNECT_TOKEN \
  --konnect-control-plane-name eks-ai-gateway

deck gateway sync voice-ai-observability.yaml \
  --konnect-token $KONNECT_TOKEN \
  --konnect-control-plane-name eks-ai-gateway
```

### EKS-Specific Recommendations

| Topic | Recommendation |
|-------|---------------|
| **Load Balancer** | Use AWS NLB (`type: nlb`) for TCP passthrough — preserves client IPs and supports HTTP/2 |
| **TLS termination** | Terminate TLS at Kong (`:8443`) for end-to-end mTLS with AI providers |
| **Secrets** | Store API keys in AWS Secrets Manager; reference via Kong Vault: `{vault://aws/my-secret/key}` |
| **Scaling DPs** | Kong DPs are stateless — use HPA on CPU/RPS; KIC controller is a singleton (leader-elected) |
| **Node groups** | Dedicated node group with `m6i.xlarge` or larger for production AI workloads |
| **Multi-tenancy** | Use `--watch-namespace` on KIC to restrict config to specific namespaces |
| **Upgrades** | Upgrade Konnect CP first, then rolling-update the KIC + gateway Helm chart |
| **Rate limits** | Use `strategy: redis` for `ai-rate-limiting-advanced` to share counters across all pods |

---

## Deployment Checklist

**Security**
- [ ] Replace all demo keys (`demo-api-key`, `voice-demo-key`) with secrets stored in a vault
- [ ] Enable `ssl_verify: true` on all OIDC and OAuth plugins
- [ ] Use System Account Tokens (SATs) instead of Personal Access Tokens in CI pipelines
- [ ] Rotate `DECK_OIDC_CACHE_TOKENS_SALT` and store in vault (not in env vars on DP nodes)

**Rate Limiting & Cost Control**
- [ ] Set per-Consumer-Group token budgets via `ai-rate-limiting-advanced`
- [ ] Use `strategy: redis` for shared counters across multiple DP nodes
- [ ] Verify `tokens_count_strategy: total_tokens` to capture both input and output tokens
- [ ] Set `cost_strategy` if you want dollar-based limits

**Observability**
- [ ] Set `KONG_TRACING_SAMPLING_RATE=1.0` on all DPs (default `0.01` silently drops 99% of spans)
- [ ] Set `KONG_TRACING_INSTRUMENTATIONS=all` on all DPs
- [ ] Confirm Langfuse / OTLP endpoint receives traces after first request
- [ ] Enable `log_statistics: true` and `log_payloads: true` on AI plugin targets

**Hybrid Mode**
- [ ] Verify CP-DP version compatibility (same major, DP minor ≤ CP minor)
- [ ] Use `strategy: local` or `strategy: redis` — never `cluster` — for rate limits on DB-less DPs
- [ ] Test DP fault tolerance: stop the CP, confirm DP continues serving from LMDB cache
- [ ] Monitor CP-DP sync lag: `curl http://localhost:8001/clustering/data-planes | jq .[].config_hash`

**CI/CD**
- [ ] Store all `.yaml` recipe files in version control
- [ ] Run `deck gateway diff` before `deck gateway sync` in CI
- [ ] Use `select_tags` to scope syncs to a specific recipe (avoid touching unrelated resources)
- [ ] Tag all resources with environment: `env:production`, `env:staging`

---

## Troubleshooting & FAQ

**Q: My decK sync fails with "entity already exists"**
```bash
# Scope sync to only your recipe's resources using select_tags
deck gateway sync myrecipe.yaml \
  --select-tag myrecipe-tag \
  --konnect-token $KONNECT_TOKEN \
  --konnect-control-plane-name my-cp
```

**Q: 401 Unauthorized even with a valid API key**
- Confirm the Consumer's `keyauth_credentials.key` exactly matches the value in the `apikey` header
- Ensure `hide_credentials: true` is set — the key is stripped before forwarding upstream
- Run `curl -v` and check the raw `apikey` header is present

**Q: Langfuse shows no traces despite the OpenTelemetry plugin being configured**
- Verify the Data Plane started with `KONG_TRACING_INSTRUMENTATIONS=all` **and** `KONG_TRACING_SAMPLING_RATE=1.0`
- The default sampling rate is `0.01` — this drops 99% of spans at the DP level before the plugin fires
- Confirm `DECK_LANGFUSE_AUTH_HEADER` is correctly base64-encoded: `Basic <base64(publicKey:secretKey)>`

**Q: Rate limits are not shared across multiple Kong DP nodes**
- Switch from `strategy: local` to `strategy: redis` and configure Redis host/port
- `strategy: cluster` is only valid in Classic (non-hybrid) mode with a shared Postgres database

**Q: The OIDC plugin returns "consumer not found"**
- Set `consumer_optional: true` if you don't pre-create Consumers for every IdP user
- Only set `consumer_groups_optional: false` if every access token includes a `groups` claim
- Check your Okta authorization server includes `groups` in the Access Token (not just ID Token)

**Q: AI Semantic Cache never returns cached responses**
- Lower `similarity_threshold` — default `0.3`; try `0.1` for more aggressive caching
- Confirm Redis is reachable from Kong DP: `redis-cli -h $REDIS_HOST ping`
- Ensure the same embeddings model and dimension are configured in cache and LLM proxy

**Q: KIC on EKS is not picking up my KongPlugin annotation**
- Verify the KongPlugin resource is in the **same namespace** as the HTTPRoute
- Check KIC logs: `kubectl logs -n kong -l app.kubernetes.io/component=controller | grep error`
- Confirm `konghq.com/plugins` annotation references the exact `metadata.name` of the KongPlugin

**Q: DP lost connection to CP — is traffic affected?**
- No. The DP continues serving requests using its LMDB cache (last known config)
- Config updates made on CP will not propagate until the connection is restored
- Check CP health: `curl http://cp-host:8001/status`; check DP logs for reconnection attempts

**Q: How does AI Gateway differ from a standard API gateway?**
Standard Kong operates at the HTTP level. AI Gateway plugins understand the **prompt body** — they can inspect, cache, compress, and rate-limit at the semantic/token level. You cannot do semantic routing, token-based billing, prompt guard, or RAG injection without AI-aware plugins.

---

## Further Reading

- [Kong AI Gateway Overview](https://developer.konghq.com/ai-gateway/)
- [All AI Gateway Cookbooks](https://developer.konghq.com/cookbooks/?q=AI+gateway)
- [AI Plugin Reference](https://developer.konghq.com/plugins/?category=ai)
- [Hybrid Mode Deployment Guide](https://developer.konghq.com/gateway/latest/production/deployment-topologies/hybrid-mode/)
- [Kong Ingress Controller Docs](https://developer.konghq.com/kubernetes-ingress-controller/)
- [Kong Kubernetes Operator](https://developer.konghq.com/operator/)
- [Konnect Documentation](https://developer.konghq.com/konnect/)
- [decK CLI Reference](https://developer.konghq.com/deck/)
- [Kong Vault — Secrets Management](https://developer.konghq.com/gateway/secrets-management/)
- [AI Data Governance](https://developer.konghq.com/ai-gateway/ai-data-gov/)
- [Streaming (SSE) Support](https://developer.konghq.com/ai-gateway/streaming/)
- [Metering & Billing](https://developer.konghq.com/metering-and-billing/)
- [Model Context Protocol (MCP) Spec](https://modelcontextprotocol.io/)
- [Gateway API (Kubernetes SIG)](https://gateway-api.sigs.k8s.io/)
- [Langfuse OpenTelemetry Integration](https://langfuse.com/d