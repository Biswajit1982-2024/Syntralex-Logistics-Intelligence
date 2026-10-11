# Freight Control Tower

A full-stack implementation of the Sea/Air/Road Freight Management BRD —
all 11 functional modules, the Dataverse-equivalent data model, the
validation/business-rule matrix, the approval hierarchy, the RFQ scoring
engine, and role-based access, built as a standalone web app.

## Why this instead of Power Apps / Dataverse / SAP / ICEGATE

The BRD specifies a Microsoft Power Platform solution (Power Apps, Dataverse,
Power Automate, SAP MM/TM, ICEGATE, Power BI, Entra ID). Those are licensed
enterprise systems this environment can't provision. This app reproduces the
**same functional behavior** — screens, data model, validations, workflow,
approvals, scoring, SLAs, escalations — as a self-contained web application,
so it can be run, demoed, and tested immediately. Where the BRD depends on a
real external system, this app uses a clearly-marked stand-in (see
"Integration stubs" below) rather than silently skipping that requirement.

## Running it

Requirements: **Node.js 22.5+** (uses the built-in `node:sqlite` module —
no `npm install` needed, zero external dependencies).

```
node server/server.js
```

Then open **http://localhost:4000** in a browser. The database is created
and seeded automatically on first run at `server/data/freight.db`. Delete
that file (and the `-wal`/`-shm` files next to it) to reset to a clean
seeded state.

To change the port: `PORT=5000 node server/server.js`.

### Logging in

This is real authentication, not a demo toggle: **every account uses the
password `Freight@123`**, hashed with scrypt server-side — there is no
header or query param that lets you claim a role without logging in as it.
The login screen lists all 8 seeded demo accounts (one per role) for
convenience. Sessions are server-side, cookie-based, and expire after 12
hours or on logout.

## How it's organized

```
server/
  db.js            SQLite schema (mirrors Appendix A field dictionary)
  seed.js          Master data + mock SAP POs + demo users
  server.js        HTTP server: routes /api/* and serves /public
  util.js          Shared helpers (ids, dates, response helpers)
  modules/
    freightRequest.js   Module 1 — validation per Appendix B.1
    approval.js         Module 2 — tiered approval hierarchy (B.2)
    rfq.js              Module 3 — RFQ + weighted scoring engine (B.3)
    shipment.js         Module 4/5 — booking + status-flow validation (B.4)
    container.js        Module 6 — ISO validation, demurrage/detention (B.5)
    boe.js              Module 7 — BOE lifecycle + SLA escalation (B.6)
    customs.js          Module 8 — clearance milestone sequencing (B.7)
    delivery.js         Module 9 — POD validation (B.8)
    grn.js              Module 10 — SAP check + aging escalation (B.9)
    invoice.js          Module 11 — 3-way match + variance approval (B.10)
    masters.js          Plant/Vendor/Forwarder/Port masters + mock SAP MM
    security.js         Role → permission matrix (B.12)
    notifications.js    In-app notification feed (stands in for Power Automate)
    audit.js            Audit_Log writer/reader
    kpi.js              Live KPI_Daily_Snapshot equivalent
    workflow.js          Mode-aware required-modules + end-to-end timeline (Shipment 360)
    cost.js               Freight cost rollup — Total Landed Cost + variance, per shipment & company-wide
public/
  index.html, app.js, styles.css   Vanilla-JS single-page frontend, no build step
```

## Single source of truth: mode-aware workflows & freight cost

This app doesn't just reskin one generic flow for all three transport modes —
Sea, Air and Road genuinely need different steps, and the system now makes
that explicit end to end instead of leaving it implicit in scattered
validations:

- **`server/modules/workflow.js`** is the one place that decides which
  sub-modules apply to a given shipment: Sea needs Container tracking + BL +
  BOE/Customs; Air needs AWB + BOE/Customs but never a container; Road
  **domestic** skips BOE/Customs entirely (nothing to clear); Road
  **cross-border** keeps BOE/Customs but still has no container. This is
  wired into the shipment's own data (`Shipment_Type` is joined in from its
  parent Freight Request in `shipment.js`, since cross-border-ness isn't a
  shipment-table field) so it can't drift out of sync.
