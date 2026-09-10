# Engineering Highlights

Problems this system actually ran into, and how they were solved. Described at a conceptual level
— the reasoning, not the implementation.

---

## Never charging a customer twice

A till runs on shop wi-fi. Requests time out, cashiers tap twice, browsers retry. Any of those can
turn one sale into two, and the shop finds out when a customer disputes their bill.

Every checkout carries a client-generated unique key. If a request arrives with a key that has
already been committed, the system returns the existing sale rather than creating another one.

The interesting case is not the retry — it is the race, where two requests arrive close enough
together that both check for an existing sale, both find nothing, and both proceed. A check alone
cannot solve that, because the gap between checking and writing is exactly where the second
request slips through. The database's own uniqueness guarantee is what settles it: the loser of
the race is rejected at write time, and that rejection is caught and resolved to the sale that
won, rather than surfacing to the cashier as an error.

**The lesson worth stating:** a uniqueness check in application code is an optimisation. The
constraint is the correctness guarantee.

---

## Receipt numbers that are unique *and* gapless

Receipt numbers are sequential per shop, per trading day. They have two requirements that pull in
opposite directions.

They must never repeat — a duplicate receipt number is a dispute that cannot be resolved after the
fact. And they must never skip, because an owner reconciling the day's takings cannot tell a
missing receipt number from a sale a cashier rang up, pocketed and deleted. The second requirement
is the one people get wrong, because gaps look harmless.

Gaplessness rules out the easy implementation. Allocating a number in its own short transaction is
fast and contention-free, but the number is burned if the sale then fails. So the counter has to
advance inside the sale's own transaction, where a rollback takes the increment back with it.

That decision puts a contended row in the middle of the checkout path, which is where it got
interesting. Load testing with multiple simultaneous tills showed a significant proportion of
concurrent sales being refused outright — not slowed, refused — because of how the database
handles a transaction that waits for a row another transaction has since modified. Reordering the
statements or rewriting them in a different shape did not help; several plausible variants were
measured head to head and all failed the same way.

What resolved it was making allocation a single atomic operation, taken as the transaction's very
first database access. Re-run under the same load, sale completion returned to 100%, and receipt
sequences were confirmed contiguous afterwards.

**Two lessons.** A defect that needs two transactions in flight cannot be found by a
single-threaded test suite, however large — it needed a load test to exist at all. And when
several variants of a fix all fail identically, the variable is not the one being changed.

---

## Preventing oversell

Two cashiers selling the last unit at the same moment must not both succeed.

Stock is drawn down inside the sale's transaction, with row-level locks taken on the specific
stock records being consumed. The second transaction waits for the first to commit and then sees
the true remaining quantity, rather than both reading the same pre-sale figure and both deciding
there is enough.

The subtlety is that the lock must be taken on the same connection the transaction was opened on.
A lock acquired on a different connection is released as soon as the query returns, which looks
identical in code and provides no protection whatsoever.

---

## FIFO cost basis

Stock is not a single number per product. It is a series of batches, each with the quantity
received, what it cost, when it was purchased and optionally when it expires. Sales consume the
oldest batch first.

This is what makes profit reporting real. A shop that bought a product at three different prices
over three months has a genuine cost for each unit sold, and each sale line records the cost of
the batch it actually drew from. Averaging that away, or using the current purchase price, gives a
margin figure that is plausible and wrong.

A consequence worth designing around: one line in the cart can consume several batches, so it
becomes several rows in the sale record. Every screen and export that reads sale lines has to
understand that split.

The ordering rule is deliberately frozen at purchase date, never expiry date. Selling the
soonest-to-expire batch first is a different and defensible policy — but it is a *different*
policy, and switching to it would change the cost basis of transactions already recorded.

---

## Consistent lock ordering

Several code paths write to both the stock records and the product records inside a single
transaction — checkout, void, manual adjustment. They had grown up independently, and they did not
agree on which table to touch first.

Two transactions that take the same two locks in opposite orders will eventually deadlock, and the
victim is whichever user arrived second. It presents as an intermittent error on a till, which is
close to undiagnosable from a support call.

