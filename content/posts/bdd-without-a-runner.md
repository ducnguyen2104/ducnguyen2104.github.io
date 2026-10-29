---
title: "From Specs to Test Cases: Behavior-Driven Development with XCTest"
date: 2026-10-29
draft: false
tags: ["testing", "bdd", "swift", "ios", "xctest"]
description: "What BDD is, why it beats a folder of unrelated test cases, and how to use it on iOS with plain Swift Testing, XCTest and XCUITest. No Cucumber required."
---

If you write iOS tests today, you probably have a folder of unit tests and maybe a few UI tests. They pass, CI is green, and yet nobody can say what the app is *supposed* to do, or which important cases have no test at all.

Behavior-Driven Development (BDD) is a way to fix that. This post explains BDD from scratch, then shows a lightweight way to use it with the tools you already have: **Swift Testing, XCTest and XCUITest**. No Cucumber, no new test framework.

## What is BDD?

BDD means you **describe how a feature should behave, in plain language, before you test it**. Each description is a short, concrete example called a *scenario*:

```gherkin
Scenario: An item with an empty name cannot be added
  Given the user is on the "Add item" screen
  When the user taps "Add" without typing a name
  Then the item is not added
  And the "Add" button stays disabled
```

Every scenario has the same three parts:

- **Given**: the starting situation.
- **When**: the one thing the user (or the system) does.
- **Then**: what should be true afterwards.

That format is called **Gherkin**, and scenarios live in plain text files ending in `.feature`. Each scenario then gets a test that proves the app really behaves that way.

The key word is *behavior*. A scenario talks about what the user sees and does, never about view models, stores or network calls. That's what lets a product manager, a designer, or a QA engineer read it and say "that's wrong" or "you forgot the offline case".

### A few terms you'll see

| Term | Meaning |
| --- | --- |
| Feature file | A `.feature` text file holding the scenarios for one feature. |
| Scenario | One example of behavior, written as Given / When / Then. |
| Gherkin | The name of that Given / When / Then format. |
| Step definition | Code that a runner matches to each Gherkin line so it can execute it. |
| Runner | A tool such as Cucumber that executes `.feature` files directly. |

You won't need step definitions or a runner in this post. They're listed so you recognize them in other BDD articles.

## Why not just write test cases?

Here's a typical test without BDD:

```swift
final class ShoppingListTests: XCTestCase {
    func testAddItem() async {
        let store = InMemoryListStore()
        let viewModel = AddItemViewModel(store: store)
        viewModel.name = ""
        await viewModel.addTapped()
        XCTAssertTrue(store.items.isEmpty)
    }
}
```

Did that last line surprise you? It's called `testAddItem`, yet it expects the list to be **empty** after tapping Add. The test is correct: it checks that an item with an empty name is *rejected*. But you only find that out by spotting the `""` on the third line.

That confusion is the point. The test works, but look at what's missing:

- **The name doesn't say what's expected.** `testAddItem` hides the rule it checks. You have to read every line to work it out, as you just did.
- **It's half a test.** The rule also says the button stays disabled, and nothing checks that. Nobody notices, because the rule was never written down.
- **Only developers can review it.** A product manager can't look at this file and tell you whether the behavior is right.
- **Gaps are invisible.** A folder of tests shows what *is* tested. It can't show what *isn't*: the offline case, the 200-character name, the double tap.

BDD flips the order. You write the list of behaviors first, then a test for each one. That gives you:

1. **A spec everyone can read.** Wrong or missing behavior gets caught in review, before any code is written.
2. **Visible gaps.** A scenario with no test stands out. A test that was never written doesn't.
3. **Better unhappy paths.** Writing scenarios makes you ask "what if it fails?" up front, instead of after a bug report.
4. **Tests that explain themselves.** `anEmptyNameIsRejected` tells you the rule; `testAddItem` doesn't.

It isn't free: you now maintain a `.feature` file next to your tests. It pays off for features with real rules and edge cases (forms, payments, sync, permissions). For a static "About" screen, skip it.

## The usual problem with BDD

Most teams that try BDD end up with one of two problems:

1. **The `.feature` files are just documents.** Nothing connects them to the tests, so they drift away from the app within a sprint.
2. **The team adopts a runner** such as Cucumber, and spends its time maintaining step definitions instead of tests.

