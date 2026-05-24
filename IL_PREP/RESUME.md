# 📄 RESUME
> **Status:** 🟡 Draft v2 in progress — spacing tweaks pending  
> **Target:** A tight, honest, IL-tailored 1-page resume for SDE2 role  
> **Files:** `aishwarya_resume.tex` + `aishwarya_resume.pdf` (in this folder)

---

## Candidate Snapshot

| Field | Detail |
|-------|--------|
| Name | Aishwarya Soni |
| Email | aishujsoni@gmail.com |
| LinkedIn | linkedin.com/in/aishwarya-soni-554203192 |
| GitHub | github.com/aishusoni |
| Location | Bangalore, India |
| Current Role | SDE2, Honeywell HCE (Jul 2023 – Present) |
| Education | IIT Kharagpur Dual Degree, ECE, CGPA 8.6/10 |

---

## Resume Structure (Current Draft)

### EXPERIENCE — Honeywell Connected Enterprise (HCE)
*Software Development Engineer 2 | Jul 2023 – Present | Bangalore*

**Forge Operation Center (FOC)** — Executive Intelligence Dashboard
- Architected and built unified executive dashboard aggregating ~980TB across 30 products, 25K customers, 282K sites, 5M assets, and 85M data points for C-suite and VP-level stakeholders
- Migrated gateway-product mapping from a 4-hour failure-prone Celery batch job to parallelised Kafka consumers with DLQ and automated alerting, reducing runtime by 87% (4 hr → 30 min)
- Integrated Google Earth API to render a live world map of 40K IoT gateways and 282K customer sites across all Honeywell products

**Platform Health Dashboard** — Internal SRE Monitoring
- Built configurable health monitoring platform tracking 50 services across 8 modules, ingesting metrics from InfluxDB, Victoria Metrics, Azure IoT Hub, Databricks, and custom APIs
- Designed zero-code service onboarding via customised Django admin — SREs configure thresholds and data sources without any engineering intervention
- Built OpenAI-powered summary agent surfacing failure patterns, cross-service correlations, and degradation trends over user-selected time ranges

**SaaSOps Portal** — SRE Automation Platform
- Built centralised Azure service-principal secret management across 5 tenants and 100+ principals with RBAC, 30-day expiry alerting, and Octopus auto-propagation — eliminating manual secret-sharing dependencies
- Developed DEC operations dashboard tracking ~1,000 tenant jobs with guided customer onboarding, real-time job-status visibility, and failure monitoring across Databricks pipelines

---

### EDUCATION — IIT Kharagpur (2018–2023)
- M.Tech Dual Degree (B.Tech + M.Tech), Electronics & ECE | CGPA: 8.6/10
- Minor: CSE (CGPA: 8.75/10)
- Specialisation: Visual Information Processing & Embedded Systems
- Micro-specialisation: Embedded Control, Software, Modelling & Design
- **M.Tech Thesis:** "The Art of Persuasion in the Face of Skepticism: Bayesian Methods for Handling (dis)Trust" — Prof. Amitalok J. Budkuley
- **IEEE ISIT 2024:** "Bayesian Persuasion: From Persuasion toward Counter-suasion" | 2nd author

---

### SKILLS
| Category | Skills |
|----------|--------|
| Languages | Python, JavaScript/TypeScript, C/C++, SQL |
| Frameworks | Django, React, Node.js, Celery |
| Infrastructure | Azure (IoT Hub, Event Grid, Service Bus), Kafka, Redis, PostgreSQL, InfluxDB, Victoria Metrics |
| Tools | Git, Databricks, OpenAI API, Google Earth/CesiumJS |
| Concepts | Microservices, event-driven architecture, multi-agent systems, REST APIs, observability |

---

### AWARDS & ACHIEVEMENTS
- JEE Advanced 2018 — All India Rank 1050 (Top 0.07% among 1.5M candidates)
- IEEE ISIT 2024 — Published "Bayesian Persuasion: From Persuasion toward Counter-suasion"
- Inter-IIT 2022–23 — Gold Medal, Music Cup (1st among all IITs)
- NSEA 2017 — State Rank 2, Gujarat

---

## Open Items
- [ ] Fix LaTeX line spacing — v2 too tight, needs visual balance
- [ ] Get IL Job Description → tailoring pass on bullet ordering + action verbs
- [ ] Update LinkedIn to match resume
- [ ] Decide on projects section (recommendation: skip unless JD reveals a gap)

---

## Tailoring Notes (from COMPANY_IL.md)

### What to emphasise for IL
1. **Scale** — 980TB, 5M assets, 85M points → parallels IL's 18M student platform
2. **AI integration** — OpenAI summary agent, multi-agent architecture → maps to Curriculum-Informed AI™
3. **Configurable platforms** — zero-code onboarding design → IL engineers build for curriculum flexibility
4. **Backend data systems** — Kafka, Celery, PostgreSQL at scale → IL needs reliable data pipelines

### Bridge story (no ed-tech experience)
> "The core engineering problems — real-time data at scale, adaptive systems, configurable platforms, AI integration — are the same. The domain changes, the problems are isomorphic."

---

## Version History
| Version | Status | Notes |
|---------|--------|-------|
| v1 | ❌ Too much whitespace | First LaTeX build |
| v2 | 🟡 Spacing too tight | Overcorrected — needs balance |
| v3 | ⬜ Pending | Spacing fix in progress |

