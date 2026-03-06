---
name: test-iteration
description: Run tests, analyze failures, fix issues, and re-run in a loop until all tests pass. Use when developing features or fixing bugs that need test validation.
---

# Test Iteration

When iterating on code changes with tests:

1. **Run the full check first**:
   ```bash
   make check
   ```

2. **If tests fail**, focus on the failing test:
   ```bash
   uv run pytest tests/<test_file>.py::<TestClass>::<test_name> -v
   ```

3. **Fix the issue**: Read the test to understand expected behavior, then fix the source code.

4. **Re-run the specific test** to confirm the fix, then run the full suite again.

5. **Lint after fixes**:
   ```bash
   uv run ruff check --fix src/srtctl/
   uv run ruff format src/srtctl/
   ```

6. **Guidelines**:
   - Always add tests for new significant features
   - Use the mock SLURM patterns (H100Rack, B200Rack) for cluster simulation
   - Use frozen dataclasses for test fixtures
   - Follow existing test patterns in `tests/`
