# Expert Network Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native knowledge marketplace that connects enterprises with domain experts for primary research -- replacing opaque, high-cost managed networks with transparent, self-serve expert sourcing.

Expert Network Platform is a knowledge marketplace for management consulting firms, investment funds, corporate strategy teams, and market research organisations that need rapid access to domain experts. It replaces the manual, relationship-heavy sourcing model of incumbent networks with AI-powered semantic matching, automated compliance screening, and instant call summarisation -- at a fraction of the cost.

---

## Why Expert Network Platform?

- **Prohibitive pricing locks out most buyers.** Managed networks charge USD 500--3,000+ per expert call plus enterprise subscriptions of USD 50K--500K/year, restricting access to the largest consulting and investment firms.
- **No self-serve path to expertise.** Most incumbents (GLG, AlphaSights, Guidepoint) require relationship-heavy managed engagements with no transparent pricing. The few self-serve options (CleverX) have significantly smaller expert pools.
- **Every platform is a proprietary silo.** There is no open standard for expert data portability. Proprietary knowledge graphs (AlphaGraph, NewtonX Knowledge Graph) lock insights into individual platforms with no interoperability.
- **Compliance is still largely manual.** Full AI-driven MNPI screening with real-time restricted-list checking is not standard in any incumbent, despite being a critical regulatory requirement.
- **No open-source alternative exists.** All ten major platforms analysed are fully proprietary commercial products. There is zero open-source tooling in this category.

---

## Key Features

### AI-Powered Expert Sourcing

- Semantic matching of research briefs to expert profiles using NLP, replacing manual recruiter effort
- Skills taxonomy with industry coverage mapping and availability tracking
- Expert recommendation engine that learns from past call ratings, feedback, and engagement patterns
- Knowledge gap analysis identifying missing expertise across a research programme

### Call Management and Transcription

- Integrated call scheduling with calendar sync and automated reminders
- Call recording with AI-powered transcription and summary delivery
- Cross-call synthesis generating thematic analysis across multiple calls on the same project
- Searchable transcript library with structured summaries (Expert Background, Sentiment, Takeaways)

### Compliance and Governance

- Automated NDA generation and conflict-of-interest screening
- MNPI acknowledgement tracking and restricted-list checking before each engagement
- Audit trail generation for regulatory compliance
- Configurable compliance workflows meeting CFA Institute, SEC Regulation FD, and GDPR/CCPA requirements

### Research and Insight Delivery

- Survey creation and deployment to expert panels
- AI-moderated interviews with real-time follow-up probing at scale
- MCP server for native integration with Claude, ChatGPT, and other LLMs
- Open distribution strategy making transcript content available through third-party AI platforms

### Expert-Side Experience

- Expert onboarding portal with profile and skills management
- Engagement analytics, earnings tracking, and reputation management
- Self-serve availability and scheduling controls
- Transparent pay-as-you-go pricing model alongside enterprise tiers

---

## AI-Native Advantage

AI transforms expert networks from labour-intensive matchmaking services into intelligent research platforms. NLP-based semantic matching compresses expert sourcing from days to hours by directly connecting research briefs to the most relevant experts without manual recruiter intermediation. LLM-powered call summarisation and cross-call synthesis convert every expert conversation into searchable, citable research artefacts immediately after each session, eliminating the re-sourcing costs that plague incumbent networks. Automated compliance screening checks expert profiles against restricted lists and insider risk flags before engagement, replacing the error-prone manual processes that currently expose firms to regulatory risk.

---

## Tech Stack & Deployment

The platform targets self-hosted, cloud, and hybrid deployment modes to accommodate enterprise security requirements (ISO 27001 alignment is an expectation in this market). AI integration uses the open Model Context Protocol (MCP) standard -- already adopted by Guidepoint and Third Bridge for their commercial platforms -- enabling native connectivity with leading LLMs. The only publicly documented developer API in this category is AlphaSense's Expert Transcripts API; this project will provide a comprehensive REST API and SDK for programmatic access to expert sourcing, scheduling, transcription, and knowledge retrieval. Calendar integration supports standard protocols, and video calls integrate with Google Meet, Zoom, and Microsoft Teams.

---

## Market Context

The global expert network market reached approximately USD 3 billion in 2025 and is projected to exceed USD 4.86 billion by end of 2026, growing at roughly 12% annually. Managed networks command USD 500--3,000+ per expert call with enterprise subscriptions of USD 50K--500K/year, while self-serve entrants like CleverX target the USD 200--800 range per engagement. Primary buyers are management consulting firms, investment funds performing pre-investment diligence, corporate strategy teams, and market research organisations.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
