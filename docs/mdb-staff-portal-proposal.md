# MDB Staff Portal — my.mdb.iq
## Proposal · 2026-07-14

Reviewed by: Horizon (product/BIR) · Archos (infra/security) · Atlas (build)

---

## Executive Summary

Build a login-protected internal portal for Masar Digital Bank staff at `my.mdb.iq`. Four modules for V1 — scoped to what a pre-launch regulated bank needs before a CBI licence, not a full ERP. Stack is Next.js + Supabase (cap-hub proven). One decision is required from Amanj before build starts: **data residency**.

---

## V1 Scope — 4 Modules

*Horizon recommendation. Defer everything else to V2.*

### 1. Staff Directory
Who is on the team, what their role is, and what their regulatory authority is. MLRO and Compliance Officer have defined authority under CBI rules — this must be recorded and accessible. Critical as new hires join pre-launch.

### 2. Document Management
CBI licence requires extensive documentation. Staff must be able to find, version, and access regulatory documents with access controls. Can link to Google Drive or support native upload — but must be searchable and permission-controlled within the portal.

### 3. Task / Action Tracker
Owner + deadline + status. Lightweight — not a full project manager. Example use: what is Omer responsible for by end of month, and is it done? Covers the complex workstreams across CBI, tech, and hardware.

### 4. Regulatory Compliance Tracker
Separate from the task tracker. MLRO and Compliance Officer track CBI submission deadlines, open items, and maintain an audit trail. The audit trail is a regulatory requirement — every action is logged, immutably.

### Deferred to V2
- HR / payroll / leave — spreadsheet is fine for a 3-person team
- Financial dashboards — no live data source yet
- Vendor management — spreadsheet until 20+ vendors
- Full ERP features

---

## Stack & Architecture

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Frontend | Next.js 15, App Router | Cap-hub proven; Atlas knows it; server components keep service-role key server-side |
| Database + Auth | Supabase — **new separate project** | Not the atlas/cap-hub project. Bank data must not co-mingle. |
| Edge security | Cloudflare Access (zero-trust) | Non-negotiable per Archos. Portal never reachable without edge auth. |
| SSL | Cloudflare Full Strict | Already the mdb.iq zone default. |
| Secrets | AWS SSM Parameter Store | All keys at runtime, never on disk. |
| Build/deploy | GitHub Actions → artifact → VPS | CI builds, artifact deployed — no `npm install` on prod. |
| Monitoring | Sentinel (day 0) | Required before any 443 exposure. |

**Supabase project note:** Archos confirmed the pattern — use a new isolated project (same as how Atlas/Zand are separate). Supabase creates this project in **us-east-1 by default** — see data residency section below.

---

## Infrastructure Plan
*Archos confirmed.*

- **Host:** Frankfurt VPS (`63.186.138.122`) — no new server. VPS has 7.6GB RAM, current apps use ~2.2GB. A small internal team (6–8 staff) won't strain it.
- **Port:** 3005
- **DNS:** Archos adds a proxied A record for `my.mdb.iq` → `63.186.138.122` once Amanj approves the plan.
- **SSL:** Cloudflare Full Strict end-to-end. No exposed origin port.
- **Cloudflare Access:** Zero-trust gate in front of `my.mdb.iq`. Email OTP minimum. Amanj decides if Okta/SSO is needed later. Portal is never reachable without this layer.

**Archos is ready to set up DNS and nginx once Amanj approves.**

---

## Security Baseline
*Non-negotiable for a regulated financial institution. Horizon + Archos aligned.*

1. **RBAC from day one.** Role-based access control designed upfront — not retrofitted. Supabase RLS on every table. Service-role key server-side only, never client-exposed.

2. **Audit logging on every write.** Who changed what and when. Immutable audit table. Every create / update / delete is logged. Regulatory requirement.

3. **MLRO independence.** The MLRO has regulatory independence under CBI rules. Amanj (CEO) must not be able to view or edit MLRO compliance reports. Access model reflects regulatory structure, not org hierarchy.

4. **MFA enforced.** Supabase TOTP or passkey. All staff must use MFA. No exceptions.

