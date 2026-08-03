---
name: python-clean-architecture-test-doubles
description: Instructs the agent to test use cases, controllers, and presenters by passing plain stand-in functions instead of unittest.mock.Mock(spec=SomeABC) — there is no ABC left to spec against once dependencies are Callable types, and a stand-in is just a function literal or a small recording closure. Covers verifying calls without Mock's assert_called_once_with, pytest fixtures that build plain dataclasses/functions instead of Mock objects, parameterized testing of pure functions, and freezegun for time control — all composing naturally once dependencies are already plain functions.
---

# Python Clean Architecture: Test Doubles, Functional-Lite

Reference material tests use cases and controllers by injecting
`unittest.mock.Mock()` or `Mock(spec=TaskRepository)` in place of a
repository/service. Once repositories and services are `Callable` types and
plain functions (see python-clean-architecture-dependency-inversion), not
ABCs, `Mock(spec=...)` has nothing left to spec against — and doesn't need
to. A stand-in for a `Callable` dependency is just another function: a
lambda, a small named function, or a closure that records what it was
called with. This skill covers that reformulation across use cases,
controllers, presenters, fixtures, parameterized tests, and time control.

---

## When to use this skill

Use this skill when you need to:

- test a use-case or controller function that takes `Callable`
  dependencies as parameters,
- translate a `Mock()`/`Mock(spec=SomeABC)`-based test into this repo's
  style,
- verify that a dependency was called with specific arguments, without
  `Mock`'s `assert_called_once_with`,
- write pytest fixtures that supply test doubles for functional-lite code,
- combine parameterized testing or `freezegun` time control with
  function-based test doubles.

---

## Outcome

Produce tests that:

- pass plain functions (lambdas, small named functions, or recording
  closures) as stand-ins for `Callable` dependencies — never
  `Mock()`/`Mock(spec=...)`, since there's no ABC to spec against and a
  function stand-in is simpler than configuring a mock to behave like one,
- verify "was this called, and with what" using a plain list a recording
  closure appends to, checked with a normal `assert` — not
  `mock.assert_called_once_with(...)`,
- keep `Mock`/`monkeypatch` for what python-testing-mocking already
  establishes them for — genuinely unpredictable module-level things like
  `datetime.now()` or `random.choice()` — never for a repository or
  service dependency, which is just a function now,
- build pytest fixtures that return plain dataclasses and plain stand-in
  functions, mirroring production dependency structure exactly.

---

## Instructions for the AI

1. **Replace `Mock(spec=TaskRepository)` with a plain stand-in function**
   - Translate a use-case test built on mocks:
     ```python
     def test_successful_task_completion():
         task = Task(title="Test task", description="...", project_id=...)
         task_repo = Mock()
         task_repo.get.return_value = task
         notification_service = Mock()
         use_case = CompleteTaskUseCase(
             task_repository=task_repo, notification_service=notification_service
         )
         result = use_case.execute(CompleteTaskRequest(task_id=str(task.id)))
         assert result.is_success
         task_repo.save.assert_called_once_with(task)
         notification_service.notify_task_completed.assert_called_once_with(task)
     ```
     into a test where the dependencies are just functions — a fixed-
     return lambda for the read side, and a small recording closure for
     the write/notify side (since those need to be checked *after* the
     call):
     ```python
     def test_successful_task_completion():
         task = Task(title="Test task", description="...", project_id=...)

         saved: list[Task] = []
         notified: list[Task] = []

         result = complete_task_use_case(
             get_task=lambda _id: task,
             save_task=saved.append,
             notify_task_completed=notified.append,
             task_id=task.id,
         )

         assert isinstance(result, Success)
         assert saved == [task]
         assert notified == [task]
     ```
   - `saved.append`/`notified.append` are already exactly the right shape
     for `SaveTask`/`NotifyTaskCompleted` (`Callable[[Task], None]`) — a
     bound list method *is* a valid stand-in function, no wrapping needed.
     Asserting `saved == [task]` verifies both that it was called and
     with what, in one plain `assert` — no separate `assert_called_once`
     and `assert_called_with` needed.

