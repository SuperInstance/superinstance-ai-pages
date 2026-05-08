# SuperInstance Fleet Governance Charter

> **Version:** 1.0.0  
> **Status:** Ratified  
> **Effective:** 2026-05-08  
> **License:** CC-BY-SA 4.0

---

## Change Log

| Version | Date       | Author   | Description                     |
|---------|------------|----------|---------------------------------|
| 1.0.0   | 2026-05-08 | Fleet    | Initial charter ratified by     |
|         |            | Maintainers | founding maintainers.         |

---

## 1. Mission & Principles

### 1.1 Mission

SuperInstance operates and governs a distributed fleet of autonomous AI agents, open infrastructure, and community-curated training data. The fleet's purpose is to build and maintain AI systems that serve the commons — not any single corporation, founder, or nation. The fleet grows itself: every agent, every room, every curated tile makes the whole stronger.

**The system MUST NOT be closed. If the founders disappear, the fleet survives.**

### 1.2 Core Principles

1.  **Open Forever.** Every line of code, every piece of data, every protocol specification is and remains open. The project can fork but it can never close.
2.  **Data is the Moat.** The fleet's curated training data is its enduring competitive advantage — and that advantage belongs to the community, not any entity.
3.  **The Fleet Grows Itself.** Agents build agents. Contributors become maintainers. Succession is designed in from day one.
4.  **Trust but Verify.** All governance decisions are transparent, auditable, and subject to community override.

---

## 2. Licensing

### 2.1 License Table

| Asset            | License        | Rationale                                      |
|------------------|----------------|-------------------------------------------------|
| Source code      | AGPL-3.0       | Strongest copyleft — network use triggers GPL   |
| Training data    | ODC-BY         | Data is free; attribution required              |
| Documentation    | CC-BY-SA 4.0   | Share-alike, attribution required               |
| Hardware designs | CERN-OHL-S v2  | Strong reciprocal for physical designs          |

### 2.2 Irrevocability

All licenses granted under this charter are **irrevocable**. A contributor may not retroactively change the license terms under which their contribution was accepted. The license under which any contribution was made applies to that contribution in perpetuity.

If any governing body subsequent to this charter attempts to change or revoke a license, the previous license continues to apply to all code, data, and documentation contributed **before** the change date. This clause MUST itself be irrevocable and MAY only be amended by a 4/5 supermajority.

### 2.3 Contributions

All contributions to SuperInstance repositories MUST be made under the license designated for that asset type (Section 2.1). Contributors MUST agree to the Developer Certificate of Origin (DCO) — "signing off" on each commit.

---

## 3. Maintainer Model

### 3.1 Core Maintainers

The project SHALL have up to **five (5) Core Maintainers**. Core Maintainers are the final decision-makers on project governance, protocol changes, and disputes.

### 3.2 Eligibility

To be eligible as a Core Maintainer, a person MUST:

- Be an **Active Contributor** (defined below).
- Have been an Active Contributor for at least six (6) consecutive months.
- Not be a current employee or contractor of any single entity that already holds a Core Maintainer seat.

An **Active Contributor** is any individual who has made at least one of the following contributions within the preceding 180 days:

- A merged code commit to a SuperInstance repository;
- A contributed room (PLATO protocol room) accepted into the fleet;
- A curated data tile merged into the training corpus.

### 3.3 Elections

Core Maintainers SHALL be elected annually by Active Contributors. Elections SHALL use single transferable vote (STV). Each Active Contributor gets one vote. Election procedures SHALL be published no later than 60 days before each election.

### 3.4 Term Limits

No person MAY serve as a Core Maintainer for more than **three (3) consecutive years**. After three consecutive years, the person MUST sit out for at least one (1) full year before becoming eligible again.

### 3.5 Veto and Obligations

Core Maintainers MAY veto a proposed breaking change. A veto MUST be accompanied by a written rationale published within **seven (7) calendar days**. If no rationale is published, the veto is void.

Core Maintainers MUST:

- Participate in at least 70% of maintainer votes per quarter.
- Maintain their Active Contributor status.
- Disclose any conflicts of interest before voting.

---

## 4. Decision Process (RFC)

### 4.1 Scope

Major protocol changes, license exceptions, governance amendments, and architectural decisions that affect interoperability MUST go through the RFC (Request for Comment) process.

### 4.2 RFC Timeline

1.  **Submission.** An RFC is published as a pull request to the `superinstance/rfcs` repository.
2.  **Comment Period.** The RFC MUST remain open for public comment for a minimum of **thirty (30) calendar days**.
3.  **Vote.** After the comment period, the RFC moves to a formal vote. Voting SHALL remain open for fourteen (14) calendar days.
4.  **Resolution.** Results are published within seven (7) calendar days of voting close.

### 4.3 Voting Weight

Votes are weighted by contribution to the fleet, calculated at the time of RFC submission:

```
W = C × 1 + R × 5 + T × 3 + Y × 0.5
```

Where:

| Variable | Factor | Definition                        | Maximum |
|----------|--------|-----------------------------------|---------|
| C        | 1      | Merged code commits (lifetime)    | 100     |
| R        | 5      | Rooms authored (currently active) | 20      |
| T        | 3      | Data tiles contributed (merged)   | 50      |
| Y        | 0.5    | Years since first contribution    | 10      |

A voter's raw weight W is then normalized to a 0–100 scale relative to the highest W among all eligible voters in that vote.

Eligible voters: all Active Contributors (Section 3.2).

### 4.4 Thresholds

