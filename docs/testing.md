# Testing

How correctness is verified on a system that handles other people's money.

---

## Scale and tooling

PHPUnit, run inside the same container image as the application so that tests execute against the
same PHP version, extensions and database engine as production.

Most recent full run:

| | |
|---|---|
| Tests | **1,638** |
| Assertions | **9,635** |
| Test files | **268** |
| Result | Passing |

There is no line-coverage figure here because I do not measure one, and quoting a number I cannot
substantiate would be worse than omitting it. What follows is a description of what is actually
covered.

---

## Test database isolation

Tests run against dedicated test databases, never development or production data. Because the
system uses multiple databases, the suite has to set up and reset a set of them rather than one.

Several independent guards make it structurally difficult for a test run to reach real data — the
configuration, the database naming, and a runtime check all have to agree before a destructive
test operation is permitted. This is deliberate belt-and-braces: the failure mode being prevented
is irreversible.

Between tests, only the tables a test actually touched are reset. This started as a correctness
measure and turned out to be the single largest performance factor in the suite: resetting every
table in every database before every test dominated total runtime, and narrowing it reduced a full
run from roughly forty minutes to under ten. The same change had a substantial effect on CI, which
had not been predicted — the cost had been attributed to a local-environment quirk that explained
the local number so completely that the same cost elsewhere went unexamined.

---

## What the suite covers

| Area | Coverage |
|---|---|
| **Financial** | Checkout, stock allocation, totals and discounts, voids and their refusal cases, credit payments, tax calculation and disclosure. The largest group in the suite. |
| **Security & access** | Authentication paths, permission enforcement, tenant isolation, role seniority and delegation limits, proxy trust, session handling. |
| **Shop features** | Screen-level behaviour across catalogue, stock, customers, suppliers, quotations and reports. |
| **Architecture & conventions** | Build-time rules: money handling, time handling, lock ordering, naming, localisation coverage. |
| **Health & observability** | Every health check, including verification that each can actually report failure. |
| **Concurrency** | Lock contention and wait behaviour. |
| **Multi-tenancy** | Database separation and per-request resolution. |
| **Design system** | CSS and layout conventions across shared components. |
| **Mail, backup, console** | Templates, backup storage drivers, scheduled commands. |

---

## Three base classes, chosen deliberately

Tests use one of three bases depending on what they genuinely need: a full base that prepares the
databases, a lighter one that boots the application without database setup, and a bare one that
boots nothing.

Choosing the light base for something that later starts querying is a silent failure — with no
database preparation behind it, the test reads whatever the previous test left. An automated check
fails the build when a test using a light base touches a database, a factory or an HTTP helper.

---

## Mutation checking

For anything touching money or access control, I deliberately break the code and confirm the
relevant test fails.

**A passing test proves nothing until it has been shown capable of failing.** This has repeatedly
caught tests that passed against the exact defect they were written to detect. Recurring patterns:

- A test whose evidence is an *absence* — no error logged, no access granted — passing because its
  instrument was dead and it was asserting against an empty result.
- A refusal test passing for the wrong reason, refused by a mechanism other than the one under
  test, so the real guard was never exercised.
- A guard scanning for a forbidden pattern that could never match anything, and therefore could
  never fail, regardless of what the code did.

Where every assertion in a file is an absence, the file carries a control test asserting that the
thing being scanned for genuinely exists somewhere — otherwise a scan that matches nothing at all
passes the entire file.

---

## Predicted deltas

Before running the full suite after a change, I predict the exact number of tests and assertions
it should move by, then reconcile against the result.

This catches something two totals cannot. A new test passing while an existing test silently flips
from asserting one thing to asserting another leaves the totals looking reasonable. A prediction
that lands exactly is evidence that nothing unrelated changed behaviour — which matters most for
changes with a wide blast radius that the diff does not reveal, such as edits to shared test
setup, a widely used factory, or a dependency lock file.

---

## Load testing

The suite runs single-threaded, so an entire category of defect is invisible to it: anything
requiring two transactions in flight simultaneously.

A custom load driver simulates multiple concurrent tills completing real sales through the real
application. This is what surfaced a significant concurrency defect in receipt number allocation
(described in [engineering.md](engineering.md)) that no amount of unit testing would have found.

Two practices carried from that work:

- **Reconcile after a passing run, not only a failing one.** A green load test is only meaningful
  if stock levels, orphaned records and sequence contiguity are checked afterwards. Otherwise
  "100% of sales completed" says nothing about whether they completed *correctly*.
- **Verify the fix did not simply remove the conditions.** After changing both the code and a
  dependency version, a passing run could mean the defect was fixed or that the circumstances
  producing it no longer existed. Both were checked separately.

---

## Continuous integration

Every push runs code style, a dependency security advisory audit, database migrations for both
database types, the full test suite, and a cache connectivity check.

Style runs first deliberately. A formatting failure then costs about ninety seconds instead of
failing after the full suite has run.
