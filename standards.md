# Standards & API Reference

> Project: Customer Journey Orchestration · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 19510:2013 — Business Process Model and Notation (BPMN 2.0)**
- URL: https://www.iso.org/standard/62652.html
- BPMN 2.0, jointly published by OMG and ratified as ISO/IEC 19510, is the canonical standard for visually representing workflow and orchestration logic as swimlane diagrams. Customer journey builders are a domain-specific application of BPMN concepts: events, activities, and gateways map directly to journey triggers, actions, and decision branches.

**ISO 20252:2019 — Market, Opinion and Social Research**
- URL: https://www.iso.org/standard/73671.html
- Defines vocabulary and service requirements for market research and data analytics — the discipline underpinning journey analytics and customer insight functions within CJO platforms.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/82875.html
- The dominant security certification framework for SaaS platforms. Enterprise buyers consistently require ISO 27001 certification from CJO vendors as a procurement gate. Certification scope covers customer data storage, access control, and incident management.

### W3C & IETF Standards

**IETF RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorisation standard for delegated API access. All major CJO platforms use OAuth 2.0 to allow third-party integrations (CRMs, CDPs, data warehouses) to authenticate without sharing credentials. Bearer tokens (RFC 6750) carry the access grant in API request headers.

**IETF RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Specifies how Bearer tokens issued under OAuth 2.0 are transmitted in HTTP requests. The standard bearer-token pattern is the authentication mechanism used by Braze, Salesforce Marketing Cloud, Klaviyo, and others for their REST APIs.

**OpenID Connect Core 1.0 (OIDC)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- An identity layer built atop OAuth 2.0 that enables CJO platforms to federate user identity with enterprise IdPs (Okta, Azure AD, Ping Identity). Used for SSO into the journey builder admin console and for linking customer identity across CDP, CRM, and channel systems.

**W3C Web Authentication (WebAuthn) Level 2**
- URL: https://www.w3.org/TR/webauthn-2/
- Adopted as a W3C Recommendation; enables phishing-resistant passkey authentication for internal staff accessing CJO platform admin interfaces, replacing passwords.

### Data Model & API Specifications

**CloudEvents v1.0.2 (CNCF Graduated Standard)**
- URL: https://cloudevents.io / https://github.com/cloudevents/spec
- CNCF-graduated vendor-neutral specification for event data in event-driven architectures. CloudEvents defines a common envelope (id, source, type, datacontenttype, time) for events, making it the natural schema for real-time journey triggers. Widely adopted by AWS EventBridge, Azure Event Grid, and Google Cloud Eventarc. CJO platforms adopting CloudEvents gain interoperability with the broader cloud-native ecosystem without vendor lock-in.

**OpenAPI Specification 3.1 / 3.2 (OAS)**
- URL: https://www.openapis.org / https://swagger.io/specification/
- The de facto standard for describing REST API contracts in a machine-readable format. OAS 3.2.0 (September 2025) added streaming media type support (Server-Sent Events, JSON Lines) and native QUERY HTTP method support — capabilities directly relevant to real-time journey event streaming. All major CJO vendors expose OAS-described REST APIs for programmatic journey management.

**Arazzo Specification v1.0.1 (OpenAPI Initiative)**
- URL: https://www.openapis.org/arazzo-specification / https://spec.openapis.org/arazzo/latest.html
- A newer standard under the OpenAPI Initiative that describes sequences of API calls and the data dependencies between them — precisely the pattern of a customer journey (trigger event → segment lookup → message dispatch → outcome record). Arazzo 1.1.0 will extend support to AsyncAPI, enabling workflows spanning both HTTP and event-driven channels. Highly relevant for an AI-native CJO platform's internal workflow execution model.

**Adobe Experience Data Model (XDM)**
- URL: https://github.com/adobe/xdm / https://experienceleague.adobe.com/en/docs/experience-platform/xdm/home
- An open-source JSON Schema–based specification for standardising customer experience data. XDM defines canonical schema for customer profiles, events, and experience records. While developed by Adobe, it is publicly documented and extensible. Relevant as a reference data model for designing a portable customer event schema in a CJO platform.

**Segment Spec (Twilio Segment)**
- URL: https://segment.com/docs/connections/spec/
- Segment's tracking API defines a de facto JSON event schema widely adopted across the MarTech ecosystem (identify, track, page, group, alias, screen calls). Many CJO platforms (Braze, Iterable, MoEngage) natively consume Segment events as journey triggers. Building compatibility with the Segment spec gives a new CJO platform instant access to the 700+ integrations in the Segment catalogue.

**BPMN 2.0 XML Serialisation**
- URL: https://www.omg.org/spec/BPMN/2.0.2/About-BPMN
- The machine-readable XML format for BPMN diagrams enables journey definitions to be imported/exported between tools. Adopting BPMN 2.0 serialisation allows an AI-native CJO platform to interoperate with process modelling tools (Camunda, Zeebe, Bizagi) and to persist AI-generated journey graphs in a portable format.

### Security & Authentication Standards