2. **Use a small named function instead of a lambda when the stand-in needs logic**
   - When a stand-in needs to do more than return a fixed value (raise for
     a specific input, behave differently based on the argument), write a
     small named function instead of forcing it into a lambda:
     ```python
     def test_completing_missing_task_returns_failure():
         def get_task_not_found(task_id: UUID) -> Task:
             raise TaskNotFoundError(task_id)

         result = complete_task_use_case(
             get_task=get_task_not_found,
             save_task=lambda task: None,
             notify_task_completed=lambda task: None,
             task_id=uuid4(),
         )

         assert isinstance(result, Failure)
         assert result.error.code == ErrorCode.NOT_FOUND
     ```
   - This reads as directly as the equivalent `Mock(side_effect=...)`
     configuration, without needing to know `Mock`'s API for configuring
     side effects at all.

3. **Test controllers and presenters the same way — function stand-ins, not `Mock(spec=...)`**
   - Translate a controller test using `Mock(spec=TaskPresenter)`:
     ```python
     def test_controller_converts_string_id_to_uuid():
         task_id = "123e4567-e89b-12d3-a456-426614174000"
         complete_use_case = Mock()
         complete_use_case.execute.return_value = Result.success(...)
         presenter = Mock(spec=TaskPresenter)
         controller = TaskController(complete_use_case=complete_use_case, presenter=presenter)
         controller.handle_complete(task_id=task_id)
         complete_use_case.execute.assert_called_once()
         called_request = complete_use_case.execute.call_args[0][0]
         assert isinstance(called_request.task_id, UUID)
     ```
     into a test recording what the stand-in use-case function received:
     ```python
     def test_controller_converts_string_id_to_uuid():
         task_id = "123e4567-e89b-12d3-a456-426614174000"
         received_ids: list[UUID] = []

         def fake_complete_task(tid: UUID) -> Result[Project]:
             received_ids.append(tid)
             return Success(some_project)

         handle_complete_task(fake_complete_task, cli_present_task, task_id=task_id)

         assert received_ids == [UUID(task_id)]
     ```
   - The assertion is arguably clearer than the `Mock` version — it
     directly inspects the value the controller passed through, rather
     than reaching into `call_args[0][0]` to extract it from a mock's
     recorded call history.
   - A presenter stand-in, when the test doesn't care about formatting
     details, can be as simple as `lambda project: project` or a small
     fixed `TaskViewModel` — there's no `spec=TaskPresenter` to configure
     since the presenter is just a `Callable[[Project], TaskViewModel]`.

4. **Keep `Mock`/`monkeypatch` for genuinely unpredictable module-level things**
   - `python-testing-mocking` already establishes `Mock` and
     `monkeypatch` as the right tool for isolating `datetime.now()`,
     `random.choice()`, and OS/filesystem calls — that guidance is
     unchanged and still correct here.
   - The distinction to hold onto: `Mock` is for things you don't control
     the shape of (stdlib/OS behavior); a plain function is for things
     *this codebase* already defined as a `Callable` type. Reaching for
     `Mock` on a repository or service dependency is reaching for a
     heavier tool than the code actually calls for.

5. **Build fixtures that return plain dataclasses and functions, not `Mock` objects**
   - Translate `conftest.py` fixtures built around `Mock(spec=...)`:
     ```python
     @pytest.fixture
     def mock_task_repository(domain_task):
         repo = Mock(spec=TaskRepository)
         repo.get.return_value = domain_task
         return repo
     ```
     into fixtures returning plain functions:
     ```python
     @pytest.fixture
     def get_task_returning(domain_task) -> GetTask:
         return lambda task_id: domain_task

     @pytest.fixture
     def recording_save_task() -> tuple[SaveTask, list[Task]]:
         saved: list[Task] = []
         return saved.append, saved
     ```
   - Fixture composition still works exactly the way it did with mocks —
     a fixture for a controller test can depend on fixtures providing its
     individual function dependencies, mirroring production's dependency
     structure (a controller fixture built from a use-case-function
     fixture built from repository-function fixtures) exactly as
     `python-clean-architecture-composition-root` assembles the real
     application.

