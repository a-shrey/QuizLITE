# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

QuizLITE is a C++17 desktop flashcard app built with Qt6 Widgets and SQLite (raw `sqlite3` C API, no ORM). Study sets are created in-app and studied via three modes: flashcards, multiple choice, and inverse multiple choice (answer → pick the question). Per-question stats (`TotalCorrect`, `TimesAsked`) are stored in SQLite.

## Build, test, lint

```bash
# Configure + build (requires Qt6, SQLite3, GoogleTest; on macOS: brew install qt googletest sqlite3)
cmake -S . -B build
cmake --build build -j8

# Run the app
./build/QuizLITE

# Run all tests (TestExecutables does NOT link Qt — only core logic is tested)
cd build && ctest --output-on-failure

# Run a single test / fixture
./build/TestExecutables --gtest_filter='MultipleChoiceTest.*'
./build/TestExecutables --gtest_filter='DatabaseManagerTest.OpenDatabase'

# Lint (CI enforces WebKit style, but only on these four dirs — not Interface/ or main.cpp)
clang-format --style=webkit -i Database/*.cpp Database/*.h User/*.cpp User/*.h StudyingMethods/*.cpp StudyingMethods/*.h tests/*.cpp
```

Builds use `-Wall -Werror -Wextra -pedantic -pedantic-errors` — any new warning is a build failure.

CI (`.github/workflows/build.yml`) runs the test suite on Ubuntu (Clang + GCC) and macOS, plus the clang-format lint job, on every push and PR.

## Architecture

Three layers, glued by two singletons:

- **`Database/DatabaseManager`** — singleton wrapper over the `sqlite3` C API (`getDatabaseManager(dbName)`). Executes raw SQL strings; `executeQueryWithResults` returns rows as `vector<map<column, value>>`.
- **`User/UserSession`** — singleton facade that owns all SQL. Despite the name there is no user/auth concept; it is the data-access layer the UI and study methods call (`createStudySet`, `addToStudySet`, `updateScore`, `getLowestAccuracies`, `getRandomEntries`, …). The app database is `user_data.db` in the working directory.
- **`StudyingMethods/`** — `StudyMethods` is an abstract base (`getQuestion`/`getAnswer`/`goToNextQuestion`); `Flashcards`, `MultipleChoice`, and `InverseMultipleChoice` implement it by pulling question lists from `UserSession` ("adaptive" = lowest-accuracy questions first via `getLowestAccuracies`).
- **`Interface/`** — `MainWindow` holds a `QStackedWidget` of page widgets (`LibraryPage`, `CreateSetPage`, `AddQuestionsPage`, `EnterSetPage`, `FlashcardPage`, `MCPage`, `InverseMCPage`). Pages signal up to `MainWindow` slots, which switch pages and call `UserSession`. `main.cpp` only instantiates `MainWindow`.

### SQLite schema

A master table `set_names(id, name)` plus **one table per study set** (table name = set name), each `(id, Key TEXT UNIQUE, Value TEXT, TotalCorrect INTEGER, TimesAsked INTEGER)`. Creating/deleting a set creates/drops a table.

### Things to know before changing code

- Most SQL is built by **string concatenation**, not prepared statements — set names and Q/A text containing quotes break queries (and are an injection vector). Prefer parameterized statements for any new SQL.
- The singletons expose `resetInstance()` only under the `TESTING` macro (defined for the test target); test fixtures are `friend` classes of `DatabaseManager`/`UserSession`. Tests share a real on-disk database, so they are order-sensitive — cleanup happens in fixture teardown.
- The `Interface/` layer has no tests and is excluded from CI lint; the WebKit format style there is inconsistent with the rest of the repo.
