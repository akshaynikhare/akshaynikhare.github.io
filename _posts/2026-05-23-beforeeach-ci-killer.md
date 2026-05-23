---
title: "How a Single beforeEach Killed Our CI for 36 Hours"
date: 2026-05-23 09:00:00 +0530
categories: [Engineering, Testing]
tags: [ci-cd, testing, postgresql, github-actions, node, performance]
description: A single line of test setup code turned a 2-minute test suite into a 6-hour timeout. Here's the math, the root cause, and the fix.
---

Six failed CI runs. Thirty-six hours of GitHub Actions time. Every run timing out at exactly the 6-hour limit.

The culprit was one line in `tests/setup.js`.

## The Setup

We were building a multi-tenant platform with a PostgreSQL backend — around 76 database models handling everything from user accounts and billing to visitor logs and real-time notifications. The test suite had grown to roughly 1,140 test cases across 36 files.

Standard stuff. CI ran on every PR. Tests passed locally. And then one day, CI just... never finished.

## The Anti-Pattern

Here's what the test setup looked like:

```javascript
// tests/setup.js
beforeEach(async () => {
  const tableNames = await getTableNames(); // 76 tables
  await sequelize.query(
    `TRUNCATE TABLE ${tableNames.join(', ')} CASCADE;`
  );
});
```

The intent was clean isolation — every test starts with a blank slate. Reasonable in theory. Catastrophic in practice.

## The Math

Do the multiplication:

```
76 tables × 1,140 tests = 86,640 TRUNCATE operations
```

Each `TRUNCATE TABLE ... CASCADE` is not a cheap operation. PostgreSQL has to:

1. Acquire exclusive locks on all referenced tables
2. Walk the foreign key graph to find dependent tables
3. Truncate each in dependency order
4. Release locks

With a moderately complex schema where most tables reference others (users → societies → members → invoices → payments → ...), a single `TRUNCATE ... CASCADE` on a central table can fan out into dozens of implicit truncations.

Multiply that by 86,640 and you have a test suite that will never complete within any reasonable timeout.

## Why It Wasn't Caught Sooner

Two reasons:

**1. It used to be fast.** When the suite had 50 tests and 20 tables, this pattern worked fine. `50 × 20 = 1,000` truncations — uncomfortable but survivable. Nobody noticed when the suite crossed a tipping point.

**2. Local runs used a different database state.** Locally, developers often ran a subset of tests with `--grep` or file-specific runs. The full suite was only ever run on CI, and CI was slow enough that most assumed it was just a "CI is slow" problem rather than a runaway complexity issue.

## The Fix

Three changes, applied together:

### 1. Move sync to `beforeAll`, not `beforeEach`

```javascript
// tests/setup.js — AFTER
beforeAll(async () => {
  await sequelize.sync({ force: true }); // once per suite, not per test
});
```

Tables are created once at the start of the suite. No teardown between tests.

### 2. Use unique identifiers in test data

Without per-test cleanup, tests can no longer share fixed data. The fix is to make every piece of test data unique:

```javascript
// BEFORE — breaks without cleanup
const user = await createTestUser({ email: 'test@example.com' });

// AFTER — safe without cleanup
const user = await createTestUser({
  email: `user_${Date.now()}_${Math.random().toString(36).slice(2)}@test.com`
});
```

Timestamps and short random suffixes make collisions statistically impossible across a test run. UUIDs work too — use whatever your helpers already produce.

### 3. Add CI timeout safety nets

The default GitHub Actions job timeout is 6 hours. That's a very long time to wait before learning something is wrong. Set explicit timeouts on test steps:

```yaml
- name: Run tests
  run: npm test
  timeout-minutes: 15
```

Now a runaway suite fails in 15 minutes instead of 6 hours. You get the signal fast and spend less Actions budget on it.

## Results

| | Before | After |
|---|---|---|
| TRUNCATE operations | 86,640 | 0 |
| Test suite duration | 6+ hours (timeout) | ~3 minutes |
| CI failure signal | After 6 hours | After 15 minutes max |

## When Transaction Rollback Is Better

If your tests don't create their own transactions internally, wrapping each test in a transaction and rolling back is faster than `TRUNCATE` and safer than the unique-ID approach:

```javascript
let transaction;

beforeEach(async () => {
  transaction = await sequelize.transaction();
});

afterEach(async () => {
  await transaction.rollback();
});
```

This works well for simpler schemas. We couldn't use it because several of our tested code paths opened their own transactions internally, which can't be nested without explicit savepoint support. The unique-ID approach was the safer choice for our case.

## The Underlying Lesson

Test isolation strategies don't scale linearly — they scale with `tests × tables`. A pattern that works at 50 tests and 20 tables can fail spectacularly at 1,000 tests and 76 tables.

If your test suite is growing and CI is getting slower, check your setup/teardown strategy before assuming you need faster hardware.
