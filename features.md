# Expert Network Platform — Feature & Functionality Survey

> Candidate #497 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| GLG (Gerson Lehrman Group) | Managed expert network | Commercial / Enterprise subscription | https://glginsights.com/ |
| AlphaSights | Managed expert network | Commercial / Per-engagement + subscription | https://www.alphasights.com/ |
| Guidepoint | Managed expert network + data products | Commercial / Per-call + subscription | https://www.guidepoint.com/ |
| AlphaSense (incl. Tegus) | Market intelligence + expert transcripts | Commercial / SaaS subscription | https://www.alpha-sense.com/ |
| Third Bridge | Analyst-led expert network + content library | Commercial / Subscription | https://www.thirdbridge.com/ |
| Catalant | Enterprise consulting marketplace | Commercial / Custom pricing | https://catalant.com/ |
| NewtonX | AI-powered expert network | Commercial / Per-engagement | https://www.newtonx.com/ |
| CleverX | Self-serve research platform | Commercial / From $500/call credit | https://cleverx.com/ |
| Inex One | Expert network aggregator | Commercial / Pay-as-you-go | https://inex.one/ |
| proSapient | Tech-led expert network | Commercial / Per-engagement | https://www.prosapient.com/ |

## Feature Analysis by Solution

### GLG (Gerson Lehrman Group)

**Core features**
- Access to 900,000+ vetted experts across 150+ countries
- One-to-one expert calls with managed scheduling
- Surveys and events at scale
- Expert content library with searchable prior research
- White-glove project management with dedicated service teams

**Differentiating features**
- Strategic Research Agent: transforms client questions into research angles, expert recommendations, and relevant prior materials using AI
- Flexible Insight Gathering: 1:1 calls or AI-moderated conversations with real-time probing; multilingual survey data collection at global scale
- Precision Synthesis Tooling: generates summaries and collates themes across calls, expert content, and uploaded client materials (including video and audio)
- Configurable data governance for enterprise compliance requirements

**UX patterns**
- Reimagined myGLG client platform (May 2026 relaunch) with AI-augmented research workflow
- Every input routes to white-glove service teams, blending self-serve AI with human concierge
- Progressive disclosure: AI surfaces relevant prior research before sourcing new experts

**Integration points**
- Proprietary client platform (myGLG) — no publicly documented external API
- Enterprise SSO and configurable compliance integrations
- Support for uploaded third-party materials and multimedia

**Known gaps**
- Premium pricing limits access for mid-market and smaller firms
- Relationship-heavy sales model; no self-serve entry point
- No publicly documented developer API or third-party integration SDK

**Licence / IP notes**
- Fully proprietary commercial platform. No open-source components identified.

---

### AlphaSights

**Core features**
- Centralized dashboard for managing expert consultations, recordings, and transcripts
- Scheduling, tagging, and categorization tools for research project management
- Real-time notifications and progress tracking
- AlphaNow searchable library of expert research updated weekly

**Differentiating features**
- AlphaGraph: proprietary knowledge graph applying ML to 25+ million expert-to-company relationships for precision matching
- AlphaGPT: generative AI providing natural language answers synthesizing insights from expert research library
- AI Call Summary: every transcript delivered with free AI-generated summary
- AI Project Summary: dynamically updates after each call to provide full project synthesis

**UX patterns**
- Intuitive client platform for scheduling calls, launching surveys, messaging experts, and browsing research on demand
- Progressive enrichment: project summaries evolve as more calls are completed
- Quick-start onboarding with managed sourcing team

**Integration points**
- Platform supports scheduling via integrated calendar
- No publicly documented external REST API or SDK
- Survey and messaging features built into the platform

**Known gaps**
- Complaints of ghosting experts after confirming a fit
- Less suited to niche or deeply technical domains
- No self-serve tier; requires managed engagement
- Limited transparency on pricing

**Licence / IP notes**
- Fully proprietary commercial platform. No open-source components identified.

---

### Guidepoint

**Core features**
- One-to-one expert calls with dedicated project managers
- Scalable surveys and in-person events
- Coverage across 150+ industries and 500+ subsectors
- 1.75 million+ advisors in the network
- Qsight healthcare data product (Tracker, Sales Measurement)

**Differentiating features**
- Guidepoint360: AI-powered platform consolidating all client content (transcripts, recordings, project briefs, research) in one place
- AskGP: conversational AI interface for querying full content library in natural language
- AI Moderation for enterprise-scale expert calls
- Guidepoint360 mobile app for AI-driven research on the go
- MCP server integration with Anthropic's Claude for embedding expert insights into AI workflows
- 5,000+ new compliance-reviewed transcripts added monthly

**UX patterns**
- Single-platform consolidation of all research content
- Mobile-first AI research capability
- Conversational AI query interface reduces friction for insight retrieval

