---

# 🌿 GreenChain — Automated ESG + Tokenized Offsets, Paid in PYUSD

**One‑liner:** *Track emissions, auto‑purchase & retire carbon credits as on‑chain tokens, and pay with PYUSD. Sensitive ESG data stays private by default, auditable on demand.*

---

## 1) IDEA SPEC SHEET

### 1.1 Problem & Why Now

* **Offset buying is fragmented** (finding, price discovery, settlement, and proof of retirement are scattered).
* **ESG reporting is manual/slow** (spreadsheets, quarterly crunches, audit anxiety).
* **Cross‑border payments add friction** (FX, delays, reconciliation).
* **Privacy is hard** (orgs want to prove impact without dumping raw facility data).

### 1.2 Solution Overview

**GreenChain** is a backend‑heavy, agent‑driven system that:

1. **Ingests energy/activity data** → computes emissions using recognized factors (EPA/UK Gov). ([Environmental Protection Agency][1])
2. **Runs a procurement Agent** that price‑checks providers and assembles a basket of carbon credits. Backed by Fetch.ai/uAgents / ASI stack. ([uagents.fetch.ai][2])
3. **Pays in PYUSD** on EVM (Sepolia for test/demo), with faucets for test tokens. ([docs.paxos.com][3])
4. **Mints/receives ERC‑1155 “credit tokens,” then retires (burns) them** with a verifiable on‑chain audit trail.
5. **Encrypts sensitive ESG docs/ledgers** and gates decryption for auditors via Lit PKPs/Actions (MPC‑TSS). ([developer.litprotocol.com][4])
6. **Explains everything in a simple dashboard** (live status → footprint, purchased/retired tons, cost, proofs) and links to an explorer (Blockscout) for trust‑but‑verify transparency. ([docs.blockscout.com][5])

> **Demo target story:** A “Hotel Lisboa” account uploads meter data (or streams from a mock). The Agent estimates monthly tCO₂e, buys credits from 2–3 mock “providers,” settles in PYUSD‑Sepolia, mints receipt tokens, retires them, and produces an auditor‑decryptable proof bundle.

---

### 1.3 Architecture (high‑level)

```
┌──────────┐       ┌──────────────────┐       ┌───────────────────────┐
│ Sources  │       │ Emissions Engine │       │ Procurement Agent (AI)│
│ (CSV/API)├──────►│  (EPA/UK factors)├──tCO₂e►│  (uAgents/ASI stack)  │
└────┬─────┘       └────────┬─────────┘       └─────────┬─────────────┘
     │                      │                           │
     │                      ▼                           ▼
     │              ┌────────────┐            ┌──────────────────────┐
     │              │ Order Book │◄──────────►│ Providers (mock svc) │
     │              └─────┬──────┘            └───────────┬──────────┘
     │                    │        PYUSD (Sepolia)         │
     │                    ▼        transfer/escrow         ▼
     │              ┌─────────────┐                  ┌───────────────┐
     │              │ Payment Svc │─────────────────►│ ERC-20 PYUSD  │
     │              └─────┬──────┘                  └───────────────┘
     │                    │ mint/retire events
     │                    ▼
     │              ┌─────────────┐   lit-encrypted docs/hash   ┌───────────────┐
     │              │ NFT1155 RWA │────────────────────────────►│ Lit Network   │
     │              │ + Retirement│◄── auditor decrypt (ACL) ───│ (PKP+Actions) │
     │              └─────┬──────┘                              └─────┬─────────┘
     │                    │   index/verify                              │
     │                    ▼                                            ▼
     │              ┌─────────────┐                               ┌────────────┐
     │              │ Blockscout  │                               │ Dashboard  │
     │              └─────────────┘                               └────────────┘
```

**Core repos (mono or poly):**

* `/contracts` (Solidity Hardhat): `CarbonCredit1155.sol`, `RetirementRegistry.sol`, `PaymentEscrow.sol`
* `/services/emissions-engine` (TypeScript/Node)
* `/services/agent` (Python/uAgents)
* `/services/providers` (Node mock market)
* `/services/payment` (Node Ethers.js for PYUSD)
* `/web` (Next.js 14, shadcn/ui, Tailwind)

