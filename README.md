# Awaken POS

A multi-tenant, cloud-hosted point-of-sale and retail management system for small and mid-sized
retail shops in Sri Lanka — architected, built, tested and deployed by me as the sole developer.

**Live product:** [pos.awakendev.com](https://pos.awakendev.com)

> **This is a portfolio and documentation repository.** The production application is
> closed-source. No application code, database schema or infrastructure configuration is
> published here.

---

## Overview

Most small retailers here run on paper, spreadsheets, or offline software locked to a single
machine. The alternatives are priced for businesses ten times their size. Awaken POS is a
browser-based system a shop can open on a desktop till, a tablet or a phone.

It covers the operational day of a retail shop end to end:

- ringing up sales at a till, with barcode scanning and printed receipts
- tracking stock as costed batches, so reported margin reflects what was actually paid
- selling on credit and collecting against outstanding balances
- receiving supplier deliveries and tracking what is owed
- reporting on sales, cost, profit, tax and stock value
- controlling what each member of staff can see and do

**Who it is for:** independent retail shops and small chains — typically an owner plus a handful
of staff, often on modest hardware and mobile connections.

**Built for its market:** LKR currency throughout, an English/Sinhala interface, Sinhala product
search, and receipts sized for the thermal printers shops here actually buy.

---

## Key Features

A full breakdown, including what is partial and what is planned, is in
**[docs/features.md](docs/features.md)**.

**Point of sale** — till screen with live product search; barcode scanning through hardware
scanners, device camera and manual entry; line and cart discounts with recorded price overrides;
cash and credit payment; per-shop sequential receipt numbering; duplicate-submission protection;
void with a separate authorisation step recording both parties.

**Catalogue** — products, categories and sub-categories; unit-priced and loose (weight/volume)
goods; category-driven custom product fields inherited down the category tree; barcode and QR
label printing; image optimisation on upload; Sinhala and transliterated search.

**Inventory** — stock held as costed batches with purchase and expiry dates, consumed
oldest-first; CSV and manual stock intake recorded as auditable imports; manual stock adjustment;
stock history, low-stock and expiry reporting; automated consistency checking.

**Customers & suppliers** — credit accounts with outstanding balance tracking and payment
allocation; supplier records, delivery history and payables.

**Reporting** — ten implemented reports covering sales, cost, profit, stock valuation, tax and
business summaries, plus data exports as CSV. Heavier reports run on a background queue.

**Tax** — per-product and per-category classification with inheritance and bulk reclassification;
tax-inclusive pricing; tax figures frozen onto the sale record so historical documents never move
when a rate later changes.

**Access control & audit** — shop-defined roles with a seniority ordering; per-feature permissions
enforced in the request pipeline rather than only hidden in the navigation; delegation bounded by
what the granting user holds. Completed sales are append-only — a void is the only permitted
correction, and it is itself recorded.

**Platform** — subscription plans gating features and staff seats, trials and renewals, generated
PDF invoices, and an operator administration area.

**Security** — session authentication, Google OAuth sign-in, email OTP multi-factor authentication
for administrator accounts, request rate limiting, a full security header set including a
nonce-based Content Security Policy, and correct client identification behind a reverse proxy.

**Operations** — scheduled backups with integrity verification and a rehearsed restore procedure,
a system health check suite, and external monitoring independent of the application host.

**Localisation** — English and Sinhala interfaces, with a deliberate boundary between system text,
which is translated, and shop-authored content such as product and customer names, which is not.

---

## Architecture

Full write-up: **[docs/architecture.md](docs/architecture.md)**.

Multi-tenancy is enforced at the database boundary rather than by query filtering. Platform-wide
records — shops, user accounts, roles, subscription plans — live in a master database. Each shop's
operational data — products, sales, stock, customers, activity logs — lives in a separate shop
database, and a request binds to the correct one for its lifetime.

```
        master database                    shop databases
  +-------------------------+      +---------------------------+
  |  Shops       Users      |      |  Products    Sales        |
  |  Roles       Plans      |  X   |  Stock       Customers    |
  |  Billing     Access     | ---> |  Suppliers   Activity     |
  +-------------------------+      +---------------------------+
                    no queries join across this line
```

Because a query cannot join across that line, one shop reading another shop's data is a
design-time impossibility rather than a runtime risk that depends on every developer remembering
a filter. Capacity is added by adding shop databases, without redistributing existing shops.

A layered request pipeline establishes identity, account status, shop membership, database
binding, subscription validity, per-feature permission, locale and response security headers —
in that order — before a controller runs. Controllers stay thin; work that touches money or stock
lives in dedicated, individually testable service classes.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Language & framework | PHP 8.2, Laravel 12 |
| Frontend | Server-rendered Blade templates, vanilla JavaScript, custom token-based CSS design system |
| Build | Vite |
| Database | MariaDB 11, separated into master and per-shop databases |
| Cache & queue | Redis |
| Web tier | nginx with PHP-FPM and opcache |
| Containers | Docker and Docker Compose, multi-stage image builds |
| Testing | PHPUnit, Mockery, Faker |
| Code style | Laravel Pint, enforced in CI |
| Authentication | Laravel session auth, Google OAuth, email OTP MFA for administrators |
| Barcode | WebAssembly in-browser camera decoding, plus hardware scanner and manual paths |
| Documents | Server-side PDF generation |
| Storage | Local disk and cloud object storage for backups |
| CI/CD | GitHub Actions |
| Edge | Cloudflare |

**A deliberate choice: no SPA framework.** The target users are on modest hardware and mobile
data. A server-rendered application with targeted JavaScript loads faster on those devices, and it
removes an entire category of client-state bugs from screens that handle money.

---

## Engineering Highlights

Detail on each: **[docs/engineering.md](docs/engineering.md)**.

- **Idempotent checkout** — a retried, double-tapped or network-interrupted sale resolves to the
  transaction that already committed rather than creating a second one, including when two
  requests race.
- **Concurrency-safe receipt numbering** — receipt numbers must be unique *and* gapless, because a
  missing number is indistinguishable from a sale someone deleted. Solved with atomic allocation
  bound to the sale's own transaction, and verified under simulated multi-till load.
- **Oversell prevention** — stock is drawn down under row-level locking inside the sale
  transaction, so two simultaneous sales cannot both pass the same availability check.
- **FIFO cost basis** — stock is consumed oldest-batch-first and each sale line records the cost it
  actually consumed, which is what makes profit reporting real rather than estimated.
- **Consistent lock ordering** — several code paths write the same pair of tables in one
  transaction; a single ordering rule prevents deadlocks under concurrent use, with an automated
  check that fails the build if new code takes them in the other order.
- **Time-zone-aware reporting** — the application runs in UTC while shops trade in a different
  zone. Reporting periods cut at the shop's day boundary, not the server's, handled in one place
  and guarded by an automated rule.
- **Centralised money arithmetic** — rounding lives in a single helper, with a build-time check
  that prevents hand-rolled copies reappearing.
- **Sinhala and Singlish search** — product names must match anywhere within a string, which rules
  out full-text indexing; the pattern-matching approach is documented so it is not later
  "optimised" into something that breaks the primary market's till.
- **Architectural guardrail tests** — automated checks that scan the codebase and fail the build
  when a rule is broken, so design decisions survive future changes instead of eroding.

---

## Testing

Full write-up: **[docs/testing.md](docs/testing.md)**.

PHPUnit, run inside the same container image as the application against dedicated test databases,
with layered guards that make it structurally difficult for a test run to touch real data.

Most recent full run: **1,638 tests, 9,635 assertions, passing**, across 268 test files. The
largest groups cover financial correctness, security and access control, and shop-facing
behaviour. Alongside those sit architecture and convention tests — build-time rules about money
handling, time handling, lock ordering and localisation coverage.

I do not claim a line-coverage percentage, because I do not measure one.

Two practices worth naming. **Mutation checking:** for anything touching money or access control,
I deliberately break the code and confirm the test fails — a passing test proves nothing until it
has been shown capable of failing. **Predicted deltas:** before a full run I predict the exact
test and assertion change and reconcile against the result, because two totals matching afterwards
cannot show that an existing test quietly flipped.

The system has also been **load tested** under simulated concurrent tills, which is what surfaced
a concurrency defect that single-threaded tests structurally could not have found.

---

## Deployment

Detail: **[docs/deployment.md](docs/deployment.md)**.

The application is containerised and deployed to a Linux VPS behind Cloudflare, with the web tier,
application, scheduler and queue workers running as separate services.

- **Separate staging and production environments.** Every change is verified on staging first.
- **CI on every push** — code style, dependency security advisories, database migrations and the
  full test suite.
- **Automated deployment**, ending in a health check and a transactional smoke test, so a green
  deploy means more than "the container started".
- **Monitoring** — scheduled health checks, external uptime monitoring independent of the
  application host, and alerting on failure.
- **Backups** on a schedule, integrity-verified, with a documented and rehearsed restore procedure.

Infrastructure identifiers, hostnames, credentials and environment configuration are deliberately
not published.

---

## Current Status

**The product is live and currently being trialled by a real business**, alongside a separate
staging environment used to verify every change before release. It is actively maintained, with
ongoing feature and defect work.

The system is built for a deliberately modest initial scale, with a data architecture that allows
growth without re-architecting. I have chosen not to build for scale that does not exist yet.

Known defects are tracked in a maintained internal register with severities and an agreed fix
order, and that register is where work comes from. I consider an honest list of known-but-unfixed
issues to be a sign of a maintained system rather than something to hide.

---

## My Role

I am the primary and sole developer of Awaken POS. Working independently, I was responsible for
the architecture, the full implementation, the test suite, debugging, third-party integration, and
deployment and operations.

I use AI coding tools extensively as part of my development workflow. I direct the work, make the
architectural and product decisions, and personally review, test and integrate what those tools
produce. I take responsibility for the software that ships.

Honestly stated: I am an early-career developer. I have not worked on a team of engineers, and I
have not operated at large scale. What I have done is take a real business problem from nothing to
a working, deployed system that a business relies on day to day — and keep it running, including
learning first-hand what breaks under concurrent load, what an unverified backup is actually
worth, and why a test suite can be green for the wrong reason.

---

## Screenshots

Screenshots are not yet published. Placeholders and intended captions are listed in
[docs/screenshots/](docs/screenshots/).

| Screen | Status |
|---|---|
| Point of sale | Pending |
| Dashboard | Pending |
| Product management | Pending |
| Stock batches and intake | Pending |
| Reports | Pending |
| Sinhala interface | Pending |
| Roles and permissions | Pending |

A guided walkthrough of the live system can be arranged on request.

---

## Why I Built It

I started this because the shops I know were running on paper and spreadsheets, and an owner who
wants to know last month's actual profit should not have to reconstruct it by hand.

What I wanted to prove to myself was that I could build the whole thing — not a demo or a tutorial
project, but a system that takes money, holds a business's records, and has to be right every
time, because a real shop depends on it. That meant learning things a tutorial does not cover:
that a race condition presents as a flaky till, that a backup nobody has restored is a guess, that
a passing test can pass for the wrong reason, and that the hard part of a feature is usually the
case you did not think of.

The product is live, and I am still learning on it.

---

## Contact

**Nishal Dilanga Ranasinghe** — [GitHub](https://github.com/NDsasuke)

Open to software engineering roles and freelance work. Happy to walk through the architecture, the
engineering decisions, or a live demo of the product.

---

*This repository contains documentation only. Awaken POS itself is proprietary and its source code
is not published.*
