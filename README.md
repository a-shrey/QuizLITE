<h1 align="center">
  <img src="/icons/quizlite.png" alt="Icon" width="100"/>
  <br>
  QuizLITE
</h1>

<p align='center'>
  <a target="_blank" href='https://isocpp.org/'><img src='https://img.shields.io/badge/C++-blue?style=for-the-badge&logo=cplusplus&color=00599C&labelColor=C++&logoColor=white' alt="C++"></a>
  <a target="_blank" href='https://cmake.org/'><img src='https://img.shields.io/badge/CMake-blue?style=for-the-badge&logo=cmake&color=064F8C&labelColor=CMake&logoColor=white' alt="CMake"></a>
  <a target="_blank" href='https://www.sqlite.org/'><img src='https://img.shields.io/badge/SQLite-blue?style=for-the-badge&logo=sqlite&color=003B57&labelColor=SQLite&logoColor=white' alt="SQLite"></a>
  <a target="_blank" href='https://www.qt.io/'><img src='https://img.shields.io/badge/Qt-blue?style=for-the-badge&logo=qt&color=41CD52&labelColor=Qt&logoColor=white' alt="Qt"></a>
  <a target="_blank" href='https://github.com/google/googletest'><img src='https://img.shields.io/badge/GTest-blue?style=for-the-badge&logo=google&color=4285F4&labelColor=GTest&logoColor=white' alt="GTest"></a>
  <a target="_blank" href='https://github.com/features/actions'><img src='https://img.shields.io/badge/GitHub%20Actions-blue?style=for-the-badge&logo=githubactions&color=2088FF&labelColor=GitHub%20Actions&logoColor=white' alt="GitHub Actions"></a>
</p>

A lightweight desktop flashcard app that tracks per-question accuracy and focuses your review on the questions you miss most. Built in C++17 with Qt 6 Widgets and SQLite.

## Project status

QuizLITE was built in summer 2024 as a two-person student project. The desktop app works and its test suite passes, but it is **feature-frozen**: development effort is moving to a native iOS rewrite that keeps this app's data model and study modes as its specification. See [PLAN.md](PLAN.md) for the roadmap and [CLAUDE.md](CLAUDE.md) for architecture and development commands.

## What it does

- Create, edit, and delete study sets of question/answer pairs
- Three study modes: **flashcards**, **multiple choice**, and **inverse multiple choice** (see the answer, pick the question)
- Records `TotalCorrect` / `TimesAsked` per question in a local SQLite database
- Multiple-choice sessions are seeded with the questions you've answered worst, plus a random sample
- 36 GoogleTest cases run in CI on Ubuntu (GCC + Clang) and macOS
- clang-format (WebKit style) enforced in CI for the core (non-UI) directories

## Known limitations

These are real, verified, and documented in detail in [docs/AUDIT-2026-06.md](docs/AUDIT-2026-06.md):

- **No spaced repetition.** "Worst questions first" is a one-shot ranking at session start, not an interval scheduler. (The iOS rewrite adopts FSRS for this.)
- **No accounts, sync, or auth.** All data lives in a local SQLite file (`StudySets.db`).
- Most SQL is built by string concatenation, so cards containing an apostrophe can break inserts.
- Multiple-choice option generation can hang (infinite loop) if a set larger than 4 cards has fewer than 3 distinct wrong answers — duplicate answers are allowed by the schema.
- The flashcard mode has known session-flow bugs (first card skipped; finish button never shown).
- The Qt UI layer (`Interface/`) is untested and excluded from CI lint.

## Building

### Prerequisites

- A C++17 compiler
- CMake 3.18 or higher
- Qt 6 (Core, Gui, Widgets)
- SQLite3
- GoogleTest
- Git

On macOS: `brew install cmake qt googletest sqlite3`

### Build and run

```bash
git clone https://github.com/a-shrey/QuizLITE.git
cd QuizLITE
cmake -S . -B build
cmake --build build -j8
./build/QuizLITE
```

### Tests

```bash
cd build && ctest --output-on-failure          # full suite
./build/TestExecutables --gtest_filter='MultipleChoiceTest.*'   # one fixture
```

The test target does not link Qt — only the database, session, and study-method layers are covered.

## Contributing

This is a personal side project. Issues and PRs are welcome, but read [PLAN.md](PLAN.md) first — new feature work targets the iOS rewrite, not the Qt app.

## License

MIT — see [LICENSE](LICENSE).