**Integration points**
- Model Context Protocol (MCP) server with Claude for AI-native research workflows
- Guidepoint360 platform with unified content library
- No publicly documented REST API or SDK beyond MCP

**Known gaps**
- Quality variance on niche topics reported by some users
- Healthcare focus may overshadow other industry verticals
- Pricing complexity with credit-based models

**Licence / IP notes**
- Fully proprietary commercial platform. MCP integration uses the open MCP standard but content is proprietary.

---

### AlphaSense (incl. Tegus)

**Core features**
- Unified search across 500+ million premium documents (filings, broker research, transcripts, financial data)
- Tegus Expert Transcript Library: 200,000+ transcripts covering 25,000+ companies
- 8,000+ new transcripts added monthly across 29,000+ companies
- NLP-powered semantic search understanding industry terminology and synonyms
- Citation-linked AI-generated insights

**Differentiating features**
- Expert transcript summaries structured into Expert Background, Expert Sentiment, and Takeaways
- Slide Builder: auto-generates executive-ready slides from Gen Search insights
- Pre-built AI agents for Investment Banking, PE/VC, and Investor Relations workflows
- Combined expert transcripts + public financial data + internal content in one platform

**UX patterns**
- Search-first paradigm: users query across all content types simultaneously
- Every AI insight links to exact source sentence for verification
- Structured transcript summaries enable rapid scanning
- Workflow-specific agent templates reduce setup time

**Integration points**
- Developer API at https://developer.alpha-sense.com/ with Expert Transcripts API (JSON format)
- API authentication via API keys, client credentials, and bearer tokens
- ETL pipeline support for pulling transcripts programmatically
- Enterprise SSO integration

**Known gaps**
- Investment-focused; narrower use case for general enterprise consulting
- Requires subscription; no pay-per-call model
- Less emphasis on live expert sourcing (transcript library focus)

**Licence / IP notes**
- Fully proprietary commercial platform. Developer API available under commercial terms.

---

### Third Bridge

**Core features**
- Analyst-led, moderated expert interviews producing structured, decision-ready content
- Library (formerly Forum) with 100,000+ analyst-led interview transcripts covering 65,000+ companies
- Maps product with visual value-chain intelligence on 100,000+ companies
- AI-powered search, personalized alerts, watchlists, and mobile apps

**Differentiating features**
- MCP-native architecture: first expert network to provide live, structured data that institutional clients can query using their own internal AI models in real-time
- Open distribution strategy: content available through multiple AI platforms (Claude, Aiera, Hebbia) rather than walled garden
- Custom-sourced experts for every request (no static panels or self-referrals)
- Analyst moderation produces comparable, reusable structured content

**UX patterns**
- Content-first approach: structured transcripts are the primary product, not just raw call access
- Personalized alerts and watchlists enable proactive insight delivery
- Open distribution means users access content wherever they already work

**Integration points**
- Model Context Protocol (MCP) server for native AI connectivity
- Content distribution through third-party AI platforms (Claude, Aiera, Hebbia)
- Standards-based MCP for compatibility with leading LLMs including ChatGPT and Claude
- No traditional REST API documented; MCP is the primary integration pathway

**Known gaps**
- Higher price point than access-model competitors
- Analyst-led model limits call volume scalability
- Focused primarily on private equity, hedge funds, and corporate strategy

**Licence / IP notes**
- Fully proprietary commercial platform. MCP integration uses the open MCP standard.

---

### Catalant

**Core features**
- Marketplace of 100,000+ independent consultants with 19+ years average experience
- Project posting, bid management, interview, selection, and payment in one platform
- Enterprise program for always-on access to external talent
- Customized contracting, workflows, and compliance standards

**Differentiating features**
- Enterprise-grade workforce augmentation: turns marketplace into an extension of internal teams
- Used by 30%+ of Fortune 100 for agile workforce needs
- Focus on project-based consulting engagements rather than one-off expert calls
- Internal talent deployment alongside external sourcing

**UX patterns**
- End-to-end project lifecycle management from scoping to payment
- Enterprise program integrates into existing corporate workflows
- Bid-based selection allows clients to compare consultants competitively

**Integration points**
- Enterprise workflow integration with corporate procurement systems
- Payment processing integrated into platform
- No publicly documented developer API

**Known gaps**
- Consulting-focused; not designed for quick knowledge calls or primary research
- Limited transcript or knowledge management features
- Not optimized for investment research use cases

**Licence / IP notes**
- Fully proprietary commercial platform. No open-source components identified.

---

### NewtonX

**Core features**
- AI-powered sourcing from a pool of 1.1 billion professionals across 140 industries
- Knowledge Graph: proprietary AI/ML-powered multi-query search engine
- Two-step ID verification for every expert
- Multiple research methodologies: panels, surveys, moderated interviews, long-term consultations

