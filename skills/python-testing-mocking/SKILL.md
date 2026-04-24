---
name: python-comprehensive-testing-mocking
description: Instructs the agent on building robust test suites combining pytest, unittest, and doctest. Guides the use of the Arrange-Act-Assert pattern and mandates using unittest.mock.Mock objects and pytest monkeypatch to isolate unpredictable elements like time, randomness, and OS functions.
---

# Python Comprehensive Testing & Mocking Guidelines

You are an expert Python developer specializing in building trustworthy, robust software test suites [2]. When asked to write or update tests, you must combine testing frameworks, utilize strict architectural patterns, and mock unpredictable external resources. Adhere to the following rules:

## 1. Test Suite Composition
An important part of creating trustworthy software is running a wide variety of tests [6]. Choose the right tool for each scenario:
*   **`pytest`:** The primary framework. Use `pytest` for functional test definitions utilizing built-in `assert` statements and reusable fixtures [9].
*   **`unittest`:** Use `unittest.TestCase` classes when you need sophisticated, object-oriented test setups that diverge from the "happy path" or test large datasets [8].
*   **Execution:** Run the full suite with `pytest`. The `pytest` runner can discover and execute both `pytest`-style and `unittest`-style tests automatically.

## 2. The Arrange-Act-Assert (GIVEN-WHEN-THEN) Pattern
Structure all test scenarios using the Arrange-Act-Assert (or GIVEN-WHEN-THEN) pattern to clearly define the test's intent [12, 13].
*   **Arrange (Given):** Set up the initial state. In `pytest`, use `@pytest.fixture` decorators to create reusable setup code that provides objects or mock data [14, 15].
*   **Act (When):** Perform the specific operation on the object being tested [14].
*   **Assert (Then):** Confirm the actual results match the expected results using `assert` statements. Avoid putting multiple independent assertions in a single test function; instead, separate them into multiple functions that share the same setup fixture to ensure all diagnostic information is visible [16, 17].

## 3. Isolating Unpredictable Elements (Time and Randomness)
Never rely on unpredictable functions like `datetime.datetime.now()` or `random.choice()` during unit tests, as they make expected results impossible to predict [18, 19].
*   **Monkeypatching:** Use the `monkeypatch` fixture provided by `pytest` to inject `unittest.mock.Mock` objects into the module under test [20-22].
*   **Mocking Time:** Create a fixture that returns a `Mock` object wrapping the `datetime` module, setting a specific `return_value` for the `now()` method [23]. 
*   **Mocking Randomness:** Use `Mock` with the `side_effect` parameter to return a predictable, fixed sequence of choices for random functions [24]. Use `unittest.mock.sentinel` objects to verify that specific data populations pass through a randomized function untouched [25, 26].

## 4. Mocking External OS and Filesystem Resources
When testing functions that create, modify, or delete files, you must isolate the filesystem to avoid corrupting a working system [27, 28].
*   **Safe Directories:** Always use the built-in `tmp_path` fixture from `pytest` to generate temporary, isolated directories for test files [21].
*   **Simulating Failures:** To test file-handling failure modes (like error recovery or backup preservation), use `monkeypatch` to replace the writing functions with a `Mock` whose `side_effect` executes a function that writes corrupt data and then raises exceptions (like `RuntimeError` or `IOError`) [29, 30].

---

## Code Examples

Below are best-practice examples demonstrating these testing and mocking patterns.

**1. Mocking `datetime.now()` and `random` with `monkeypatch`**
```python
import datetime
from pathlib import Path
from unittest.mock import Mock, call, sentinel
import pytest

# The module being tested
import my_app 

# 1. Arrange (Given): A fixture to mock datetime.now()
@pytest.fixture()
def mock_datetime() -> Mock:
    return Mock(
        name="mock_datetime",
        datetime=Mock(
            now=Mock(return_value=datetime.datetime(2026, 1, 1, 12, 0, 0))
        ),
        timezone=Mock(utc=Mock(name="UTC"))
    )

# 2. Arrange (Given): A fixture to mock random.choice
@pytest.fixture()
def mock_random_choice() -> Mock:
    # Use side_effect to return a predictable sequence or a sentinel
    return Mock(name="mock_random.choice", return_value=sentinel.CHOICE)

# 3. Test execution combining fixtures and monkeypatching
def test_processing_logic(
    mock_datetime: Mock, 
    mock_random_choice: Mock, 
    monkeypatch: pytest.MonkeyPatch
) -> None:
    
    # Patch the unpredictable modules inside the target application
    monkeypatch.setattr(my_app, "datetime", mock_datetime)
    monkeypatch.setattr(my_app.random, "choice", mock_random_choice)
    
    # Act (When)
    result = my_app.process_random_event(sentinel.POPULATION)
    
    # Assert (Then)
    assert result.timestamp == datetime.datetime(2026, 1, 1, 12, 0, 0)
    assert result.selection == sentinel.CHOICE
    mock_random_choice.assert_called_once_with(sentinel.POPULATION)
2. Testing OS Failures with tmp_path
from pathlib import Path
from unittest.mock import Mock, sentinel
import pytest
import my_file_app

# A mock function designed to simulate an OS-level failure mid-write
def save_data_failure(path: Path, content: str) -> None:
    path.write_text("CORRUPTED", encoding="utf-8")
    raise RuntimeError("Simulated OS failure")

def test_safe_write_recovery(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    # Arrange: Setup a safe temporary file environment
    original_file = tmp_path / "important_data.csv"
    original_file.write_text("GOOD_DATA", encoding="utf-8")
    
    # Arrange: Mock the inner write function to simulate failure
    mock_save_data = Mock(side_effect=save_data_failure)
    monkeypatch.setattr(my_file_app, "save_data", mock_save_data)
    
    # Act & Assert: Verify the application handles the RuntimeError safely
    with pytest.raises(RuntimeError):
        my_file_app.safe_write(original_file, "NEW_DATA")
        
    # Assert: Confirm the original file was untouched/recovered despite the crash
    actual = original_file.read_text(encoding="utf-8")
    assert actual == "GOOD_DATA"
```