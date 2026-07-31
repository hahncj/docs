# Career Achievements Summary
### Prepared in support of Distinguished Engineer application

*Note: This is a first pass based on an initial recorded recollection covering ~30 years in the industry. The author noted they will record a follow-up session to add items that were missed, so this document should be treated as a living draft.*

---

## Professional Summary

Software engineer with close to 30 years of experience, specializing in two closely connected areas:

- **Middleware, domain services, and APIs** — roughly 25+ years designing and building back-end services, from early mainframe middleware through modern cloud-native microservices.
- **Financial account domain (account opening and maintenance)** — 20+ years of deep domain expertise in brokerage account types (individual, joint, IRA, 529, etc.) and the rules, regulations, and data involved in creating and maintaining them.

A recurring pattern across the career: being among the first to learn and adopt a new technology or architecture, then building the tools, documentation, and training programs to help other teams adopt it at scale.

---

## Career Timeline

### Late 1990s — Mainframe Middleware Foundations (A.G. Edwards)

- Joined the industry during the shift from green-screen terminal applications to desktop/GUI applications for account management and trading at brokerage firm **A.G. Edwards**.
- Worked on **Tandem Himalaya**, a fault-tolerant mainframe system, using a new middleware product, **Tuxedo**, running in a UNIX/POSIX environment — used to build services connecting mainframe systems of record, databases, and front-end desktop applications.
- Built account reference and security reference services, among others, for the firm.
- Built foundational cross-cutting infrastructure from the ground up for a high-transaction-volume environment, including:
  - Logging frameworks
  - Store-and-forward / guaranteed delivery messaging
  - Exception and error handling
  - Deployment automation (shell-script-based deployment tooling — early/novel for the time) to let other application teams push their services to the platform
  - Monitoring/observability dashboards (Java-based) giving visibility into backend service activity for a high-TPS system
- Wrote COBOL stubs to bridge decades-old mainframe (COBOL) applications with new C-based libraries running on the Tandem/UNIX side.

### Early-to-Mid 2000s — Front-End Experience & Entry into the Account Domain

- Sought front-end/customer-facing experience after years as a back-end service developer; took a lead engineer role on A.G. Edwards' **Electronic New Account Card (ENAC)** application (~2005).
- This is the point where deep experience in the **account opening and maintenance domain** began.
- Led the rewrite of ENAC from Java to **.NET**, to solve integration problems between the Java-based ENAC system and the Microsoft-based advisor desktop application it needed to work alongside.

### Late 2000s — A.G. Edwards / Wachovia Merger

- A.G. Edwards was acquired by/merged with **Wachovia**; moved onto Wachovia's systems for account opening and maintenance.
- Contributed to the web-form-based new account opening and maintenance applications — **1AM1 (Account Maintenance)** and **181/1AM1/1AM2, 1HP1** — applications that (notably) are, in some form, still in production today.
- These were monolithic applications at the time: all business logic, service logic, and direct database (system-of-record) access lived inside each application, causing duplicated logic across systems.

### ~2010-2013 — Shift to Service-Oriented Architecture (SOA)

- Industry/firm trend moved toward **Service-Oriented Architecture**: separating front-end, domain/service logic, and data-access layers (an early N-tier architecture) to enable reusability instead of duplicating logic (e.g., account balance retrieval) across every application.
- Recruited back into services work (having been a front-end/full-stack developer for a few years) specifically because of prior mainframe-era service-building experience.
- **Led the creation of 1BAS**, one of the very first service-domain applications in the firm's now-familiar "9-star" service domain landscape (e.g., 9CUS/customer, 9ITS/IT, 9COM/communication, etc.) — 1BAS predates the "9" naming convention because it came first.
  - Extracted shared logic out of the 181 (account opening) and 1AM1 (account maintenance) applications into SOAP services.
  - Defined governance criteria, a domain steering committee, and reusable patterns/standards for how services should be built and contracts shared.
  - Taught and helped other teams apply these same patterns to stand up their own service domains (the 9-star applications that exist today), marking an early leadership/mentoring milestone.