On iOS, option 2 is harder than it sounds. There's no official Cucumber for Swift, the community runners don't fit Swift Testing, and debugging a failure means jumping from a feature line to a regex to the step code to the app.

This post takes a middle path: **keep Gherkin as the spec, write ordinary XCTest and Swift Testing tests, and use a small script to prove every scenario points to a real test.**

## The idea in one picture

```
Specs/ShoppingList.feature        →  what the app should do, in plain language
        │  each scenario names its test
        ▼
ShoppingListTests/…               →  unit tests (Swift Testing or XCTest)
ShoppingListUITests/…             →  UI tests (XCUITest)
        ▲
        │  checks that every named test exists
Scripts/check-specs.py            →  runs on every pull request
```

A scenario is never "just documentation". It either names the test that proves it, or it says why there isn't one yet.

## Anatomy of a scenario

Here's a scenario from a made-up shopping list app:

```gherkin
Feature: Shopping list

  Background:
    Given the user is signed in with a list called "Weekend"

  @SL_012 @validation @covered-by:AddItemViewModelTests.anEmptyNameIsRejected
  Scenario: An item with an empty name cannot be added
    When the user adds an item with an empty name
    Then the item is not added
    And the add button stays disabled
```

`Background` holds the `Given` steps shared by every scenario in the file. The tags on the line above `Scenario` carry the information the script needs:

| Tag | Purpose |
| --- | --- |
| `@SL_012` | A stable ID. Never reused or renumbered, even when scenarios are deleted. |
| `@validation` | One of three categories: `@happy_path`, `@validation`, `@edge_case`. |
| `@covered-by:Type.method` | The test that proves this scenario. |

## Three ways to be "covered"

Every scenario must be in exactly one of these states.

**1. An existing test already proves it.** Point at it with `@covered-by`, as in the scenario above. This Swift Testing test is the one it names:

```swift
import Testing
@testable import ShoppingList

@MainActor
struct AddItemViewModelTests {
    @Test
    func anEmptyNameIsRejected() async {
        let store = InMemoryListStore()
        let viewModel = AddItemViewModel(store: store)

        viewModel.name = ""
        await viewModel.addTapped()

        #expect(store.items.isEmpty)               // Then the item is not added
        #expect(viewModel.isAddButtonEnabled == false) // And the add button stays disabled
    }
}
```

Each `Then` line has a matching assertion. That's the rule that makes "covered" mean something.

**2. No test exists yet, so write one named after the scenario.** Start the method name with `test_` plus the scenario ID. XCTest needs the `test` prefix anyway:

```swift
import XCTest
@testable import ShoppingList

@MainActor
final class ShoppingListScenarioTests: XCTestCase {
    // SL_014: A failed save keeps what the user typed
    func test_SL_014_aFailedSaveKeepsTheDraft() async {
        let store = FailingListStore(error: .offline)
        let viewModel = AddItemViewModel(store: store)

        viewModel.name = "Milk"
        await viewModel.addTapped()

        XCTAssertEqual(viewModel.name, "Milk")
        XCTAssertEqual(viewModel.error, .offline)
        XCTAssertTrue(store.items.isEmpty)
    }
}
```

The `test_SL_014_` prefix is the link, so the scenario doesn't need a `@covered-by` tag.

**3. It can't be tested yet, and the spec says why:**

```gherkin
@SL_031 @edge_case @blocked:mock
# The mock backend cannot return a list shared by two users yet.
Scenario: A list removed by its owner disappears for everyone
```

Keep a small, fixed set of these statuses, such as `@automation-pending`, `@blocked:<reason>` and `@manual`. What matters is that "untested" is a **visible, deliberate choice**, not an accident.

## Unit test or UI test?

A scenario describes behavior, not *how* to test it. Pick the cheapest test that proves every `Then` line:

| The scenario is about… | Write a… | With |
| --- | --- | --- |
| A rule or validation (empty name, max length) | Unit test on the view model | Swift Testing or XCTest |
| Error handling (offline, failed save) | Unit test with a fake store | Swift Testing or XCTest |
| A journey across screens | UI test | XCUITest |

Most scenarios should be unit tests. They run in milliseconds and rarely flake. Save UI tests for a handful of journeys that really need the whole app, like this one:

```gherkin
@SL_001 @happy_path
Scenario: The user adds an item to the list
  When the user adds an item called "Milk"
  Then "Milk" appears in the list
```

