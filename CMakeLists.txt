cmake_minimum_required(VERSION 3.18)
project(QuizLITE)

# Set compiler flags.
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Werror -Wextra -pedantic -pedantic-errors -g")

# Define all testing related content here
enable_testing()

find_package(SQLite3 REQUIRED)
# Some platform/toolchain combinations resolve SQLite3 through a find module
# that reports the package found without defining the imported target.
if(NOT TARGET SQLite3::SQLite3)
    message(STATUS "SQLite3::SQLite3 target missing after find_package "
            "(CMake ${CMAKE_VERSION}, lib='${SQLite3_LIBRARIES}', inc='${SQLite3_INCLUDE_DIRS}'); defining it manually")
    add_library(SQLite3::SQLite3 UNKNOWN IMPORTED)
    set_target_properties(SQLite3::SQLite3 PROPERTIES
            IMPORTED_LOCATION "${SQLite3_LIBRARIES}"
            INTERFACE_INCLUDE_DIRECTORIES "${SQLite3_INCLUDE_DIRS}")
endif()
find_package(GTest REQUIRED)
find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets)
qt_standard_project_setup()

add_executable(QuizLITE
        main.cpp
        User/UserSession.h
        User/UserSession.cpp
        Database/DatabaseManager.cpp
        Database/DatabaseManager.h
        StudyingMethods/StudyMethods.h
        StudyingMethods/Flashcards.cpp
        StudyingMethods/Flashcards.h
        StudyingMethods/MultipleChoice.cpp
        StudyingMethods/MultipleChoice.h
        StudyingMethods/InverseMultipleChoice.cpp
        StudyingMethods/InverseMultipleChoice.h
        Interface/MainWindow.cpp
        Interface/MainWindow.h
        Interface/LibraryPage.cpp
        Interface/LibraryPage.h
        Interface/CreateSetPage.cpp
        Interface/CreateSetPage.h
        Interface/AddQuestionsPage.cpp
        Interface/AddQuestionsPage.h
        Interface/EnterSetPage.cpp
        Interface/EnterSetPage.h
        Interface/MCPage.cpp
        Interface/InverseMCPage.cpp
        Interface/InverseMCPage.h
        Interface/FlashcardPage.cpp
        Interface/FlashcardPage.h
)

target_link_libraries(QuizLITE
        SQLite3::SQLite3
        Qt6::Widgets
        Qt6::Core
        Qt6::Gui
)

# Test executables
add_executable(TestExecutables
        tests/main.cpp
        tests/testDatabase.cpp
        Database/DatabaseManager.cpp
        Database/DatabaseManager.h
        tests/testUser.cpp
        User/UserSession.h
        User/UserSession.cpp
        tests/testFlashcards.cpp
        StudyingMethods/StudyMethods.h
        StudyingMethods/Flashcards.cpp
        StudyingMethods/Flashcards.h
        StudyingMethods/MultipleChoice.cpp
        StudyingMethods/MultipleChoice.h
        StudyingMethods/InverseMultipleChoice.cpp
        StudyingMethods/InverseMultipleChoice.h
        tests/testMultipleChoice.cpp
        tests/testInverseMultipleChoice.cpp
)
target_link_libraries(TestExecutables
        GTest::GTest
        GTest::Main
        SQLite3::SQLite3
)

# Define TESTING macro for test build
target_compile_definitions(TestExecutables PRIVATE TESTING)

# Add tests to CTest
add_test(NAME TestExecutables COMMAND TestExecutables)

# Custom target to run tests with verbose output
add_custom_target(run_tests
        COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
        DEPENDS TestExecutables
)