**Differentiating features**
- NXAD engine: extends sample feasibility for niche segments
- AI-moderated interviews: automates qualitative collection at quantitative scale
- NewtonX Prime: expert intelligence platform for investors (scale of survey + depth of interview + speed of search)
- Synthetic research powered by augmented data
- Hub platform centralizing recruitment, fieldwork, and analysis

**UX patterns**
- Research-methodology-first: clients select approach (survey, interview, panel) and AI optimizes sourcing
- Hub centralizes the full research lifecycle
- Speed emphasis: AI sourcing compresses time from brief to expert engagement

**Integration points**
- Hub platform with centralized workflow
- No publicly documented external API or SDK
- Trusted by Microsoft, Salesforce, Pinterest, LinkedIn

**Known gaps**
- Narrower expert coverage than GLG for some domains
- Less established brand recognition than Big Five networks
- Limited self-serve options for smaller engagements

**Licence / IP notes**
- Fully proprietary commercial platform. No open-source components identified.

---

### CleverX

**Core features**
- Self-service platform for sourcing, scheduling, and paying experts independently
- 8 million+ verified participants across industries (entry-level to CXO)
- AI-powered participant screening and fraud prevention
- Multiple research modes: AI-moderated interviews, human-moderated interviews, unmoderated testing, surveys

**Differentiating features**
- Full self-serve model with no middlemen required
- AI analyzes every response in real-time: surfacing themes, extracting quotes, generating reports
- Live 1:1 video interviews integrated with Google Meet, Zoom, and Teams
- Lower cost structure than managed networks (from $500/call credit)
- AI-moderated and human-moderated testing capabilities

**UX patterns**
- Self-serve onboarding with intuitive filtering and direct expert connection
- Clean, modern interface with minimal friction
- Real-time analysis feedback during research sessions

**Integration points**
- Video call integration with Google Meet, Zoom, Microsoft Teams
- No publicly documented developer API or SDK
- AI-driven incentive optimization built into platform

**Known gaps**
- Less curation than managed networks; quality depends on self-serve selection
- Smaller network than GLG or Guidepoint
- Limited enterprise compliance features compared to managed networks

**Licence / IP notes**
- Fully proprietary commercial platform. No open-source components identified.

---

### Inex One

**Core features**
- Aggregator combining 25+ expert networks and survey firms in one platform
- Consolidated sourcing, scheduling, and invoicing across all networks
- AI tools for aggregating and structuring expert insights
- Industry-leading speech-to-text transcription within 15 minutes of each call
- Proprietary transcript library accumulating all client research

**Differentiating features**
- Multi-network aggregation: intelligent algorithm matches projects to the most relevant expert network
- Transparent pay-as-you-go pricing in dollars (no opaque credit systems)
- Survey aggregation: full suite of primary research (expert calls + B2B/B2C surveys) in one platform
- Cross-network deduplication and standardized expert profiles

**UX patterns**
- Single interface for managing research across multiple networks
- Transparent pricing reduces procurement friction
- Knowledge management: all research accumulates in one searchable repository

**Integration points**
- API integration demonstrated (e.g., Enquire AI database integrated into Uzabase SPEEDA)
- Aggregation layer connecting 25+ expert network backends
- Transcription pipeline with automated delivery

**Known gaps**
- Dependent on underlying expert networks for sourcing quality
- Limited direct expert relationships (intermediary model)
- Platform features constrained by lowest-common-denominator of connected networks

**Licence / IP notes**
- Fully proprietary commercial platform. No open-source components identified.

---

### proSapient

**Core features**
- AI-powered semantic search and expert matching
- Integrated calendar and auto-scheduling for expert calls
- 99% accurate call transcription (AI + human vetters) in 48 languages
- Unified expert management across all sources with standardized bios and ratings
- Survey design, deployment, and panel management

**Differentiating features**
- Blend of AI transcription and human quality assurance for industry-leading accuracy
- Multi-network expert consolidation with smart deduplication
- Broader research management beyond calls: surveys, talent recruiting, post-deal role placement
- All communications with experts conducted through one platform

**UX patterns**
- Calendar-first workflow: auto-scheduling reduces administrative burden
- Unified expert view removes cross-platform friction
- In-platform messaging keeps all communications centralized

**Integration points**
- Calendar integration (work/team calendars)
- In-platform communication tools
- No publicly documented external API or SDK

**Known gaps**
- Smaller scale than the Big Five networks
- Less brand recognition in the US market
- Limited AI-native content synthesis compared to newer entrants

**Licence / IP notes**
- Fully proprietary commercial platform. No open-source components identified.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Expert sourcing and matching (manual or AI-assisted)
- One-to-one expert call scheduling and management
- Call recording and transcription
- Compliance screening and MNPI guardrails
- Expert profile management with qualifications and industry coverage
- Basic search and filtering of expert database
- Payment processing and invoicing
- Project management and engagement tracking

