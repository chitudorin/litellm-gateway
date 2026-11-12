# litellm-gateway

One unified, OpenAI-compatible API in front of many LLM providers — mostly free tiers.
Point your client (Codex, Cursor, `curl`, anything OpenAI-compatible) at LiteLLM, ask for a
**single model name**, and the gateway decides which upstream provider actually serves it.

Collect API keys from every free LLM provider you can find, wrap them all under
[LiteLLM](https://docs.litellm.ai/) even though they serve different models, and upstream use
one alias such as `deepseek-flash`. Your client never learns that there are three providers
behind it.

## Why

Free providers hand out a few keys, small quotas, and inconsistent model names. Juggling them
per-client gets old fast. Behind the gateway:

- **One model name, many providers.** Codex is configured once, with a single model.
- **Priority + automatic failover.** Requests try providers in a fixed order; a rate-limited
  or dead provider is skipped, cooled down, and retried later.
- **Provider changes stay server-side.** Add, remove or reprioritise a provider by editing
  `litellm_config.yaml` — no client reconfiguration.

## How it works

`model_list` declares the *same* `model_name` several times, each entry pointing at a
different provider (`api_base` + `api_key`). What makes the priority list behave:

- `litellm_params.order` — strict priority, lowest number first. Every deployment in a group
  needs an `order`; a group that mixes ordered and unordered entries silently drops the
  unordered ones.
- Each order tier gets its own retry budget, so a failure on order 1 escalates to order 2 and
  then order 3 before the request is reported as failed.

`enable_pre_call_checks` is *not* related to priority: it only drops deployments the request
cannot fit on (context window, region, unsupported parameters), which is why it is useful here
— these providers differ in what they accept. When a deployment keeps failing it is cooled
down for `cooldown_time` seconds before being tried again.
See [`litellm_config.yaml`](./litellm_config.yaml).

Current alias in the shipped config: `deepseek-flash`

| order | provider | upstream model | base URL |
|---|---|---|---|
| 1 | `tokenharbor` | `deepseek-v4.1-flash:free` | `https://tokenharbor.ai/v1` |
| 2 | `apinex` | `free/deepseek-v4.1-flash` | `https://api.apinex.bond/v1` |
| 3 | `reflection-fallback` | `Beam-1.0-501B-A23B` | `https://api.reflection.ai/openai/v1` |

Note that the alias is a *slot*, not a model: order 3 serves a completely different model and
clients still see `deepseek-flash`. That is the point — any provider you can reach becomes
another backend for the same alias.

## Setup

### 1. Get provider API keys

Sign up with each provider and create an API key. The config references them through
environment variables (`api_key: os.environ/TOKENHARBOR_API_KEY`), so nothing is committed.
Export one variable per provider:

```sh
export TOKENHARBOR_API_KEY=...
export APINEX_API_KEY=...
export REFLECTION_API_KEY=...
```

Also set a master key for the gateway itself — clients authenticate against LiteLLM with this,
not with the provider keys:

```sh
export LITELLM_MASTER_KEY=sk-...
```

### 2. Run the gateway

```sh
docker run -d --name litellm -p 4000:4000 \
  -v "$PWD/litellm_config.yaml:/app/config.yaml" \
  -e LITELLM_MASTER_KEY \
  -e TOKENHARBOR_API_KEY \
  -e APINEX_API_KEY \
  -e REFLECTION_API_KEY \
  ghcr.io/berriai/litellm:latest \
  --config /app/config.yaml --port 4000
```

Sanity check:

```sh
curl -s http://localhost:4000/v1/models -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

### 3. Point Codex at it

`~/.codex/config.toml`:

```toml
model = "deepseek-flash"
model_provider = "litellm"

[model_providers.litellm]
name = "litellm"
base_url = "http://localhost:4000/v1"
env_key = "LITELLM_MASTER_KEY"
wire_api = "responses"   # the only supported value in current Codex
```

Codex now asks for `deepseek-flash`; LiteLLM translates and routes that to one of the
providers above.

**Why the `use_chat_completions_api` flag matters.** Codex only speaks the Responses API, so it
POSTs to `/v1/responses`. For an `openai/` deployment with a custom `api_base`, LiteLLM forwards
that request to `{api_base}/responses` — which most free OpenAI-compatible providers do not
implement, so the call fails. Setting `use_chat_completions_api: true` (already set on every
deployment in the shipped config) makes LiteLLM translate `/v1/responses` into the provider's
`/v1/chat/completions` instead. Drop the flag only if the provider natively serves `/responses`.

## Adding a provider

Append one entry to `model_list` in [`litellm_config.yaml`](./litellm_config.yaml) and
restart the gateway:

```yaml
- model_name: deepseek-flash
  litellm_params:
    model: openai/<upstream-model-id>   # as returned by the provider's /v1/models
    api_base: https://<provider>/v1
    api_key: os.environ/<PROVIDER>_API_KEY
    order: 4
    use_chat_completions_api: true   # needed for Responses-API clients (Codex)
  model_info:
    id: <provider>
```

Rules of thumb:

- Keep the `model_name` identical to attach to the same alias.
- `order` decides preference; ties are shuffled (`routing_strategy: simple-shuffle`).
- `order` is the failover mechanism, and retries are per tier — do not raise `num_retries` to
  walk the list.

## Limitations

- **Free tiers are unreliable.** Expect rate limits and outages; the order/cooldown machinery
  is what keeps the alias usable.
- **Not the same model everywhere.** Fallback providers serve different models with different
  context windows and quality. A request served at order 3 is not the same as order 1.
- **Responses-only clients need the bridge.** Anything speaking only `/v1/responses` (Codex)
  depends on `use_chat_completions_api: true` unless the provider implements that endpoint
  itself.
- **Some params are dropped.** `drop_params: true` silently strips parameters a provider does
  not support (`reasoning`, tool variants, and so on), so behaviour can differ per provider.
- **No per-key quota balancing.** Nothing rotates keys for one provider when that key is
  exhausted — add another key as its own deployment if you want that.
- **LiteLLM is a real dependency.** You are running a gateway; it needs to be up for your
  client to work.

## License

See [LICENSE](./LICENSE).