- **`server/modules/cost.js`** is the freight **cost rollup** — the single
  source of truth for what a shipment has actually cost: Freight Cost
  (invoiced once an invoice exists, else the awarded quote), Customs Duty
  (summed from BOE filings), Demurrage and Detention (computed from container
  free-day breaches), rolled into one **Total Landed Cost** with variance
  against the original quote. `GET /api/shipments/:id/full` returns this
  alongside the mode-aware timeline and every linked record in one payload.
- **Shipment 360** (new nav: a "View Shipment 360 →" link on every shipment
  in the Shipment Tracker) is a single page per shipment combining the
  end-to-end timeline, the cost rollup, which modules apply to this mode, and
  every linked record — replacing flat per-module browsing as the way to
  answer "where is this shipment and what has it cost so far?".
- **Freight Cost Management** (new nav item) is the company-wide version:
  total freight spend / customs duty / demurrage & detention, broken down by
  mode, with a sortable per-shipment cost table (at-risk shipments
  highlighted) linking into each Shipment 360 page. `GET /api/cost-summary`.

Together these make the app usable as a **standalone freight cost & tracking
SaaS**: every shipment has one page that is its complete record, and every
cost figure traces back to the module that produced it rather than being
re-keyed by hand.

## What's implemented

- **All 11 modules** from Part C of the BRD, wired into one continuous flow:
  Freight Request → Approval → RFQ → Award → Booking → Shipment Tracking →
  Container Tracking → BOE → Customs Clearance → Delivery/POD → GRN →
  Invoice Validation.
- **Appendix A data model** as real tables (Freight_Request, Approval_Header,
  Forwarder_RFQ, Vendor_Quotation, Shipment_Master, Container_Master,
  BOE_Tracker, Customs_Clearance, Delivery_Master, GRN_Master,
  Freight_Invoice, master tables, Audit_Log).
- **Appendix B validation matrix** enforced server-side for every module
  (mandatory fields, format checks, date sequencing, duplicate checks,
  cross-field rules).
- **Approval hierarchy** (B.2): auto-routes to SCM Manager / + Logistics
  Manager / + SCM Head based on cargo value bands.
- **RFQ scoring engine** (B.3): Price 40% / Transit Time 30% / Performance
  20% / Historical Rating 10%, minimum 3 quotations enforced before award.
- **Shipment status flow** (B.4): sequential-only progression, no skipping
  stages (e.g. Booked → Delivered is rejected).
- **Container monitoring** (B.5): demurrage, detention, and ETA-variance
  calculated from recorded dates.
- **BOE SLA escalation** (B.6/B.13): 24h/48h/72h/96h breach tiers routed to
  CHA → Logistics Head → SCM Head → Procurement Director.
- **Customs clearance** (B.7): milestone date sequencing + Clearance_Days
  calculation.
- **GRN aging escalation** (B.9): Day 2 / 5 / 10 reminders.
- **Freight invoice 3-way match** (B.10): variance % auto-approve (≤5%) /
  exception (≤10%) / Finance Head (≤20%) / CFO (>20%).
- **Real authentication**: email/password login, scrypt password hashing,
  server-side sessions via HttpOnly cookies — no client-supplied identity.
- **Role-based access matrix** (B.12) **enforced server-side** in
  `security.js` and wired into every mutating route in `server.js` — a
  logged-in Requester gets a 403 from the API itself if they try to approve
  or book a shipment, not just a hidden button in the UI. Approval decisions
  additionally check that the acting user's role matches the specific
  approval level's required role (B.2).
- **Real file uploads**: POD attachments go through actual
  `multipart/form-data` handling, validated server-side against file
  bytes (PDF/JPG/PNG, ≤20MB per B.8) and stored under `server/uploads/`,
  not just a client-supplied filename string.
- **Hardening**: SQLite WAL mode for concurrent access, per-IP rate limiting
  (tighter on `/api/login` to blunt brute-forcing), and security response
  headers (CSP, X-Frame-Options, X-Content-Type-Options) on every response.