### Differentiating Features
- AI-powered semantic matching using knowledge graphs (AlphaGraph, NewtonX Knowledge Graph)
- Generative AI synthesis across multiple calls and documents (GLG Precision Synthesis, AlphaGPT, AskGP)
- AI-moderated interviews enabling qualitative research at quantitative scale (NewtonX, CleverX, GLG)
- Model Context Protocol (MCP) integration for AI-native research workflows (Guidepoint, Third Bridge)
- Searchable expert transcript libraries with structured summaries (AlphaSense/Tegus, Third Bridge Library)
- Multi-network aggregation with transparent pricing (Inex One)
- Open distribution strategy making content available across multiple AI platforms (Third Bridge)
- Mobile-first AI research applications (Guidepoint360 mobile app)

### Underserved Areas / Opportunities
- **Self-serve access for mid-market**: Most platforms require enterprise subscriptions ($60K+/year) or managed engagement; self-serve options (CleverX) have smaller networks
- **Cross-platform interoperability**: No standard protocol for expert data portability between networks; each platform is a silo
- **Real-time compliance automation**: Compliance is largely manual or semi-automated; full AI-driven MNPI screening with real-time trading surveillance is not standard
- **Expert-side experience**: Most platforms optimize for buyers; expert onboarding, profile management, and engagement quality feedback is underserved
- **Non-finance verticals**: Heavy focus on investment research and consulting; technology, healthcare operations, and public sector use cases are underserved
- **Open-source tooling**: No open-source expert network platform exists; all solutions are proprietary
- **Knowledge graph portability**: Proprietary knowledge graphs lock insights into individual platforms
- **Expert no-show management**: High expert no-show rates are a persistent industry complaint with no standardized mitigation

### AI-Augmentation Candidates
- **Expert sourcing and matching**: Replace manual recruiter effort with NLP-based semantic matching of research briefs to expert profiles
- **Compliance pre-screening**: Automate restricted-list checks, insider risk flagging, and MNPI boundary enforcement before each engagement
- **Call summarization and insight extraction**: LLM-powered transcription, summarization, and thematic analysis immediately after each session
- **Cross-call synthesis**: AI aggregation of themes, contradictions, and consensus across multiple expert calls on the same research question
- **Knowledge gap identification**: Analyze existing research body to identify specific expert profiles and questions needed to fill critical gaps
- **Expert recommendation engine**: Learn from past call history, ratings, and feedback to surface highest-rated experts for similar future briefs
- **AI-moderated interviews**: Conduct structured interviews with real-time follow-up probing, enabling scale without proportional headcount
- **Automated compliance documentation**: Generate NDA packages, conflict-of-interest screenings, and engagement audit trails automatically

## Legal & IP Summary

All ten solutions analysed are fully proprietary commercial platforms with no open-source components or openly licensed codebases. No patents were specifically identified in the research, though GLG, AlphaSights, and NewtonX reference proprietary AI/ML systems (Strategic Research Agent, AlphaGraph, Knowledge Graph respectively) that may be subject to trade secret or patent protection. AlphaSense offers the only publicly documented developer API (https://developer.alpha-sense.com/) among the solutions analysed. Third Bridge and Guidepoint have adopted the open Model Context Protocol (MCP) standard for AI integration, which uses an open specification but delivers proprietary content. There are no licence compatibility concerns for building an open-source alternative, as the category has no existing open-source entrants to consider.

## Recommended Feature Scope

**Must-have (MVP)**
- AI-powered expert profile matching using semantic search against research briefs
- Expert profile management with skills taxonomy, industry coverage, and availability
- Call scheduling with calendar integration and automated reminders
- Call recording, transcription, and AI-generated summary delivery
- Compliance workflow: NDA generation, conflict-of-interest screening, MNPI acknowledgement tracking
- Buyer/researcher project management dashboard with engagement tracking

**Should-have (v1.1)**
- Cross-call synthesis: AI-generated thematic analysis across multiple calls on the same project
- Searchable transcript library with structured summaries (Expert Background, Sentiment, Takeaways)
- Expert recommendation engine learning from past ratings, feedback, and engagement patterns
- Self-serve expert sourcing tier with transparent pay-as-you-go pricing
- Survey creation and deployment to expert panels
- MCP server for AI-native integration with Claude, ChatGPT, and other LLMs

**Nice-to-have (backlog)**
- AI-moderated interviews with real-time follow-up probing at scale
- Multi-network aggregation layer connecting external expert networks
- Knowledge gap analysis: AI identifies missing expertise in a research programme
- Mobile app for on-the-go research management
- Open distribution strategy: make transcript content available through third-party AI platforms
- Expert-side portal with engagement analytics, earnings tracking, and reputation management