---

### 1.4 Smart‑contract design (minimum viable, auditable)

**A) CarbonCredit1155 (ERC‑1155)**

* `struct CreditMeta {string registry; string projectId; uint16 vintage; uint256 gramsCO2e; string country; string methodology;}`
* `mint(to, id, amount, metaURI)` — only provider role
* `setMeta(id, metaURI)` — immutable after first retirement
* Events: `CreditMinted`, `MetaURISet`

**B) RetirementRegistry**

* `retire(id, amount, proofHash)` — burns `amount` and logs `Retired(id, amount, buyer, proofHash, timestamp)`
* `proofHash` = keccak256 of a canonical JSON bundle (invoice, delivery note, provider attestation) which itself is **Lit‑encrypted** off‑chain (see §1.6).

**C) PaymentEscrow**

* `createOrder(orderId, buyer, amountTCO2e, maxUnitPrice, expiry)`
* Agent fills against providers; escrow holds PYUSD (ERC‑20) via `transferFrom`.
* `settle(orderId, provider, unitPrice, creditId, tokenAmt)` mints credits + pays provider.
* For demo: allow direct Agent settlement (one tx).

> **PYUSD facts we’ll rely on**: It’s an ERC‑20 issued by Paxos, with public mainnet contract, **and** a **Sepolia testnet** address + faucet via Google Cloud for testing. We’ll run all demo flows on Sepolia. ([docs.paxos.com][3])

---

### 1.5 Emissions & data model

**Input data objects:**

* `Facility { id, name, country, egrid_subregion?, grid_intensity_source }`
* `MeterReading { facilityId, tsStart, tsEnd, kWh }`
* `EmissionFactor { region, year, kgCO2ePerKWh, source }`

**Computation:**

* Location‑based emissions: `tCO2e = sum(kWh) × factor_kg_per_kWh / 1000`.
* Factors sourced from **EPA Emission Factors Hub 2025** and **UK Gov 2024 conversion factors** (both widely used in disclosures). Include a T&D loss option per EPA update. ([Environmental Protection Agency][1])
* Keep factors editable per region to avoid hard‑coding national averages.
* For the demo story we can keep 25 t/month as a sample, but the engine is authoritative.

---

### 1.6 Privacy & auditability with Lit

* All **raw ESG artifacts** (invoices, attestations, meter export CSVs) are client‑side encrypted; only a **content hash** lands on‑chain via `proofHash`.
* Use **Lit Access Control** rules to allow decrypt for addresses that: (a) hold an “Auditor NFT” we mint, or (b) match an allowlist. Lit runs MPC‑TSS with TEEs; devs use **PKPs + Lit Actions** to sign/decrypt under policy. ([developer.litprotocol.com][4])

---

### 1.7 Agents (Procurement & Ops)

* Build the procurement Assistant in **Python** with **uAgents** (Fetch.ai/ASI). Capabilities:

  1. Pull facility tCO₂e for period.
  2. Request quotes from provider microservices (REST) and score by price/vintage/registry.
  3. Construct a **basket** (e.g., 60% nature‑based, 40% tech‑based mocks).
  4. Call Payment service to escrow PYUSD and settle.
  5. Trigger retirement and attach `proofHash`.
* uAgents has “hello‑world” and production patterns, plus Dockerized local runs; Agentverse is available if we want managed hosting for the demo. ([uagents.fetch.ai][2])

---

### 1.8 Explorer & indexing

* Spin up a **Blockscout Autoscout** explorer pointed at Sepolia filters for our contracts (or simply use a hosted Sepolia explorer and pre‑configure links in UI). Autoscout can stand up a custom instance in minutes. ([docs.blockscout.com][6])

---

### 1.9 Front‑end (explicit design direction)

**Framework:** Next.js 14 + Tailwind + shadcn/ui.
**IA:**

