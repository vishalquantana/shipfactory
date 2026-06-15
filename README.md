# ShipFactory

A collection of production-tested feature specs, ready to hand to AI or developers for quick implementation.

## What is this?

ShipFactory is an open-source library of `.md` feature specifications. Each file describes a common product feature in enough detail — user flows, components, APIs, database schemas, integrations, and a porting checklist — that you can hand it to an AI coding assistant or a developer and get a working implementation fast.

These specs are extracted from real, shipped products. No theory — just battle-tested patterns.

## Who is this for?

- **Teams porting features across products** — grab a spec, hand it to your AI assistant, ship it
- **Solo founders & developers** — skip the design phase for common features and go straight to building
- **Anyone using AI-assisted development** — these specs are structured to work well as AI prompts

## How to use

1. Browse the [`/features`](./features) directory
2. Pick a feature spec
3. Feed it to your AI coding assistant (Claude, Cursor, Copilot, etc.) or use it as a reference for manual implementation
4. Adapt to your stack and ship

Each spec includes a **Porting Checklist** at the bottom — a step-by-step list of everything you need to implement.

## Feature Specs

| Feature | Description |
|---------|-------------|
| [Right-Click Bug Reporter](./features/rightclickbugreport.md) | In-app feedback & screenshot bug reporter with auto-capture, S3 upload, email notifications, and Jira integration |
| [Voice Conversational Intake](./features/voice-conversational-intake-prd.md) | Voice-powered form intake using ElevenLabs Conversational AI — full WebSocket protocol, agent setup, state machine, and client/server implementation |
| [AI Chatbot with RAG](./features/chatbotrag.md) | Intent-routed AI chatbot with pgvector RAG pipeline, proposed-changes review UI, 3-layer deduplication, voice/file input, and meeting prep |
| [E-Signature Documents (Self-Hosted DocuSign)](./features/esignature-docusign.md) | DocuSign-style e-signature module — PDF field builder, ordered token-based signers, type/draw/upload signatures, server flatten + Certificate of Completion, tamper-evident hash-chained audit trail, plus client-side PDF compress & merge |
| [Meeting Transcript Intelligence & RAG](./features/meeting-transcripts-rag.md) | End-to-end meeting-transcript pipeline — ingest from recorder bot / Google Meet / paste / file / API, LLM extraction into actions, notes, contacts & deal signals (auto-apply vs review), pgvector RAG over the whole collection, and a cited "Ask anything" agentic chat |

## Spec Structure

Every feature spec follows a consistent format:

- **Overview** — what the feature does
- **User Flow** — step-by-step interaction diagram
- **Frontend Components** — component breakdown with props, triggers, and behavior
- **Frontend Dependencies** — packages and versions
- **Backend API** — endpoints, auth, request/response formats
- **Database Schema** — table definitions with constraints
- **Integrations** — third-party services (S3, email, issue trackers, etc.)
- **Data Collected** — what's captured automatically vs. manually
- **Porting Checklist** — actionable implementation steps

## Contributing

Have a feature you've shipped and want to share? PRs are welcome.

1. Add your spec to `/features` as a `.md` file
2. Follow the structure above (adapt sections as needed — not every feature needs all of them)
3. Include a porting checklist at the end
4. Update the feature table in this README

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
