# Customer Journey Orchestration

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for cross-channel journey design, trigger-based automation, and real-time personalization.

Customer Journey Orchestration (CJO) coordinates how a brand engages individual customers across email, SMS, push, in-app, and web touchpoints. This project targets marketing, CX, and growth teams who today depend on heavyweight enterprise suites or narrowly scoped point tools, and aims to deliver real-time, AI-driven orchestration without the multi-month implementations and six-figure contracts that characterise the incumbent landscape.

---

## Why Customer Journey Orchestration?

- Enterprise platforms such as Adobe Journey Optimizer and Salesforce Marketing Cloud Journey Builder require 3–6 month implementations, systems-integrator partners, and custom contracts often exceeding $200k/year.
- Mid-market platforms (Braze, Insider, MoEngage) typically run $60k–$250k/year and are weighted toward mobile-first or specific regional ecosystems, leaving B2B and offline channels poorly served.
- SMB tools (Klaviyo, HubSpot) are accessible but offer limited real-time decisioning, shallow non-email channels, and cannot scale to enterprise volumes.
- Built-in AI in incumbents is largely bolted on; few platforms support autonomous journey design from natural-language goals or millisecond-latency predictive next-best-action.
- The category is large and growing: the global CJO market was approximately USD 13.1 billion in 2025 with forecasts of USD 86–89 billion by 2034 (CAGR ~24–25%), yet there is no credible open-source, AI-native alternative.

---

## Key Features

### Journey Design & Execution

- Visual journey builder with drag-and-drop canvas
- Event-based triggering and rule-based branching (if/then logic)
- Multi-step conditional branching and complex decision trees
- Journey versioning and A/B testing of entire flows
- Frequency capping and suppression rules

### Multi-Channel Messaging

- Email and SMS channel support with template builder
- Push notification and in-app messaging channels
- Template library and reusable journey components
- Generative message variant creation (copy)

### Audiences & Data

- Audience segmentation and dynamic list building
- CRM and CDP data integration
- Real-time event-stream processing for sub-second triggers
- Federated data evaluation patterns informed by CDP architecture

### Optimisation & Analytics

- A/B testing for messages
- Journey analytics and conversion tracking
- Advanced funnel analytics with drop-off insights
- Predictive send-time optimisation
- Automated next-best-channel recommendations

### Advanced / Backlog

- Autonomous journey design from natural-language goals
- ML-based multi-touch attribution
- Real-time journey anomaly detection
- Predictive drop-off prevention with intervention recommendations
- Account-based journey design and orchestration
- Journey outcome prediction before launch

---

## AI-Native Advantage

Where incumbents bolt AI onto existing canvases, this project is designed around AI from the ground up: an LLM agent can propose journey flows from a natural-language goal (e.g. "reduce churn in trial users"), real-time ML scoring replaces hand-built if/then logic with millisecond next-best-action decisioning, and generative models produce channel-specific message variants without creative-team bottlenecks. Continuous anomaly detection flags degrading steps before they surface in weekly reports, and ML-based multi-touch attribution feeds budget optimisation back into the orchestrator.

---

## Tech Stack & Deployment

The project aligns with established industry patterns and open standards: the CDP Institute Data Model for canonical event schemas and identity resolution, CloudEvents (CNCF) for vendor-neutral event payloads, Next Best Action decisioning for combining predictive models with business rules, and OpenID Connect / OAuth 2.0 for federated identity across CRM, CDP, and channel systems. Real-time streaming architectures (Kafka- and Databricks-style pipelines) are expected to underpin sub-second triggering, with REST and webhook APIs plus mobile and web SDKs for channel and CDP integrations. Consent management against GDPR, CCPA, and PDPA is a baseline requirement.

---

## Market Context

The global Customer Journey Orchestration market was valued at approximately USD 13.1 billion in 2025, with forecasters projecting USD 86–89 billion by 2034 at a CAGR of approximately 24–25% (Custom Market Insights, 2025). Enterprise contracts for Adobe and Salesforce frequently exceed $200k/year, mid-market platforms run $60k–$250k/year, and SMB tools start at tens of dollars per month. Primary buyers are VPs and Directors of Marketing and CX, Chief Digital Officers, and MarTech leads at retail, financial services, travel, and telecom companies with multi-million customer bases.

Candidate metadata: Complexity 8 / 10 · Domain availability: Low · Demand: High · Category: Marketing & Growth.

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
