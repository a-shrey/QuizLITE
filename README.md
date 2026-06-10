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




A lightweight desktop flashcard app that tracks per-question accuracy and focuses your review on the questions you miss most.

## Features

- Create, edit, and delete study sets
- Add and manage questions and answers within sets
- Three study modes: flashcards, multiple choice, and inverse multiple choice
- Per-question accuracy tracking stored locally in SQLite
- Review sessions prioritize the questions you get wrong most often
- Desktop UI built with Qt 6 Widgets
- 36 GoogleTest cases run on Ubuntu (GCC + Clang) and macOS via GitHub Actions
- clang-format (WebKit style) enforced in CI

## Getting Started

These instructions will help you get a copy of the project up and running on your local machine for development and testing purposes.

### Prerequisites

What you need to install the software:

- A C++17 compiler
- CMake 3.18 or higher
- Qt 6 (Core, Gui, Widgets)
- SQLite3
- GoogleTest
- Git

On macOS: `brew install cmake qt googletest sqlite3`

### Installing

A step-by-step guide to set up your development environment:

1. Clone the repository:

    ```bash
    git clone https://github.com/aditya-shrey/quizlite.git
    ```

2. Navigate to the project directory:

    ```bash
    cd quizlite
    ```

3. Create a build directory and navigate into it:

    ```bash
    mkdir build && cd build
    ```

4. Run CMake to configure the project:

    ```bash
    cmake ..
    ```

5. Build the project:

    ```bash
    cmake --build .
    ```

6. Run the application:

    ```bash
    ./QuizLITE
    ```

## Running the Tests

To run the tests, ensure you are in the build directory and execute:

  ```bash
  ctest
  ```

## Deployment

Deployment specifics depend on your target environment. Ensure to follow your internal guidelines for deploying C++/Qt applications.

## Built With

- C++ - Primary programming language
- CMake - Build system
- SQLite - Database engine
- Qt - UI framework
- GoogleTest (GTest) - Testing framework
- GitHub Actions - CI/CD

## Contributing
Contributions to this project are welcome. If you find any issues or have suggestions for improvements, please open an issue or submit a pull request.

## License
This project is licensed under the MIT License. Feel free to use and modify the code as per the license terms.
