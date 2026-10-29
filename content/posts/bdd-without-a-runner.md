---
title: "BDD Without a Gherkin Runner"
date: 2026-10-29
draft: true
tags: ["testing", "bdd", "swift", "ios"]
description: "Keep Gherkin scenarios as the spec, let Swift Testing do the testing, and use a small checker to keep the two from drifting apart."
---

Most teams that try BDD end up with one of two problems. Either the `.feature` files are documentation that nobody runs, and they drift from the app within a sprint. Or the team adopts a Gherkin runner and spends its time maintaining step definitions instead of tests.

On iOS the second option is harder than it sounds. There's no official Cucumber for Swift, the community runners don't fit Swift Testing, and debugging a failure means jumping from a feature line to a regex to the step code to the app.

This post describes a middle path: **keep Gherkin as the spec, write ordinary Swift tests, and use a small script to prove every scenario points to a real test.** No runner, no step definitions, and no silent drift.

## The idea in one picture

```
Specs/ShoppingList.feature      →  the contract, in plain language
        │  @covered-by / test name
        ▼
Tests/…/ShoppingListTests.swift →  the proof, in Swift Testing
        ▲
        │  checks the link exists
Scripts/check-specs.py          →  runs on every pull request
```

A scenario is never "just documentation". It either names the test that proves it, or it says why there isn't one yet.

## Anatomy of a scenario

Here's a scenario from a made-up shopping list app:

```gherkin
Feature: Shopping list

  Background:
    Given the user is signed in with a list called "Weekend"

  # SL_012
  @SL_012 @req:REQ_SL_012 @validation
  @severity:medium @priority:P2 @covered-by:AddItemViewModelTests.anEmptyNameIsRejected
  Scenario: An item with an empty name cannot be added
    When the user adds an item with an empty name
    Then the item is not added
    And the add button stays disabled
```

The tags carry all the machine-readable information:

| Tag | Purpose |
| --- | --- |
| `@SL_012` | A stable ID. Never reused or renumbered, even when scenarios are deleted. |
| `@req:REQ_SL_012` | Links to a row in a requirements register (more on that below). |
| `@validation` | One of three categories: `@happy_path`, `@validation`, `@edge_security`. |
| `@severity:` / `@priority:` | Lets you filter what matters for a release. |
| `@covered-by:Type.method` | The Swift test that proves this scenario. |

## Three ways to be "covered"

Every scenario must be in exactly one of these states.

**1. An existing test already proves it.** Point at it:

```gherkin
@covered-by:AddItemViewModelTests.anEmptyNameIsRejected
```

**2. No test exists yet, so write one named after the scenario:**

```swift
@MainActor
struct ShoppingListScenarioTests {
    // SL_014
    @Test
    func test_SL_014_aFailedSaveKeepsTheDraft() async throws {
        let store = ScriptedListStore(save: .failure(.offline))
        let viewModel = AddItemViewModel(store: store)

        await viewModel.send(.nameChanged("Milk"))
        await viewModel.send(.addTapped)

        #expect(viewModel.state.draftName == "Milk")
        #expect(viewModel.state.error == .offline)
        #expect(store.savedItems.isEmpty)
    }
}
```

The `test_SL_014_` prefix is the link, so the scenario needs no extra tag.

**3. It can't be tested yet, and the spec says why:**

```gherkin
@SL_031 @req:REQ_SL_031 @edge_security
@severity:high @priority:P1 @blocked:mock
# The mock backend cannot return a list shared by two users yet.
Scenario: A list removed by its owner disappears for everyone
```

