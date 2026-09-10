# Deployment and Operations

A high-level description of how Awaken POS is built, shipped and kept running. Infrastructure
identifiers, hostnames, credentials and environment configuration are deliberately not published.

---

## Runtime shape

The application is containerised and deployed to a Linux VPS behind Cloudflare. It runs as several
cooperating services rather than one process:

- a **web tier** terminating requests and serving static assets
- the **application** itself
- a **scheduler** running recurring work
- **queue workers** processing background jobs
- **database** and **cache** services

Images are built in multiple stages, so dependency resolution and asset compilation happen in
build stages and the runtime image carries only what it needs to run.

---

## Environments

Staging and production are fully isolated environments with separate data and separate
configuration. Every change reaches staging first and is verified there before it can go to
production.

The two environments differ deliberately in a few places — production runs stricter startup checks
and caching that staging does not. That asymmetry has a cost worth naming: a production-only code
path is unproven until it has actually run. Two such steps were broken the first time they
executed for real. Both are now exercised deliberately on staging using the production form of the
command, rather than being trusted because the deploy went green.

**The rule that came out of that:** anything that only runs in production is unverified until you
have run it yourself.

---

## Continuous integration

Every push runs, in order:

1. **Code style** — a formatting failure costs about ninety seconds instead of failing after the
   whole suite has run.
2. **Dependency security audit** against known advisories for locked dependencies.
3. **Database migrations** for both database types, as a smoke check.
4. **The full test suite.**
5. **Cache connectivity verification.**

The dependency audit exists because a set of advisories once sat unnoticed in the lock file for
months. It is now a gate rather than something somebody has to remember to look at.

---

## Release process

Staging deploys automatically once a change is merged. Production deploys are triggered
deliberately, not automatically, and are restricted to a low-traffic window — the system is a
working shop's till, and a deploy is an interruption to a business day.

Deployment finishes with an **automated health check and a transactional smoke test**, so a
successful deploy means the application can actually serve and record a transaction, not merely
that a container started.

A drift check independently confirms that what is deployed is the commit that was intended.

---

## Monitoring

Several layers, chosen so that no single failure hides the others:

- **Scheduled health checks** covering database connectivity, cache, queue, scheduler liveness,
  mail transport, alert transport, disk headroom and backup freshness.
- **Heartbeats and a canary job**, because a dead scheduler or a dead queue worker is otherwise
  invisible — everything looks healthy and work simply stops happening.
- **External monitoring independent of the application host.** This one matters
  disproportionately: every other signal in the system is emitted by the machine it reports on, so
  if that machine dies, so does its ability to tell anyone. The external check is the only signal
  that survives the failure it exists to report.
- **Alerting on failure**, with delivery outcomes recorded rather than assumed.

Every health check has been deliberately driven to failure to confirm it can report a problem. A
check that cannot fail is worse than no check, because it answers the question.

---

## Backups and recovery

Backups run on a schedule to configurable destinations, including cloud object storage, and are
integrity-verified rather than assumed complete.

Backup freshness is itself a monitored condition with a threshold, after an earlier version
reported success while accurately describing a backup that was weeks old.

The restore procedure is documented and has been rehearsed. An unrestored backup is a hypothesis,
not a recovery plan.

---

## Operational discipline

Some practices adopted after specific incidents, kept because they were earned:

- **Verify on the box, not from the exit code.** A command can return success for having done
  nothing — a prompt taking its default without a terminal attached is the cheapest example. Check
  the outcome, not the return value.
- **Match the build environment to the runtime environment.** Dependency resolution performed
  against a different language version than the one that will run the code can produce a lock file
  that is valid, installable, and unable to start.
- **Rebuild every service that shares an image.** Services built from the same definition drift
  apart if only one is rebuilt, and the half-updated state is harder to diagnose than a clean
  failure, because the visible part of the system looks healthy.
- **Know the recovery time before the outage, not during it.**
