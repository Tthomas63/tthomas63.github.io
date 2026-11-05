
---
layout: ../../layouts/NoteLayout.astro
title: Separate historical backfills from deployment
description: Schema changes and historical data updates have different operational requirements. They shouldn't automatically share a deployment lifecycle.
category: DATABASES / DEPLOYMENT
---


## The problem

One recurring deployment problem I've encountered throughout my
career is coupling expensive historical data backfills to
application migrations.

The pattern is familiar. A developer introduces a new database
column and needs to populate it using existing data. They add
a data migration, test it locally, and everything works.

Then it reaches staging or production.

The local database contained a relatively small dataset.
The target environment contains years of records, additional
relationships, and workloads the developer didn't account for.
An operation that completed almost instantly in development
can suddenly take minutes or hours, hold locks, or fail.

The migration itself isn't necessarily poorly written. The
problem is that its execution characteristics weren't evaluated
against the environment where it actually matters.

**Developers shouldn't discover the operational properties of
a migration during a production deployment.**

## My preferred approach

I prefer to separate the schema change, application behavior,
and historical data processing into distinct operations.

The sequence generally looks like this:

1. Introduce the necessary schema changes, preserving
   compatibility where practical.
2. Deploy application code that can tolerate the transition
   between old and new data.
3. Execute the historical backfill independently, with
   progress visibility and a recovery strategy.
4. Verify the resulting data before removing transitional
   behavior or enforcing stricter constraints.

The backfill itself should be designed around the workload.
For substantial datasets, that may mean bounded batches,
idempotent updates, checkpoints, and explicit monitoring.


## An example: a controlled backfill in an EF application

I encountered this decision again while working on an
application using Entity Framework.

A database change required additional data processing.
Rather than couple that processing to the EF migration,
I separated the backfill into an explicitly invoked,
super-admin-only operation exposed through the application's
administrative interface.

The operation was designed to be idempotent, allowing it
to be invoked again without blindly duplicating or
reapplying completed work.

This wasn't necessarily the most elaborate solution.
A dedicated background worker, scheduled job, or separate
console application could also have been appropriate.

But the administrative operation was suitable for the
requirements and constraints of that particular application.
It made the historical data work a deliberate operation
instead of an implicit requirement of every deployment.

The implementation worked for its intended purpose, and
the additional administrative step wasn't an inconvenience
in practice.

The important decision wasn't choosing a button over a job
queue. It was separating the execution lifecycle of the
backfill from the deployment lifecycle.

## Why the separation matters

A deployment and a backfill have different failure modes.

A deployment should have a manageable release and recovery
process. A data backfill might need to run for an extended
period, pause under database load, or resume after interruption.

Separating the two makes those operational decisions
independent. It also avoids treating application startup
as an implicit batch-processing mechanism.

## Recovery and consistency

One of the more frustrating consequences of a failed
migration is the possibility of leaving environments in
different states.

Depending on the database, transaction behavior, and
operations involved, a failure may roll back cleanly
or require intervention to reconcile schema and migration
history.

I strongly prefer correcting discrepancies immediately.
Allowing migration histories and actual database schemas
to diverge can turn a single deployment failure into a
recurring source of problems.

Separating historical data processing introduces a related
but distinct concern: the schema can be current while the
data backfill remains incomplete.

That intermediate state needs to be intentional.

Application code may need to tolerate old and new data
temporarily. Backfill progress and completion should be
verifiable, and any final constraints should only be
introduced when the necessary data is ready.

The goal isn't to eliminate every transitional state.
It's to avoid uncontrolled or poorly understood states
that make subsequent deployments and recovery harder.

## Where I make exceptions

This isn't a rule against data migrations.

Small, predictable data transformations can reasonably belong
in a migration when their runtime and failure characteristics
are understood.

Likewise, some schema changes are expensive enough to require
their own operational planning.

The important distinction is the actual work being performed,
not whether Django provides a convenient place to execute it.
