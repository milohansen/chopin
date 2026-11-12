# Decisions classifier from Azure AI Foundry

Chopin needs a [decisions classifier](self-hosting.md#decisions) whenever the
Planner is on. When a deployment's models live in Azure AI Foundry, a small
loopback adapter, `scripts/foundry-decisions.ts`, serves that classifier from
Foundry instead of from an OpenAI or TypeSafe key. Chopin's code is unchanged:
Atomic's `models.json` points its classifier providers at the adapter, and the
adapter signs in to Foundry with the operator's Azure CLI login.

This suits a single trusted machine, such as
[local device-flow sign-in](authentication.md#local-device-flow-sign-in). The
adapter listens on `127.0.0.1` only and accepts only requests that carry a
random token from `models.json`.

## Two routes

| `DECISIONS_MODEL`             | Adapter route   | Foundry model                                                                                      |
| ----------------------------- | --------------- | -------------------------------------------------------------------------------------------------- |
| `typesafe/jev-latest`         | `/systemone`    | A Microsoft decision model deployment, such as `Decision-1`. Recommended.                          |
| `openai-decisions/gpt-6-luna` | `/v1/decisions` | A chat deployment named by `FOUNDRY_CHAT_DEPLOYMENT`. Fallback when no decision model is deployed. |

- **Decision model.** Foundry's Microsoft decision models accept the System One
  request Atomic already sends for `typesafe/*` classifiers and return
  calibrated probabilities. The adapter replaces the model name with
  `FOUNDRY_DECISION_DEPLOYMENT` and forwards the request to
  `/providers/microsoft/v1/systemone`. Nothing is sent to TypeSafe.
- **Chat model.** Foundry does not offer OpenAI's Decisions API, and reasoning
  models return no token probabilities. The adapter asks the chat deployment
  for structured answers instead, so every probability is the model's own
  estimate. Conversation cards, turn-taking and visual admission then run on
  numbers they were not tuned for. In one comparison over three imported
  documents, visual admission demanded about twice as many visuals with the
  chat route as with `Decision-1`, and took over ten times as long.

## Configure

Add these to the Chopin `.env`. Bun loads that file when the adapter starts from
the repository root.

```text
DECISIONS_MODEL=typesafe/jev-latest
DECISIONS_TIMEOUT_MS=60000
# Used only by Chopin, never your own Atomic CLI directory.
ATOMIC_CODING_AGENT_DIR=/home/<you>/.config/chopin/atomic-agent
FOUNDRY_ENDPOINT=https://<resource>.services.ai.azure.com
# Optional: the Foundry resource's tenant, when the Azure CLI is signed in to several.
FOUNDRY_TENANT_ID=
# Optional; defaults to Decision-1.
FOUNDRY_DECISION_DEPLOYMENT=
# Optional: a chat deployment for the openai-decisions route.
FOUNDRY_CHAT_DEPLOYMENT=
```

Leave `OPENAI_API_KEY`, `JEV_API_KEY` and `TYPESAFE_API_KEY` empty, so the
only credential for these providers is the adapter's token.

Then sign in to the Foundry tenant once (`az login --tenant <tenant>`), with an
identity that may call the deployments, and write `models.json`:

```bash
bun scripts/foundry-decisions.ts init
```

`init` creates the directory with owner-only permissions, generates the token,
and adds whichever of the two providers is missing. It never changes an
existing entry.

## Run

Start the adapter before Chopin and leave it running:

```bash
bun scripts/foundry-decisions.ts
bun run dev
```

The adapter prints its routes, and Chopin's startup banner shows
`visual assessment: typesafe/jev-latest`. Each classifier request logs one
line, such as `200 systemone 1115ms in=158 out=3`.

Chopin starts without the adapter, because startup only checks that the
classifier is configured. Planner writes then fail visual admission with
`E_VISUAL_ASSESSMENT` until the adapter is running.

Set `FOUNDRY_ADAPTER_PORT` to move the adapter off 8790, and run `init` again
in a fresh `ATOMIC_CODING_AGENT_DIR`, because `init` keeps existing entries.