- Considers this the founding of the firm's service-oriented architecture approach to be one of the most significant and proudest accomplishments of the career.

### Performance & Scalability Wins (SOA era)

**1. Asynchronous new account opening (major latency reduction)**
- Problem: submitting a new account application required sequential calls to a mainframe system (referred to as "BETA") across ~10 screens' worth of data entry/validation/commit, taking 20–60 seconds while the user waited on a spinning submit button.
- Solution: re-architected the service to perform only the minimum steps needed to confidently return a valid account number, then placed the remaining work on a message queue for asynchronous processing by a separate consumer.
- Result: reduced user-facing submission time from an average of ~20 seconds (up to 45–60 seconds) down to **2–3 seconds**.

**2. Multi-threaded account retrieval for client dashboard (~2015)**
- Problem: a new "client dashboard" feature needed to display a customer's entire book of accounts at once, but the existing account-retrieval services only supported fetching one account at a time, causing unacceptable load times for customers with multiple accounts.
- Solution: designed and implemented a multi-threaded rewrite of the account retrieval service (delivered within about a week) that could accept a list of account numbers in a single request and retrieve them in parallel.

### 2017-2019 — Cloud Migration (Pivotal Cloud Foundry) & Java Adoption

- At the time, nearly all services were .NET Framework applications hosted on Windows server farms — scaling required new hardware and had roughly a 6-month lead time.
- Evaluated moving to a private cloud platform (Pivotal Cloud Foundry) as one of the first in the brokerage organization to explore cloud hosting.
- Identified that .NET Framework applications don't run natively on the Linux/UNIX-based infrastructure most cloud providers (and this platform) used, and that Windows-hosted alternatives carried prohibitive per-CPU licensing costs at scale.
- Ran a proof of concept: rewrote the account retrieval service (from the client dashboard project) in both **Java** and **.NET Core**, deployed both to a sandbox cloud instance, evaluated them, and **recommended Java** to leadership — a recommendation that was accepted despite most engineering teams at the time being .NET-only, given Java's stronger cloud-era ecosystem (open-source libraries, Spring framework, enterprise support, existing internal libraries).
- Piloted the approach by hiring a contractor (still with the firm today, now an FTE lead engineer) and porting two existing services (a 9TRN service — memo post-processing — and a 9ITS save service) to Java on the cloud platform, exposed as **RESTful/JSON APIs** rather than SOAP, while building a transformation layer (XML↔JSON) to keep existing SOAP consumers working without changes; also proved the alternate path of migrating a consumer directly to REST/JSON.
- Scaled this pattern to the rest of the application portfolio:
  - Stood up local/on-prem cloud instances with infrastructure engineering.
  - Built the **ADS (Application Delivery Services) framework** — a set of shared libraries handling cross-cutting concerns (logging, exception handling, messaging, BETA communication, etc.) — precursor to today's **Orchestra libraries**.
  - Built the **ADS Initializer**, a UI tool to scaffold new applications with these frameworks pre-integrated — precursor to today's **Orchestra IDP**.
  - Personally onboarded multiple partner teams' applications (9ITS, 9TRN, and others outside their own team) to the cloud platform, including writing some of that code directly.
  - Established a regular (weekly or twice-weekly) training/office-hours forum to teach other teams how to onboard to the cloud platform — this forum was later formalized by the architecture team into what is known today as the **Cloud Guild**, the first of the firm's now-multiple guilds (AI, Data, Quality, etc.).

### 2020 — Modern API Pattern Development with EY

- Spent roughly six months working with **Ernst & Young (EY)** to develop a more modern, standardized banking API pattern — a REST API backed by a local data store populated from upstream sources of record, with asynchronous updates back to systems of record (e.g., BETA, Hogan) via Kafka events/backend processors, rather than synchronous monolithic calls.

### 2022-2023 — 9HAB (Holdings, Activity, and Balances) & True Microservices