Use a small, fixed vocabulary of statuses, such as `@automation-pending`, `@blocked:<reason>`, `@manual` and `@gap` (for behavior the product doesn't have yet). What matters is that "untested" is a **visible, deliberate state**, not an accident.

## The checker

The whole mechanism is a short script. It's not a Gherkin parser and doesn't run anything. It checks structure and that every link points somewhere real:

```python
#!/usr/bin/env python3
"""Checks that every scenario is well formed and traces to a test. Not a Gherkin runner."""
import re
import sys
from pathlib import Path

CATEGORIES = {"@happy_path", "@validation", "@edge_security"}
PENDING = ("@automation-pending", "@blocked:", "@manual", "@gap", "@pending-spec")

swift_files = [p.read_text() for p in Path("Tests").rglob("*.swift")]


def scenarios(path):
    found, tags, current, background = [], [], None, []
    for number, raw in enumerate(path.read_text().splitlines(), 1):
        line = raw.strip()
        if line.startswith("@"):
            tags += line.split()
        elif line.startswith(("Scenario:", "Scenario Outline:")):
            current = {"where": f"{path.name}:{number}", "tags": tags, "steps": list(background)}
            found.append(current)
            tags = []
        elif line.startswith("Background:"):
            current = None
        elif re.match(r"(Given|When|Then|And|But) ", line):
            (current["steps"] if current else background).append(line)
    return found


def test_exists(type_name, method):
    return any(
        re.search(rf"\b(class|struct) {re.escape(type_name)}\b", source)
        and re.search(rf"func {re.escape(method)}\(", source)
        for source in swift_files
    )


errors, seen = [], set()
for path in sorted(Path("Specs").glob("*.feature")):
    for scenario in scenarios(path):
        tags, where = scenario["tags"], scenario["where"]
        ids = [t[1:] for t in tags if re.fullmatch(r"@[A-Z]+_\d{3}", t)]
        if len(ids) != 1:
            errors.append(f"{where}: needs exactly one scenario ID")
            continue
        sid = ids[0]
        if sid in seen:
            errors.append(f"{where}: duplicate {sid}")
        seen.add(sid)
        if len(CATEGORIES & set(tags)) != 1:
            errors.append(f"{where}: needs exactly one category")
        for keyword in ("Given", "When", "Then"):
            if not any(step.startswith(keyword + " ") for step in scenario["steps"]):
                errors.append(f"{where}: missing {keyword}")
        covered = [t.split(":", 1)[1] for t in tags if t.startswith("@covered-by:")]
        for reference in covered:
            type_name, _, method = reference.rpartition(".")
            if not test_exists(type_name, method):
                errors.append(f"{where}: {reference} does not exist")
        named = any(re.search(rf"func test_{sid}_", source) for source in swift_files)
        if not covered and not named and not any(t.startswith(PENDING) for t in tags):
            errors.append(f"{where}: {sid} has no test and no pending status")

print("\n".join(errors) or "every scenario traces to a test or says why not")
sys.exit(1 if errors else 0)
```

Run it in CI, and a pull request that renames `anEmptyNameIsRejected` without updating the spec fails immediately. That's the drift problem solved at the cheapest possible point.

You can grow it from here: check that every `@req:` tag has a row in the register, that a `Scenario Outline` has an `Examples` table naming its placeholders, or print counts per category.

## A register for where each requirement came from

Next to each `.feature` file, keep a plain Markdown table:

```markdown
| Requirement | Scenario | Requirement in one line | Source |
| --- | --- | --- | --- |
| REQ_SL_012 | SL_012 | An item needs a name | Product spec, "Add item", v3 |
| REQ_SL_014 | SL_014 | A failed save keeps what the user typed | `AddItemViewModel.swift` |
```

It answers the question every reviewer eventually asks: *"Says who?"* Some behavior comes from a written spec, and some only exists because the code does it. Writing that down makes the second kind visible, and it gives you a place to record conflicts ("the spec says X, the app does Y") instead of quietly picking one.

## Balance the categories

Left alone, a spec fills up with happy paths, because they're the easiest to write. A rough target keeps it honest:

- **~20% `@happy_path`**: the journey works.
- **~40% `@validation`**: inputs, limits and rejected actions.
- **~40% `@edge_security`**: offline, slow, concurrent, unauthorized, and odd lifecycles.

Treat the target as a prompt, not a quota. If one area drifts, ask what's missing; never add filler scenarios to fix the numbers.

## Make scenarios cheap to test

Traceability only helps if writing the test is easy. The tests that prove most scenarios are **behavior tests**: one user story, through the app's real composition root, with every network call answered from recorded fixtures. A sketch:

```swift
@Test
func aSlowListShowsLoadingThenItsItems() async throws {
    try await withFixtureApp(replies: ["GET /lists/{id}/items": .slow]) { app in
        let list = app.makeShoppingList()
        await list.load()
        #expect(list.state.isLoading)

        await app.gate.open()      // release the held "slow" reply, without sleeping
        try await list.untilLoaded()
        #expect(list.state.items.count == 3)
    }
}
```

A few properties make these tests trustworthy:

- **No real time.** "Slow" replies are held until the test opens a gate, so there are no sleeps and no timing flakiness.
- **A hard time limit.** A test that stalls fails after a few seconds instead of hanging the run.
- **A leak check.** Objects the story creates are tracked, and the test fails if any outlive it.
- **No assertions on visible text.** Copy changes, so assert on state and recorded calls.

## One set of fixtures for every layer

Record the backend's replies once, as JSON per endpoint and scenario: `happy`, `empty`, `slow`, and errors with real bodies. Then serve **the same files** to unit tests, behavior tests, snapshot tests and UI tests (through an in-app mock transport). Shared replies such as a generic 500 or offline live once, not copied into every endpoint.

The payoff comes when the backend changes a field. A contract check that compares live replies with the fixtures catches it in one place, and every test layer sees the change. If UI tests use their own hand-written mock, they keep passing against a backend that no longer exists.

## Prove it on every pull request

The last piece is making all of this visible. Every pull request gets one comment with the result, updated on each push:

```markdown
| Check | Result |
| --- | --- |
| Spec traceability | ✅ passed |
| Lint, PR rules, unit + behavior tests, coverage | ❌ failed |
```

Two choices make that report useful:

1. **Stages don't stop each other.** Lint, static checks and the build run side by side, and the tests run whenever the build succeeds. One run lists *every* failure, so you don't fix one thing, rerun, and find the next.
2. **Statuses mean something.** Plain pass/fail isn't enough:

| Status | Meaning |
| --- | --- |
| PASSED | Ran, all green |
| FAILED | A test or check failed |
| FLAKY | Failed, then passed on a rerun. **Counts as failed** |
| CRASHED | Crash or hang |
| BLOCKED | Couldn't run: build failed, or the backend was unavailable |
| SKIPPED | Out of scope for this change |

"FLAKY counts as failed" is the rule that stops teams from normalising retries.

This is what makes small, frequent merges safe. When every PR arrives with proof, trunk-based development stops being a leap of faith.

## What the checker can't prove

Be honest about the gap. The script proves a link **exists**. It can't prove the test asserts what the `Then` lines say. A test called `anEmptyNameIsRejected` could check that the name is rejected and never look at the button.

A Gherkin runner closes that gap by construction, because each `Then` line *is* code. Without one, it's a review rule:

> A scenario counts as covered only when its test asserts every `Then` line. If it asserts only some, extend the test or leave the scenario pending.

In practice this is the main cost of the approach. The checker removes the drift; reviewers still have to judge whether the test is complete.

**[TODO: add your own experience here: how often reviews caught a partial test, and whether it was worth the trade.]**

## When to choose this over a runner

**Choose spec + checker when:**
- your platform's Gherkin tooling is weak (iOS today),
- your tests are already in a framework you like (Swift Testing),
- you want product people to read the spec but not to write executable steps.

**Choose a real runner when:**
- non-engineers write new scenarios and expect them to run,
- you have a large, stable library of reusable steps,
- the `Then`-line gap is a risk you can't cover in review.

## Wrapping up

The approach comes down to four rules:

1. The `.feature` file is the contract; Swift tests are the proof.
2. Every scenario names its test, or says why it has none.
3. A small script checks those links on every pull request.
4. Reviewers check that a linked test asserts every `Then` line.

It's less magic than Cucumber, and that's the point: nothing to learn beyond Gherkin, Swift Testing and a short Python script.

**[TODO: closing line: what you'd do differently, or a link to a follow-up post.]**