- **Audit log** of every create/update/decision across all modules.
- **In-app notifications** reproducing the Appendix C event catalogue
  (Request Submitted, Approval Pending, RFQ Released, Booking Confirmed,
  Shipment Delay, Demurrage/Detention Risk, BOE SLA breach, GRN aging, etc).
- **Live KPI dashboard** (Appendix A.13 KPI_Daily_Snapshot, computed on read).

## Integration stubs (where a real enterprise system is required)

These are implemented as clearly-labeled mocks so the workflow logic that
*depends* on them is still fully exercised — swap the stub for a real
connector when deploying against actual systems:

- **SAP MM** (`masters.js: sapGetPO/sapPoValid`) — a `SAP_PO_Mock` table
  stands in for the EKKO/EKPO/LFA1 inbound interfaces; PO validation logic
  (Released status, not closed, supplier match) runs against it exactly as
  B.11 specifies.
- **ICEGATE / Entra ID / Power BI refresh** — not simulated; BOE filing is
  handled natively by this app's own BOE module, and login is this app's
  own email/password + session system (see "Going further" below for the
  real-SSO path).
- **Notifications** — delivered as an in-app feed instead of Outlook/Teams;
  the event catalogue and recipient logic from Appendix C are intact, only
  the delivery channel differs.
- **Database** — `node:sqlite` (zero-install, but single-process). A
  drop-in Postgres schema is provided at `server/postgres-schema.sql`
  (tested against a real Postgres 16 instance) with a migration note at the
  top of that file for when you deploy with real package-registry access.

## Going further (production checklist)

- **Auth**: replace the local email/password login with your real identity
  provider (Entra ID via OIDC, matching the BRD) — `modules/auth.js` is the
  one file that owns session/credential logic, so this is a contained swap.
- **Database**: move to `server/postgres-schema.sql` + the `pg` driver once
  you have npm access (see the migration note at the top of that file).
- **File storage**: point `Uploaded_File.Stored_Path` at S3/Azure Blob
  instead of local disk.
- **Notifications**: wire `modules/notifications.js`'s `notify()` calls to
  Microsoft Graph (Outlook/Teams) instead of just writing to the
  `Notification` table.
- **TLS**: this demo serves plain HTTP; terminate TLS in front of it (reverse
  proxy/load balancer) and uncomment the `Strict-Transport-Security` header
  noted in `server.js`, plus add `; Secure` to the session cookie.
- **SAP / ICEGATE**: swap `modules/masters.js`'s `sapGetPO`/`sapPoValid`
  mock for your real SAP MM/TM integration; the validation logic calling
  them doesn't need to change.

## Demo walkthrough

1. Open the app and log in — the login screen lists all 8 demo accounts,
   each with password `Freight@123`. Start as **Requester** (Ravi Kumar).
2. Go to **Freight Requests**, submit one (pick any seeded Plant/PO/Supplier).
3. Log out, log back in as **SCM Manager** → **Approvals** → Approve.
   (For cargo value over ₹5L you'll also need Logistics Manager, and over
   ₹25L, SCM Head — log in as each in turn; the API itself rejects an
   approval decided by the wrong role, not just the UI.)
4. **RFQ & Forwarders** → issue an RFQ against the approved request, submit
   3+ quotations from different forwarders, then award the top-ranked one.
5. **Shipment Tracker** → book the shipment from the awarded RFQ, then
   advance it stage by stage.
6. **Container Tracker**, **BOE Tracker**, **Customs Clearance**,
   **Delivery & POD**, **GRN Tracker**, **Invoice Tracker** — work through
   each in order, referencing the shipment you just booked. (For a domestic
   Road shipment, skip BOE/Customs — the Shipment 360 timeline will show
   those steps as "not applicable" instead of pending.)
7. From **Shipment Tracker**, click **View Shipment 360 →** on your shipment
   to see the single combined page: timeline, cost rollup, linked records.
8. Check **Freight Cost Management** for the company-wide cost rollup across
   every shipment, and **Dashboard** for live KPIs / **Notifications** for
   the event trail the workflow generated.