* **Overview:** footprint (this month), offsets bought, % coverage, PYUSD spent, last retirement (with block link).
* **Procure:** slider/select “offset N tCO₂e,” see live basket & PYUSD cost, click “Auto‑procure.”
* **Ledger:** list of *Retired Credits*; each row has `txn → explorer` + `View Proof` (Lit‑gated).
* **Settings:** facilities, factors source, auditor keys, API tokens.
  **Visuals:** charcoal/graphite base, emerald accents (#16A34A), monospace for on‑chain values.
  **Empty‑state copy:** “Connect a facility or upload a CSV to estimate your footprint.”

---

### 1.10 Key constraints & mitigations

* **Registry integrity:** Real carbon markets hinge on off‑chain registries like **Verra** and **Gold Standard**; tokenization must respect their policies. For hackathon we’ll use **mock providers** and clearly label tokens as *demo receipts*; we’ll **burn/retire** them in‑app with a clear “not a real credit” notice. ([Verra][7])
* **Factors variability:** Region/year matters—engine stores source & version for audits. ([Environmental Protection Agency][1])
* **Payments on testnet:** Use **PYUSD Sepolia** + faucet so judges can reproduce. ([docs.paxos.com][8])

---

### 1.11 “Golden path” demo flow (script)

1. **Upload CSV** (or click “simulate facility” → 50k kWh).
2. **Footprint shows** (e.g., 25 tCO₂e) with source badge “EPA 2025 (US)” or “UK Gov 2024”. ([Environmental Protection Agency][1])
3. Click **Auto‑procure** → Agent preview basket, PYUSD total, then confirm.
4. **Transaction toast** → “Paid 25.00 PYUSD (Sepolia)… Minted 3 credit tokens… Retired 25 tCO₂e.”
5. **Explorer link** → opens Blockscout instance filtered to our tx. ([docs.blockscout.com][6])
6. **Auditor view** → connect an Auditor wallet; “View proof” decrypts Lit bundle (invoice + attestation hash).

---

## 2) COMPLETE SPONSOR ENGAGEMENT PLAN

**Your TOP‑3 sponsors for this build:**

1. **PayPal USD (PYUSD)** — *Payments & settlement*
2. **Artificial Superintelligence Alliance (Fetch.ai / ASI)** — *Agents*
3. **Lit Protocol** — *Privacy & audit‑grade key management*

(We’ll still nod to **Blockscout** for explorer UX, but the focused prize pushes are the three above.)

### 2.1 PayPal USD (PYUSD) – How we’ll integrate (and show it)

* **Why them:** Stable, well‑known ERC‑20; demoable on **Sepolia** with an **official faucet** (Google Cloud). Judges can reproduce your flows. ([docs.paxos.com][3])
* **Build specifics:**

  * `PaymentEscrow` uses **PYUSD (Sepolia addr)**; FE shows “Add PYUSD‑Sepolia to wallet” helper. ([docs.paxos.com][8])
  * Button “Get test PYUSD” → links to **Google Cloud faucet** page. ([PayPal Developer][9])
  * Settlement receipts show **token, amount, tx hash** and unit price in PYUSD.
* **30‑sec sponsor pitch:** “GreenChain turns ESG payments into a one‑click PYUSD checkout—for offsets. Instant settlement, clean ledger, repeatable for any business.”
* **Scoring hooks:** on‑chain payments, good developer docs, and reproducible test path (Sepolia, faucet). ([docs.paxos.com][3])

### 2.2 Artificial Superintelligence Alliance (Fetch.ai / ASI) – How we’ll integrate

* **Why them:** They want real **agent** builds. uAgents is Pythonic and fits your team. Agentverse is optional managed infra for the demo. ([uagents.fetch.ai][2])
* **Build specifics:**

  * **Procurement Agent** in Python/uAgents: queries mock providers, optimizes basket, calls Payment + Contracts.
  * (Optional) publish on **Agentverse** for visibility; include route to trigger via REST. ([Fetch.ai Innovation Lab][10])
* **30‑sec sponsor pitch:** “Our agent transforms messy ESG procurement into autonomous, explainable decisions and executes settlement onchain—interoperable, auditable, and private.”
* **Scoring hooks:** use of official SDKs, composable marketplace logic, potential for cross‑agent negotiation.

### 2.3 Lit Protocol – How we’ll integrate

* **Why them:** Clean story: privacy‑by‑default ESG, auditable on demand. **PKPs + Lit Actions** + **MPC‑TSS** hit their sweet spot. ([Spark by Lit Protocol][11])
* **Build specifics:**

  * Encrypt “proof bundle” (attestations/invoices) client‑side; store CID in DB; store `keccak256(bundle)` on‑chain.
  * Gate decrypt by Auditor NFT or allowlist using **Access Control Conditions**; show an **Auditor mode** in FE. ([developer.litprotocol.com][4])
* **30‑sec sponsor pitch:** “GreenChain demonstrates how sensitive ESG data can be private while remaining verifiable—Lit‑gated proofs connected to on‑chain retirements.”

### 2.4 Nice‑to‑have sponsor taps

* **Blockscout (Autoscout)** for a branded explorer instance with our logo, filtered panels for our contract & token pages (quick wow for judges). ([docs.blockscout.com][6])
* **Hardhat** is already our Solidity workflow; call it out in README.
* **Pyth/Oracles** (optional): if we price credits in USD from a mock vendor feed later.

---

## 3) TEAM PLAN & 25‑DAY TIMELINE

**Team capability recap (5 ppl, overlapping skills):**

* **AI‑Lead A** — strong AI; comfortable Python; can own uAgents + decision logic.
* **AI‑Lead B** — strong AI; can own emissions engine & data quality.
* **Optimizer** — algorithmic thinker; can build basket optimizer & price scoring; supports Agent.
* **Backend Ace** — Solidity + Node services + infra glue.
* **FE Builder** — can ship Next.js UI if given explicit components/styles.

> Everyone is backend‑leaning; we’ll keep the UI small but sharp, and double down on infra/agents/payments.

### 3.1 Work breakdown by component

**Contracts (Backend Ace + Optimizer support):**

* `CarbonCredit1155`, `RetirementRegistry`, `PaymentEscrow` (unit tests with Hardhat).
* Deploy to **Sepolia**; document addresses; verify in explorer.

**Emissions Engine (AI‑Lead B):**

* Factor store (EPA/UK Gov), calculator lib, CSV importer, REST endpoints. ([Environmental Protection Agency][1])

**Procurement Agent (AI‑Lead A + Optimizer):**

* uAgents app (quote gathering, scoring, settlement orchestration). ([uagents.fetch.ai][2])

**Mock Providers (Backend Ace):**

* 2–3 Node services with different pricing/vintage/registry fields; simple signing of invoices.

**Payment Service (Backend Ace):**

* Ethers.js bindings for **PYUSD‑Sepolia**; faucet helper link; allowance/transfer/settle flows. ([docs.paxos.com][8])

**Privacy/Audit (AI‑Lead B):**

* Lit integration (encrypt bundle, Access Control Conditions, PKPs). ([developer.litprotocol.com][4])

**Front‑end (FE Builder + one buddy):**

* Next.js dashboard pages; wallet connect; “Auto‑procure” flow; auditor unlock; explorer links.

**Infra/DevEx (all):**

* Monorepo scripts, CI, seed scripts, README with sponsor callouts and a 5‑minute “Run the Demo” section.

---

### 3.2 25‑Day Gantt (pragmatic, parallelized)

**Sprint 0 — Day 1–2: Kickoff & scaffolding**

* Agree MVP scope & success criteria (one facility, monthly run, 1‑click procure).
* Repos scaffolded; envs; Sepolia RPC; Hardhat; Next.js; uAgents boilerplate.
* **Deliverable:** Repo boots locally; CI lints/tests; demo data CSV checked in.

**Sprint 1 — Day 3–7: Core rails**

* **Contracts**: 1155 + Registry stubs; events defined; typechain ready.
* **Emissions Engine**: Factor store + compute endpoint w/ unit tests. Sources cited in README. ([Environmental Protection Agency][1])
* **FE**: Layout + Overview page with mocked numbers; wallet connect; faucet link. ([PayPal Developer][9])
* **Agent**: uAgents “hello world,” receives a JSON order and logs a plan. ([uagents.fetch.ai][2])
* **Deliverable:** Walk‑through with mocked procure button → console plan.

**Sprint 2 — Day 8–12: Payments & market**

* **PaymentEscrow** implemented; PYUSD‑Sepolia integration; allowance/transfer test. ([docs.paxos.com][8])
* **Mock Providers** live (3 flavors).
* **Agent** now fetches quotes, scores, and calls Payment.
* **FE** shows live PYUSD cost, “Procure” button wiring.
* **Deliverable:** “Pay → Mint demo credits” end‑to‑end (no retirement yet).

**Sprint 3 — Day 13–16: Retirement & proofs**

* **RetirementRegistry** finalized; burn + event; FE ledger page.
* **Proof bundle** (JSON) created post‑settlement; on‑chain `proofHash`.
* **Blockscout** linkouts (or Autoscout mini‑instance). ([docs.blockscout.com][6])
* **Deliverable:** “Pay → Mint → Retire → See on explorer”.

**Sprint 4 — Day 17–20: Lit‑gated audit mode & polish**

* **Lit** encrypt/decrypt integrated; Auditor NFT or allowlist gating. ([developer.litprotocol.com][4])
* **FE** Auditor switch (connect wallet → “View Proof”).
* **Docs**: Sponsor READMEs (PYUSD, ASI, Lit) with short GIFs.

**Sprint 5 — Day 21–23: Hardening & demo**

* Seed script: deploy contracts, fund demo wallets (test PYUSD notes), register providers.
* Backtest script: run 2 months to show history.
* Observability: structured logs; basic health endpoints.

**Sprint 6 — Day 24–25: Dry‑runs & submission**

* Record the golden‑path video.
* Final README → “Reproduce in 5 minutes” (Sepolia, faucet, .env).
* Submit sponsor‑specific blurbs/screenshots.

---

### 3.3 Who does what (mapped to your team)

| Role            | Primary Ownership                                                                 | Backup/Support           |
| --------------- | --------------------------------------------------------------------------------- | ------------------------ |
| **AI‑Lead A**   | uAgents Procurement Agent, quote scoring interface, settlement orchestration      | Optimizer                |
| **AI‑Lead B**   | Emissions Engine (factors, calc), Lit integration, audit mode                     | Backend Ace              |
| **Optimizer**   | Basket optimizer (constraints/weights), provider scoring, perf tuning             | AI‑Lead A                |
| **Backend Ace** | Solidity contracts, Payment service (PYUSD), provider services, deployments       | AI‑Lead B                |
| **FE Builder**  | Next.js UX, wallet connect, flows (Procure, Ledger, Auditor mode), explorer links | Backend Ace (APIs/types) |

> Cross‑training daily: 15‑minute morning standup + 30‑minute evening code review.

---

### 3.4 “Definition of Done” (D25)

* [ ] Upload or simulate a facility, compute footprint with **source‑tagged** factor. ([Environmental Protection Agency][1])
* [ ] One‑click **Auto‑procure**: Agent builds a basket, settles with **PYUSD‑Sepolia**, and mints credits. ([docs.paxos.com][8])
* [ ] **Retire** credits on‑chain; show explorer link. ([docs.blockscout.com][6])
* [ ] **Lit‑gated** proof bundle decrypts for auditor wallet/NFT. ([developer.litprotocol.com][4])
* [ ] README: how to get **test PYUSD** (Google Cloud faucet), run scripts, and verify. ([PayPal Developer][9])
* [ ] 3 one‑page sponsor briefs (what we used, where in code, a 30‑sec pitch).

---

## Appendix — Technical Details You Can Plug In

### A. Contract events to show in Blockscout

* `CreditMinted(id, amount, provider, metaURI)`
* `OrderSettled(orderId, buyer, provider, unitPrice, creditId, amount)`
* `Retired(creditId, amount, buyer, proofHash, timestamp)`

### B. REST surface (thin but explicit)

* `POST /facilities` (create)
* `POST /readings: {facilityId, tsStart, tsEnd, kWh}`
* `GET /emissions/:facilityId?period=2025-09` → `{ tCO2e, factor, source }`
* `POST /orders` → `{ tCO2e, budgetPYUSD }` → returns `orderId`
* `POST /orders/:id/execute` → triggers Agent
* `GET /retirements?facilityId=...` → list with tx hashes and proof status

### C. Emission factor storage

* Table: `{ region, year, kgCO2ePerKWh, sourceURL, tAndDLossPct? }`
* Seed with EPA (US) + UK Gov (international fallback). ([Environmental Protection Agency][1])

### D. Lit integration sketch

* FE encrypts bundle → stores CID in object storage; smart contract stores `keccak256(bundle)` as `proofHash`.
* AccessControlConditions: “holder of Auditor NFT” OR “address in allowlist.”
* Lit Action mints a short‑lived decrypt signature if conditions satisfied. ([developer.litprotocol.com][4])

### E. PYUSD details for devs

* **Mainnet** (FYI reference): ERC‑20 by Paxos; public contract on Etherscan. ([Ethereum (ETH) Blockchain Explorer][12])
* **Testnet:** **Sepolia** address exposed in Paxos docs + **Google Cloud faucet**. Use this for all demos. ([docs.paxos.com][8])

### F. Agent framework links

* uAgents “create your first agent” guide + local run with Docker; Agentverse optional for hosting. ([uagents.fetch.ai][2])

---

## Why this wins with judges

* **Real sponsor deep‑links (not name‑drops):** actual PYUSD flows on testnet; actual Lit‑gated proofs; a working procurement Agent with uAgents. ([docs.paxos.com][8])
* **Back‑end excellence showcased:** events, receipts, explorer links, deterministic seeds. ([docs.blockscout.com][6])
* **Privacy + compliance narrative:** On‑chain truth + off‑chain confidentiality (exactly what ESG buyers and auditors want). ([developer.litprotocol.com][4])

---

### Quick links used above (for your README later)

* PYUSD integration & testnet: Paxos docs + Google Cloud faucet. ([docs.paxos.com][3])
* Lit Protocol: Access Control & PKPs/Actions; MPC‑TSS explainer. ([developer.litprotocol.com][4])
* ASI/Fetch.ai: uAgents/Agentverse docs. ([uagents.fetch.ai][2])
* Blockscout Autoscout (optional): deployment guide. ([docs.blockscout.com][6])
* Emission factors (EPA/UK Gov): references. ([Environmental Protection Agency][1])

---

If you want, I can also drop a ready‑to‑copy **repo structure**, a **seed script outline** (addresses, mint calls), and the **exact demo script** the presenter should read.

[1]: https://www.epa.gov/climateleadership/ghg-emission-factors-hub?utm_source=chatgpt.com "GHG Emission Factors Hub | US EPA"
[2]: https://uagents.fetch.ai/docs/getting-started/create?utm_source=chatgpt.com "Creating your first agent docs"
[3]: https://docs.paxos.com/guides/stablecoin/integrate-pyusd?utm_source=chatgpt.com "Integrate with PYUSD on Ethereum"
[4]: https://developer.litprotocol.com/sdk/access-control/encryption?utm_source=chatgpt.com "Encryption"
[5]: https://docs.blockscout.com/?utm_source=chatgpt.com "Blockscout Docs - Blockscout"
[6]: https://docs.blockscout.com/setup/deployment?utm_source=chatgpt.com "Deployment"
[7]: https://verra.org/programs/verified-carbon-standard/?utm_source=chatgpt.com "Verified Carbon Standard"
[8]: https://docs.paxos.com/guides/stablecoin/pyusd/testnet?utm_source=chatgpt.com "PYUSD on Test Networks"
[9]: https://developer.paypal.com/community/blog/testnet-pyusd/?utm_source=chatgpt.com "Get Instant Testnet PYUSD—Powered by Google Cloud"
[10]: https://innovationlab.fetch.ai/resources/docs/agentverse/?utm_source=chatgpt.com "Agentverse | Innovation Lab Resources"
[11]: https://spark.litprotocol.com/lit-actions-programmable-key-pairs-intro-guide-2/?utm_source=chatgpt.com "Intro to Lit Actions & Programmable Key Pairs"
[12]: https://etherscan.io/token/0x6c3ea9036406852006290770bedfcaba0e23a0e8?utm_source=chatgpt.com "ERC-20 Token | Address: 0x6c3ea903...a0e23a0e8 - Etherscan"