**IAB Transparency and Consent Framework (TCF) v2.3**
- URL: https://iabeurope.eu/transparency-consent-framework/
- Industry standard consent signal format for digital advertising and personalisation. TCF v2.3 (April 2025) mandates that all vendors disclosed in a consent string are verifiably shown to users. Enforcement deadline: 28 February 2026. CJO platforms that trigger personalised messaging must consume and honour TCF consent strings; failure to do so blocks access to programmatic ad channels and exposes vendors to GDPR enforcement.

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- URL: https://gdpr-info.eu/
- The overarching EU privacy regulation governing all personal data processed in customer journeys. Requires lawful basis for each data processing activity (consent, legitimate interest, contract), right to erasure, and data minimisation. Cumulative GDPR fines have passed €7.1 billion. CJO platforms must implement consent-gating of journey steps, suppression lists, and audit logs.

**CCPA / CPRA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- US state privacy law (opt-out model) requiring CJO platforms to honour Global Privacy Control (GPC) browser signals and process data subject requests (access, deletion, opt-out of sale). Since January 2025, the California Privacy Protection Agency enforces violations immediately without a cure period.

**NIST SP 800-53 Rev 5 — Security and Privacy Controls**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- US federal security control catalogue widely adopted by enterprise buyers in regulated industries (financial services, healthcare, federal). CJO platforms pursuing enterprise contracts in the US financial or government sector are evaluated against NIST SP 800-53 controls.

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io / https://github.com/modelcontextprotocol/specification
- Open protocol enabling AI models (LLMs) to invoke external tools and data sources through a standardised server/client interface. Directly relevant for an AI-native CJO platform: MCP servers can expose journey builder actions (create journey, activate segment, dispatch message) to AI orchestration agents, enabling the autonomous journey design use case without bespoke LLM integration code.

---

## Similar Products — Developer Documentation & APIs

### Adobe Journey Optimizer (AJO)
- **Description:** Enterprise cross-channel journey orchestration built on Adobe Experience Platform; supports email, SMS, push, web personalisation, and in-app messaging with real-time decisioning.
- **API Documentation:** https://developer.adobe.com/journey-optimizer-apis/
- **SDKs/Libraries:** Adobe Experience Platform Mobile SDK (iOS, Android); Adobe Experience League docs at https://experienceleague.adobe.com/en/docs/journey-optimizer
- **Developer Guide:** https://experienceleague.adobe.com/en/docs/journey-optimizer/using/get-started/by-role/developer
- **Standards:** REST/JSON; OpenAPI-described endpoints; XDM-aligned data model
- **Authentication:** OAuth 2.0 with IMS (Adobe Identity Management System); client credentials flow via Adobe Developer Console

### Salesforce Marketing Cloud — Journey Builder
- **Description:** Visual canvas–based journey orchestration across email, SMS, push, in-app, and social channels; real-time triggers from Salesforce CRM and Data Cloud events.
- **API Documentation:** https://developer.salesforce.com/docs/marketing/marketing-cloud/guide/journey-builder-api-overview.html
- **SDKs/Libraries:** Salesforce Marketing Cloud SDKs (Node.js, Python, Java, PHP) at https://developer.salesforce.com/docs/marketing/marketing-cloud/guide/sdks.html
- **Developer Guide:** https://developer.salesforce.com/docs/marketing/marketing-cloud/guide/get-started-jb.html
- **Standards:** REST/JSON; Journey Specification is a structured JSON representation of journeys; Interaction Service REST API for CRUD operations on journeys; Event API for firing entry events
- **Authentication:** OAuth 2.0 client credentials (client ID + secret via installed package); Bearer token in Authorization header

### Braze
- **Description:** Mobile-first real-time engagement platform with visual Canvas journey builder across push, SMS, in-app, email, and web; millisecond-latency event processing.
- **API Documentation:** https://www.braze.com/docs/api/home
- **SDKs/Libraries:** iOS, Android, Web, React Native, Flutter, Unity, Cordova, .NET MAUI, Expo — full list at https://www.braze.com/docs/developer_guide/references
- **Developer Guide:** https://www.braze.com/docs/developer_guide/home / https://www.braze.com/docs/developer_guide/getting_started/integration_overview
- **Standards:** REST/JSON; API endpoints documented at https://documenter.getpostman.com/view/4689407/SVYrsdsG
- **Authentication:** REST API key as Bearer token in Authorization header; RSA256 JWT-based SDK Authentication for client-side requests; IP allowlisting support

### Iterable
- **Description:** Multi-channel lifecycle messaging platform with drag-and-drop journey builder supporting email, SMS, push, in-app, LINE, and RCS; developer-friendly REST API.
- **API Documentation:** https://api.iterable.com/api/docs
- **SDKs/Libraries:** Web SDK at https://github.com/Iterable/iterable-web-sdk; iOS and Android SDKs available via GitHub
- **Developer Guide:** https://support.iterable.com/hc/en-us/categories/360002288712-Developer-and-API-Docs / https://support.iterable.com/hc/en-us/articles/41044692130196-Getting-Started-with-Iterable-s-API
- **Standards:** REST/JSON; event schema follows standard webhook payload conventions
- **Authentication:** Standard API key or JWT-enabled key in request headers; stricter rate limits applied to query-string key delivery since November 2025