There is now one ordering rule, set by the checkout path because that is the operation that must
never fail, and an automated check fails the build if new code takes the locks the other way
round. Ordering is also applied *within* a transaction, so that two operations touching the same
set of products cannot deadlock against each other either.

---

## Time zones: the shop's day, not the server's

The application runs in UTC. The shops trade several hours ahead of it. "Today's sales" therefore
has two possible meanings, and only one of them is the one a shopkeeper means.

Getting this wrong is quiet. Every report still returns numbers; they are simply cut at the wrong
moment, so the first hours of trading are filed under the previous day. Nobody notices until
someone reconciles a till by hand.

All reporting windows are cut at the shop's own day boundary, through a single time abstraction.
An automated convention check fails the build when new code asks the platform what day it is in a
context that means a shop's day.

There is a second trap on the other side of it. A correctly zoned timestamp handed to the database
layer can be formatted using the *application's* clock rather than converted — producing an error
of the same size in the opposite direction, with numbers just as plausible. The abstraction
normalises on the way out, and the tests assert through the database layer rather than by
inspecting a timezone, because the timezone is not what goes wrong.

---

## Money arithmetic in one place

Rounding of monetary values had been written by hand in a large number of places. Every copy was
individually correct and there was no guarantee they would stay that way, or that the next one
would be.

It is now a single helper, and a build-time check refuses to let a hand-rolled copy reappear. The
same treatment was applied to two other quantities that had drifted the same way — one of which
had *already* diverged, producing two different renderings of the same value on two screens an
administrator reads side by side.

**The lesson:** when consolidating several copies of a rule, run them all against the same inputs
first. Reading them tells you they look the same. Running them tells you whether they are.

---

## Search that works in Sinhala

Product names are entered in Sinhala, in English, and in Sinhala transliterated into Latin
characters — often inconsistently within one shop. Search has to match part-way into a string,
because a shopkeeper types the distinctive part of a name rather than its beginning.

Full-text indexing cannot do leading-wildcard matching, and handles Sinhala poorly. Search
therefore uses pattern matching deliberately, with the reasoning documented prominently, because
the change to a full-text index looks like an obvious optimisation to anyone who has not read why
it is not one — and it would silently break search for the primary market.

Query cost is managed instead by caching catalogue projections, bounding result sets, and keeping
the till's search path narrow.

---

## Alerting that is proven, not assumed

The alert transport originally discarded its own delivery result. A dead alert channel and a
perfectly healthy system produced identical evidence: silence.

That is worse than having no alerting, because it answers the question. Every health verdict in
the system sat downstream of it.

Delivery outcomes are now recorded, the transport is actively probed, and the alerting chain has
been deliberately driven to failure to confirm that a real problem reaches a human. The same
principle applies to the health checks themselves: each one has been shown capable of reporting
failure before being trusted to report success.

**A health check that cannot fail is worse than no health check**, for the same reason. It answers
the question.

---

## Backups that are verified

Backups run on a schedule to configurable destinations and are integrity-verified rather than
assumed complete. The restore procedure is documented and has been rehearsed, because an
unrestored backup is a hypothesis.

Backup freshness is itself a monitored condition with a threshold. An earlier version reported
success while describing a backup that was weeks old — technically accurate, and useless.

---

## Architectural guardrails as tests

Alongside behavioural tests, the suite contains checks that scan the codebase and fail the build
when an architectural rule is violated: money rounded by hand, a report using the platform's
clock, locks taken in the wrong order, an authentication entry point added without the
corresponding revocation check, an untranslated string on a screen that has been localised.

This is the mechanism by which design decisions survive. Documentation describing a rule is a
request. A test enforcing it is a constraint. Every one of these checks exists because the rule it
protects had already been broken at least once, and writing the check was cheaper than finding the
next violation in production.

One caveat learned the hard way: a scanning check must be proven able to fail before it is
trusted. A rule that silently matches nothing looks exactly like a rule nothing violates.
