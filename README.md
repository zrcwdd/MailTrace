# MailTrace

> An agentic AI email and DNS diagnostic tool that turns a vague deliverability problem into an evidence-backed troubleshooting trace.

[![Live demo](https://img.shields.io/badge/demo-live-1c7293?style=flat-square)](https://mail-trace--zrcwdd.replit.app)
[![Built with TypeScript](https://img.shields.io/badge/built%20with-TypeScript-3178c6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Powered by Gemini](https://img.shields.io/badge/powered%20by-Gemini-4285f4?style=flat-square&logo=google&logoColor=white)](https://ai.google.dev/)

![MailTrace hero](attached_assets/mailtrace-hero.png)

## What it does

MailTrace helps diagnose problems such as:

- Emails going to spam
- Emails not being delivered
- A domain not receiving mail
- Suspicious sender-authentication behavior

The user supplies a symptom and a domain. MailTrace then:

1. Forms a ranked set of likely causes.
2. Chooses the next most useful check.
3. Runs a live DNS lookup for that domain.
4. Interprets the result and updates its hypotheses.
5. Repeats until it has enough evidence for a diagnosis.
6. Presents a plain-language summary alongside the technical evidence.

The UI reveals the trace progressively so the diagnostic process is easy to follow instead of presenting a black-box answer.

## Why this is agentic

This is more than a single prompt-and-response flow. The reasoning agent selects its next tool based on the symptom and the evidence returned from the previous tool call. Each DNS result is fed back into the conversation history, allowing the agent to rule out causes, prioritize the next check, and stop when it reaches a confident conclusion.

The available tools are:

- **MX** — checks whether the domain has mail-receiving servers.
- **SPF** — checks sender authorization, including lookup-count limits and soft/hard fail behavior.
- **DKIM** — probes common selectors while correctly treating a miss as inconclusive.
- **DMARC** — checks the domain's enforcement policy (`none`, `quarantine`, or `reject`).

## Architecture

```text
React + Vite frontend
          │
          │ POST /api/diagnose
          ▼
Express API server
          │
          ├── Node DNS resolver
          │     ├── MX
          │     ├── SPF
          │     ├── DKIM
          │     └── DMARC
          │
          └── Gemini reasoning loop
                ├── choose next check
                ├── interpret evidence
                └── return diagnostic trace
```

## Tech stack

- **TypeScript** across the frontend and backend
- **React 19** and **Vite** for the web interface
- **Tailwind CSS** for styling
- **Framer Motion** for progressive trace animations
- **Lucide React** for icons
- **Express 5** for the API
- **Node.js DNS promises API** for live DNS resolution
- **Google Gemini API** using `gemini-3.1-flash-lite`
- **pnpm workspaces** for the monorepo
- **Replit artifact workflows** for development and deployment

## Repository layout

```text
artifacts/
├── api-server/
│   └── src/
│       ├── dnsChecks.ts          # MX, SPF, DKIM, and DMARC checks
│       ├── reasoningAgent.ts     # Gemini tool-selection and reasoning loop
│       └── routes/diagnose.ts    # POST /api/diagnose
└── mailtrace/
    └── src/
        ├── App.tsx               # Main diagnostic experience
        └── index.css             # MailTrace visual theme
lib/
└── api-zod/                     # Shared API types and validation
attached_assets/
└── mailtrace-hero.png           # Project showcase artwork
```

## Run locally

### Requirements

- Node.js 20+
- pnpm
- A Google Gemini API key with access to the selected model

### Install

```bash
pnpm install
```

### Configure the API server

Set the key in your shell or your platform's secret manager. Never commit the value:

```bash
export GEMINI_API_KEY="your-key-here"
```

The API server also requires a port:

```bash
export PORT=8080
```

### Start the services

In one terminal:

```bash
PORT=8080 GEMINI_API_KEY="$GEMINI_API_KEY" \
  pnpm --filter @workspace/api-server run dev
```

In another terminal:

```bash
PORT=22685 BASE_PATH=/ \
  pnpm --filter @workspace/mailtrace run dev
```

In Replit, the configured artifact workflows provide the expected ports and route the frontend's `/api` requests to the API artifact.

## API

### `POST /api/diagnose`

Request:

```json
{
  "symptom": "Our emails are going to spam",
  "domain": "example.com"
}
```

Response:

```json
{
  "trace": [
    {
      "type": "reasoning",
      "step": 1,
      "reasoning": "The symptom suggests an authentication or policy issue...",
      "hypotheses": ["...", "..."],
      "nextCheck": "DMARC"
    },
    {
      "type": "check_result",
      "step": 1,
      "check": "DMARC",
      "result": {
        "check": "DMARC",
        "success": true,
        "found": true,
        "policy": "none"
      }
    }
  ]
}
```

The agent runs for up to six reasoning steps and returns a structured trace. DNS checks are defensive: lookup failures become typed results rather than crashing the request.

## Important diagnostic behavior

MailTrace intentionally does not treat a missing record at a few guessed DKIM selectors as proof that DKIM is broken. Providers frequently use provider-specific or rotated selector names, so the result is reported as inconclusive unless there is corroborating evidence.

Similarly, a DMARC policy of `p=none` is interpreted as monitoring-only, not enforcement. That distinction matters when explaining why messages may still reach spam even when other settings exist.

## Security

- `GEMINI_API_KEY` is read only from the environment.
- No API key is required in the frontend.
- Local environment files and common credential file extensions are ignored by Git.
- Do not paste keys into source files, README examples, issue reports, or commit messages.
- If a key is ever exposed, revoke it and create a replacement immediately.

## Verification

```bash
pnpm run typecheck
pnpm run build
```

## Live demo

Try the deployed app at **[mail-trace--zrcwdd.replit.app](https://mail-trace--zrcwdd.replit.app)**.

## License

MIT