### Insider One
- **Description:** AI-driven cross-channel orchestration platform with built-in predictive next-best-action; strong presence in APAC/EMEA mid-market and enterprise.
- **API Documentation:** https://developers.insiderone.com/ / https://academy.insiderone.com/docs/api-reference-welcome
- **SDKs/Libraries:** iOS (Swift/Objective-C), Android (Kotlin/Java), React Native, Flutter, Cordova plugins; Postman collection at https://academy.insiderone.com/docs/insider-apis-in-postman
- **Developer Guide:** https://academy.insiderone.com/docs/developer-guide-overview
- **Standards:** REST/JSON
- **Authentication:** API key (Administrator-generated) passed in request headers

### MoEngage
- **Description:** Analytics-led mobile-first journey orchestration platform with strong funnel analytics and drop-off insights.
- **API Documentation:** https://developers.moengage.com/hc/en-us / OpenAPI-powered reference docs available via the developer portal
- **SDKs/Libraries:** iOS SDK at https://github.com/moengage/MoEngage-iOS-SDK; Android SDK at https://github.com/moengage/android-sdk; documentation at https://github.com/moengage/moengage-documentation
- **Developer Guide:** https://developers.moengage.com/hc/en-us/articles/4404674776724-Overview
- **Standards:** REST/JSON; data centre–specific API endpoints (region-aware base URLs)
- **Authentication:** API key per data centre; credentials passed in request headers

### Klaviyo
- **Description:** E-commerce–focused lifecycle messaging and automation with deep Shopify integration; accessible pricing and rapid onboarding.
- **API Documentation:** https://developers.klaviyo.com/en/reference/api_overview
- **SDKs/Libraries:** Server-side and client-side SDKs documented at https://developers.klaviyo.com/en
- **Developer Guide:** https://developers.klaviyo.com/en/docs/get_started / https://help.klaviyo.com/hc/en-us/articles/360045726811
- **Standards:** REST/JSON; OpenAPI-described endpoints
- **Authentication:** Private key (Bearer token) for server-side APIs; public site ID for client-side APIs; OAuth 2.0 for third-party integrations (token endpoint at https://a.klaviyo.com/oauth/token since March 2025)

### HubSpot Marketing Hub
- **Description:** SMB-friendly marketing automation and workflow platform bundled with HubSpot CRM; date-versioned REST API (2026-03 latest).
- **API Documentation:** https://developers.hubspot.com/docs/api-reference/latest/overview
- **SDKs/Libraries:** Node.js, Python, PHP, Ruby client libraries; Postman workspace at https://www.postman.com/hubspot/hubspot-public-api-workspace/overview
- **Developer Guide:** https://developers.hubspot.com/docs/getting-started/overview
- **Standards:** REST/JSON; date-based API versioning (replaces numeric /v3/ paths)
- **Authentication:** OAuth 2.0 for apps; private API keys for internal integrations

### Twilio Segment (CDP)
- **Description:** Leading customer data platform used as the data foundation for many CJO platforms; 700+ pre-built source and destination connectors; defines a widely adopted event tracking spec.
- **API Documentation:** https://segment.com/docs/connections/spec/ (Tracking API spec) / https://www.twilio.com/docs/segment
- **SDKs/Libraries:** Analytics.js 2.0 (web), iOS, Android, and 40+ server-side libraries; full list at https://segment.com/docs/connections/sources/catalog/
- **Developer Guide:** https://www.twilio.com/docs/segment/guides
- **Standards:** REST/JSON; proprietary Segment Spec (identify, track, page, group, alias, screen) as de facto JSON event schema across MarTech ecosystem
- **Authentication:** Write Key per source passed in request headers or SDK initialisation

---

## Notes

**Emerging: AsyncAPI for event-driven channels.** CloudEvents and the core OpenAPI Specification describe request/response HTTP APIs well, but push notification channels, SMS gateways, and real-time streaming (WebSockets, SSE) require event-driven API descriptions. The AsyncAPI Specification (https://www.asyncapi.com) is gaining adoption as the AsyncAPI counterpart to OpenAPI for message-driven architectures. Arazzo 1.1.0 is expected to bridge HTTP and AsyncAPI workflows, which is directly relevant once a CJO platform adds real-time streaming channel support.

**Evolving: Consent signal interoperability.** IAB TCF v2.3 (deadline February 2026) is the most immediate compliance forcing function for CJO platforms operating in Europe. US state law fragmentation (20+ states by end 2025) means platforms must build a consent signal abstraction layer rather than hardcoding GDPR or CCPA logic.

**Evolving: AI-native orchestration standards.** Model Context Protocol (MCP) is emerging as the standard interface for AI agents to invoke platform capabilities. No CJO vendor has published an MCP server specification yet; this is a first-mover opportunity for an open-source AI-native platform to define the standard interface between LLM agents and journey orchestration primitives.