```swift
import XCTest

@MainActor
final class ShoppingListUITests: XCTestCase {
    // SL_001: The user adds an item to the list
    func test_SL_001_theUserAddsAnItem() {
        let app = XCUIApplication()
        app.launchArguments = ["-useMockData"]
        app.launch()

        app.buttons["addItemButton"].tap()
        let nameField = app.textFields["itemNameField"]
        nameField.tap()
        nameField.typeText("Milk")
        app.buttons["saveItemButton"].tap()

        XCTAssertTrue(app.staticTexts["Milk"].waitForExistence(timeout: 5))
    }
}
```

Three habits keep UI tests reliable:

- **Use mock data.** A launch argument such as `-useMockData` tells the app to load canned responses instead of calling the real backend.
- **Find elements by accessibility identifier**, like `addItemButton`, not by their title. Button text changes; identifiers don't.
- **Never `sleep`.** Use `waitForExistence(timeout:)`, which returns as soon as the element appears.

## Don't only write happy paths

Left alone, a spec fills up with happy paths, because they're the easiest to write. A rough target keeps it honest:

- **~20% `@happy_path`**: the main journey works.
- **~40% `@validation`**: bad input, limits and rejected actions.
- **~40% `@edge_case`**: offline, slow network, double taps, missing permissions, the app going to the background.

Treat the numbers as a prompt, not a quota. If one category is thin, ask what's missing. Don't add filler scenarios to hit the target.

## The checker

The script is what keeps the spec and the tests from drifting apart. It doesn't run any tests and it isn't a full Gherkin parser. For every scenario it checks that:

- it has exactly one ID and one category,
- it has at least one `Given`, `When` and `Then`,
- any test named in `@covered-by` really exists,
- it has a test (by tag or by `test_<ID>_` name), or a pending status.

```python
#!/usr/bin/env python3
"""Checks that every scenario is well formed and traces to a test. Not a Gherkin runner."""
import re
import sys
from pathlib import Path

SPECS = Path("Specs")
TEST_DIRS = [Path("ShoppingListTests"), Path("ShoppingListUITests")]
CATEGORIES = {"@happy_path", "@validation", "@edge_case"}
PENDING = ("@automation-pending", "@blocked:", "@manual")

swift_files = [p.read_text() for folder in TEST_DIRS for p in folder.rglob("*.swift")]


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
for path in sorted(SPECS.glob("*.feature")):
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

Change `TEST_DIRS` to your own test target folders, then run `python3 Scripts/check-specs.py` from the project root. Add the same command as a step in CI. Now a pull request that renames `anEmptyNameIsRejected` without updating the spec fails straight away, which is the cheapest moment to catch drift.

When a scenario is broken, the output points at the exact line:

```
ShoppingList.feature:12: AddItemViewModelTests.anEmptyNameIsRejectd does not exist
ShoppingList.feature:29: SL_020 has no test and no pending status
```

## What the checker can't prove

Be honest about the gap. The script proves a link **exists**. It can't prove the test checks what the `Then` lines say. A test called `anEmptyNameIsRejected` could check that the item isn't added and never look at the button.

A Gherkin runner closes that gap, because each `Then` line *is* code. Without one, it's a review rule:

> A scenario counts as covered only when its test asserts every `Then` line. If it asserts only some, extend the test or leave the scenario pending.

That's the main cost of this approach. The script stops drift; reviewers still judge whether each test is complete.

## When to choose this over a runner

**Choose spec + checker when:**
- your platform's Gherkin tooling is weak (iOS today),
- your tests are already in a framework you like (Swift Testing or XCTest),
- you want product people to read the spec, but not to write executable steps.

**Choose a real runner when:**
- non-engineers write new scenarios and expect them to run,
- you have a large, stable library of reusable steps,
- the `Then`-line gap is a risk you can't cover in review.

## Wrapping up

BDD comes down to writing the behavior first, in words everyone can read, and then proving each piece with a test. On iOS you can do that with four rules:

1. The `.feature` file says what the app should do; XCTest and Swift Testing tests prove it.
2. Every scenario names its test, or says why it has none.
3. A small script checks those links on every pull request.
4. Reviewers check that each test asserts every `Then` line.

There's no new framework to learn: just Gherkin, the tests you already write, and a short Python script.