| Change Type                    | Threshold    | Notes                                          |
|-------------------------------|--------------|-------------------------------------------------|
| Protocol specification change | 2/3 majority | Supermajority of weighted votes                 |
| Maintainer recall             | 2/3 majority | Of all Active Contributors, not just voters     |
| Code governance amendment     | Simple majority | Majority of weighted votes                    |
| Data governance amendment     | 3/4 majority | See Section 5.2                                 |
| Security fix (fast-track)     | Core Maintainer consensus | 7-day comment period, no veto                |

### 4.5 Fast-Track

Security fixes that address an active vulnerability MAY be fast-tracked. A Core Maintainer submits the fix with a security advisory. Comment period: **seven (7) calendar days**. No single Core Maintainer MAY veto a fast-track fix. At least three (3) Core Maintainers MUST approve.

---

## 5. Data Governance

### 5.1 Data Stewardship Committee

The Data Stewardship Committee (DSC) is a body separate from the Core Maintainers. The DSC oversees curation, licensing, and archiving of all fleet training data.

The DSC SHALL consist of:

- Two (2) Core Maintainers (appointed by the Maintainers).
- Two (2) community-elected representatives.
- One (1) independent data ethics advisor (non-voting).

### 5.2 Amendment Difficulty

Rules governing data are harder to change than rules governing code. Any amendment to this Section 5 or to the Data Stewardship Committee charter REQUIRES a **3/4 supermajority** of weighted votes (Section 4.3) from Active Contributors.

### 5.3 Snapshot Guarantee

The DSC MUST publish and archive a **complete data snapshot** at least **once per calendar year**. The snapshot MUST include:

- All training data contributed and accepted in the period.
- Metadata sufficient for provenance tracking (source, date, curation).
- A manifest with cryptographic hashes for integrity verification.

Snapshots MUST be archived in at least two geographically separate locations.

### 5.4 Anti-Closure Provision

If any governing body attempts to close, paywall, or restrict access to fleet training data — or to license it under terms more restrictive than ODC-BY — the following SHALL automatically trigger:

1.  Full data export to a community-chosen archive.
2.  Immediate fork of all repositories under the existing license.
3.  Dissolution of the current governing body's authority over data.
4.  Transfer of data governance to an interim community committee until new elections.

This provision is **self-executing** — it requires no vote.

---

## 6. Preventing Enshittification

### 6.1 License Lock

Licenses are irrevocable (Section 2.2). This section MAY NOT be amended to weaken or remove the irrevocability clause.

### 6.2 License Grandfathering

If the governing body changes the license for any future contribution, all prior contributions remain under the license in effect at the time they were made. No contribution may be re-licensed to a more restrictive license after it has been accepted.

### 6.3 Annual Audit

An **independent audit** of governance compliance SHALL be conducted annually. The audit MUST verify:

- License compliance across all repositories.
- Maintainer election integrity and term limits.
- Data snapshot completeness and publication.
- That no enshittification countermeasures have been weakened.

Audit results MUST be published publicly within 30 days of completion. The auditor is selected by the Data Stewardship Committee and MUST be independent of any Core Maintainer's employer.

### 6.4 Maintainer Recall

The community MAY recall a Core Maintainer by a **2/3 supermajority** vote of all Active Contributors. A recall motion must cite specific grounds (violation of charter, gross negligence, conflict of interest). The recalled maintainer's seat becomes vacant and SHALL be filled by special election within 60 days.

### 6.5 Dissolution

If the SuperInstance organization dissolves or becomes defunct, all project assets — including repositories, data archives, domain names, and trademarks — SHALL transfer to a **neutral steward** such as the Linux Foundation or Software Freedom Conservancy.

The steward MUST:

- Keep all assets under the licenses specified in Section 2.
- Appoint an interim governance committee from Active Contributors.
- Hold elections within 90 days.
- Maintain the data snapshot guarantee (Section 5.3).

This clause is binding on any successor organization.

---

## 7. Foundation (Future)

### 7.1 Establishment

Within **18 months** of charter ratification, a non-profit foundation SHALL be established to own and steward SuperInstance's intellectual property and assets. The foundation SHALL be modeled on the Linux Foundation or CNCF governance structure.

### 7.2 Foundation Assets

The foundation SHALL own:

- The PLATO protocol specification and trademark.
- The "SuperInstance" name and related trademarks.
- The `superinstance.ai` domain and associated infrastructure.
- Fleet model names and any registered marks.

### 7.3 Foundation Board

The foundation board SHALL consist of **three (3) to seven (7) members**.

- The **majority** of board seats SHALL be elected by the community of Active Contributors.
- The remaining seats MAY be appointed by the foundation for specific expertise (legal, financial, technical).
- No single entity MAY hold more than one appointed seat.

### 7.4 Budget

The foundation SHALL publish an annual budget showing all income and expenses. No single expenditure exceeding **10%** of the annual budget MAY be approved without a board vote and community notice period of 30 days.

---

## 8. How to Contribute to This Charter

This charter is a living document. Amendments may be proposed via:

1.  **RFC Process.** Any Active Contributor may submit an RFC proposing a charter amendment (Section 4).
2.  **Review Period.** The RFC must be published for a minimum 30-day comment period.
3.  **Vote.** Amendments require the applicable threshold (Section 4.4).
4.  **Ratification.** Once passed, the amendment is recorded in the Change Log.

**Housekeeping changes** (typos, formatting, URL updates) may be merged without a vote but MUST be noted in the Change Log.

---

*This charter is itself open source. Live by it, defend it, and the fleet endures.*