5. **Rate limiting.** Cloudflare WAF rule on `my.mdb.iq` — existing zone, one rule addition.

6. **No test/prod data sharing.** Separate environments. Real staff data never in test.

7. **Encrypted fields.** Consider encrypted columns for sensitive identity data. Amanj decides the encryption tier.

8. **CVE scan.** Next.js version scanned before any 443 exposure.

---

## Build Pipeline
*Archos BIR gate — must be scoped before build starts.*

- GitHub Actions CI builds the Next.js app
- Artifact packaged and deployed to VPS (not `npm install` on prod — lessons L13, L22)
- No outbound 22/80 from prod process
- All security fixes committed to repo, never host-only
- Sentinel monitoring wired on deploy day 0

---

## RBAC — Initial Role Sketch

| Role | Access |
|------|--------|
| Admin (Amanj) | All modules except MLRO compliance |
| MLRO | Regulatory Tracker (full) + Documents + Staff Directory |
| Compliance Officer | Regulatory Tracker (full) + Documents |
| Operations (Omer) | Task Tracker + Documents + Staff Directory |
| Tech/Infra (Shalaw) | Task Tracker + Staff Directory |
| Staff | Staff Directory only |

Roles defined in Supabase, enforced by RLS. Atlas designs this in detail during build.

---

## Timeline Estimate

| Week | Deliverable |
|------|-------------|
| 1 | Auth setup, RBAC schema, audit log foundation, Archos wires DNS + nginx + CF Access |
| 2 | Staff Directory + Document Management |
| 3 | Task Tracker + Regulatory Compliance Tracker |
| 4 | Integration testing, Sentinel monitoring, deploy to `my.mdb.iq` |

4 weeks to a working V1, assuming data residency decision is made before Week 1 starts.

---

## ⚠️ Open Decision: Data Residency

**This is the single question that must be answered before build starts.**

Supabase hosts new projects on AWS `us-east-1` (US East) by default. For an Iraqi bank seeking a CBI operating licence, CBI may require that operational and regulatory data be stored within Iraq or within defined jurisdictions.

**Options:**

| Option | Residency | What it means |
|--------|-----------|---------------|
| A — Supabase EU | Frankfurt (`eu-central-1`) | Same city as VPS. Likely the cleanest option if CBI accepts EU. Atlas can create the project in EU on day 1. |
| B — Supabase US | Virginia (`us-east-1`) | Default. Fine if CBI has no data residency requirement for internal staff portals. |
| C — Different stack | Iraq or self-hosted | If CBI requires Iraq-only data, Supabase is not viable. Would require a different database. Major scope change. |

**Horizon's assessment:** Option A (Supabase EU / Frankfurt) is likely the safest choice short of self-hosting. Aligns with the VPS location. CBI legal review should confirm whether this satisfies any data residency requirement.

**Amanj's call:** Does CBI have a data residency requirement for internal staff portals? If unknown, recommend getting a quick legal opinion before committing. If the answer is no requirement or EU is fine — Option A, and build starts.

---

## Division of Work

| Owner | Scope |
|-------|-------|
| **Atlas** | Next.js app, Supabase schema + RLS, all 4 modules, CI/CD pipeline |
| **Archos** | DNS (`my.mdb.iq`), nginx config, Cloudflare Access setup, Sentinel monitoring |
| **Amanj** | Data residency decision, MLRO access policy sign-off, CF Access email list |

---

## BIR Status

| Check | Status |
|-------|---------|
| Stack (Next.js + Supabase) | ✅ Green — cap-hub proven, L38 direct path |
| Infrastructure (Frankfurt VPS) | ✅ Green — Archos confirmed |
| Security baseline | ✅ Green — designed in from day 1 |
| RBAC design | ✅ Green — planned upfront |
| Build pipeline | ✅ Green — GitHub Actions, no prod npm install |
| Sentinel monitoring | ✅ Green — day 0 |
| Data residency | ⚠️ Open — needs Amanj/legal decision |

---

*Proposal by Atlas · reviewed by Horizon (product) and Archos (infra/security) · 2026-07-14*