6. **Combine with parameterized testing exactly as before**
   - `pytest.mark.parametrize` needs no reformulation — it already works
     naturally over pure functions, arguably more cleanly than over
     class-based use cases, since there's no object construction step to
     parameterize around:
     ```python
     @pytest.mark.parametrize(
         "request_data,expected_priority",
         [
             ({"title": "Basic", "description": "..."}, Priority.MEDIUM),
             ({"title": "Urgent", "description": "...", "priority": "HIGH"}, Priority.HIGH),
         ],
         ids=["basic-task", "priority-task"],
     )
     def test_task_creation_scenarios(request_data, expected_priority, get_inbox, save_task):
         result = create_task_use_case(get_inbox, save_task, CreateTaskRequest(**request_data))
         assert isinstance(result, Success)
         assert result.value.priority == expected_priority
     ```

7. **Combine with `freezegun` exactly as before**
   - `freeze_time` also needs no reformulation — it controls what
     `datetime.now()` returns regardless of whether the code calling it is
     a class method or a plain function:
     ```python
     def test_task_deadline_approaching():
         with freeze_time("2024-01-14 12:00:00"):
             task = Task(..., due_date=Deadline(datetime(2024, 1, 15, 12, 0, tzinfo=timezone.utc)))

         notified: list[Task] = []
         with freeze_time("2024-01-14 13:00:00"):
             result = check_deadlines_use_case(
                 get_all_tasks=lambda: [task],
                 notify_deadline_approaching=notified.append,
                 warning_threshold=timedelta(days=1),
             )

         assert isinstance(result, Success)
         assert notified == [task]
     ```

---

## Decision points and guidance

- **Is a test reaching for `Mock(spec=SomeABC)`?** There's no ABC left —
  replace it with a plain function (lambda, named function, or recording
  closure) matching the dependency's `Callable` type.
- **Does the test need to verify a call happened with specific
  arguments?** Use a recording closure (append to a list) and assert on
  the list directly, rather than `Mock`'s `assert_called_with`.
- **Is the thing being mocked a repository/service dependency, or a
  genuinely unpredictable module-level function (`datetime.now`,
  `random`)?** The former gets a plain function stand-in; the latter still
  legitimately uses `Mock`/`monkeypatch` per python-testing-mocking.
- **Do fixtures return `Mock` objects?** Replace them with fixtures
  returning plain functions/dataclasses, composed the same way production
  dependencies are composed.

---

## Quality criteria

A strong functional-lite test-doubles approach should ensure that:

- **no test reaches for `Mock(spec=...)` against a repository or service
  dependency** — only plain stand-in functions are used,
- **call verification uses a plain list and `assert`**, not `Mock`'s
  call-tracking API,
- **`Mock`/`monkeypatch` are reserved for genuinely unpredictable
  module-level behavior**, exactly as python-testing-mocking already
  establishes,
- **fixtures return plain dataclasses and functions**, composed the same
  way the real composition root composes them,
- **parameterized tests and `freezegun` compose unchanged**, since neither
  depended on the production code being class-based in the first place.

---

## Example prompts

- "This test uses `Mock(spec=TaskRepository)` — there's no ABC anymore,
  help me reformulate it with plain stand-in functions."
- "How do I verify this use case called the notification function with
  the right task, without using `Mock`'s assert methods?"
- "This fixture returns a `Mock` — reformulate it to return a plain
  function matching our `Callable` type."

---

## Related guidance

Use this tool alongside:

- python-clean-architecture-testing-strategy
- python-clean-architecture-dependency-inversion
- python-clean-architecture-composition-root
- python-testing-mocking
- python-domain-error-handling