- Identified continued duplication of business logic across applications and domains even under SOA (e.g., inconsistent account balance calculations between the e-brokerage application and the 1BAS services).
- Focused on consolidating the transactional account space (balances, holdings, activity), which had outgrown the original owning team's capacity.
- Led (as a key contributor) domain-driven design and event-storming work with business stakeholders to define requirements, resulting in a new application: **9HAB (Holdings, Activity, and Balances)**.
- Helped stand up a brand-new team for this application — hiring some members and transferring others — including identifying and mentoring a principal engineer to lead the team, and mentoring/training engineers, scrum masters, and product owners from the ground up.
- Designed the software architecture: one of the first applications in the brokerage organization to own its own database (a dedicated MongoDB instance), moving away from a shared/centralized data store toward true microservice ownership of data.
- 9HAB is now a significant part of the firm's transition from the BETA mainframe system to **Maxit** for account transactional data.

### ~2020-2022 — Comms Platform Architecture

- Collaborated on modernizing the account opening/maintenance experience with a new UI initiative called **Comms**, which needed a new backing services layer.
- Co-architected (in collaboration with the Comms team) a modern API design departing from prior BETA-specific field-level naming, moving toward **business/ubiquitous language** (e.g., "account," "investment objective") shared between technology and the business.
- Designed to support a **micro-frontend** model (resource-oriented APIs with standard GET/PUT/POST/DELETE operations per micro-frontend) rather than one large monolithic form submission.
- Architecture pattern: services backed by local MongoDB data stores; every action publishes an event to Kafka; independent backend "micro-back-end" processors subscribe to those events and persist data to the appropriate systems of record (BETA, Hogan, Trust, SEI, etc.), each independently testable, deployable, and scalable.
- This architecture is still being extended today as Comms functionality expands, including into account maintenance.

### Late 2024-2025 — OpenShift (OCP) Migration

- Firm decision to migrate from the original Pivotal Cloud Foundry-based platform (now Tanzu Application Service) to a Kubernetes-based cloud provider (**OpenShift Container Platform / OCP**).
- One of the first to onboard components to the OpenShift platform.
- Completed formal "train the trainer" enablement on OCP, building on existing cloud and Kubernetes experience.
- Personally led three separate weeks of training sessions for engineers in the St. Louis office on OCP, and mentored other team members on the platform.

### 2025-2026 — AI Adoption Leadership

- Positioned as one of the leaders in AI adoption within the organization, including a leadership/administrative role connected to the organization's AI enablement efforts (referred to in the recording as "DSDC" — exact scope/title unclear from the recording and should be confirmed).
- Working with peer admins and organizations to establish AI usage standards and educate teams.
- Supporting Copilot onboarding — teaching teams how to use it effectively and efficiently.
- Involved in using AI for vulnerability management and remediation, helping ensure the organization stays compliant while adopting AI tools effectively at scale.

---

## Recurring Themes Across the Career

1. **Deep, dual expertise**: back-end services/APIs/middleware (~25+ years) combined with the financial account opening/maintenance domain (~20+ years).
2. **Early adopter, then multiplier**: consistently among the first to learn a new technology or architecture (Tuxedo middleware, SOA, cloud/Java migration, OpenShift, AI), and then documents, tools, and trains others so the whole organization can adopt it at scale.
3. **Founder of lasting organizational structures**: originated the firm's first service-domain application (1BAS) and the informal training forum that became the **Cloud Guild** — both of which grew into standing organizational patterns still used today.
4. **Performance/scalability problem-solving**: repeatedly identified and resolved significant user-facing performance problems (async account opening, multi-threaded account retrieval).
5. **Team- and capability-building**: stood up the 9HAB team from scratch (hiring, mentoring a principal engineer, training scrum masters/product owners) in addition to architecture design.

---

## Open Items / To Follow Up

The speaker explicitly noted this recording does not cover everything and that a follow-up recording is planned. Known gaps to revisit:
- Additional achievements not yet covered in this pass.
- Clarification on the exact title/scope of the "DSDC" AI-related role mentioned near the end.
- Any quantifiable metrics (team sizes, exact dates, performance numbers) that could be tightened up with documentation or data where memory was approximate (marked with "~" above).
