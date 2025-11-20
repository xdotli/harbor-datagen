# AGENTS.md

This document provides comprehensive guidance for AI coding agents working on the harbor-datagen repository. It supplements the README.md with detailed technical instructions specific to agent workflows.

## Project Overview

Harbor-datagen is an agent-driven data generation system for creating Harbor format evaluation tasks. This repository contains tasks that are:

- **Generated** by AI coding agents following specific patterns
- **Validated** through automated testing and verification
- **Built** using Docker containerization
- **Reviewed** for quality and difficulty calibration

The primary goal is to scale data generation for AI evaluation and training by leveraging agents to create realistic software engineering challenges.

## Repository Structure

Each task follows a standardized directory structure:

```
<task-name>/
├── instruction.md          # Natural language problem description
├── task.toml              # Task metadata and configuration
├── environment/           # Initial buggy/incomplete code
│   ├── Dockerfile        # Container build instructions
│   ├── requirements.txt  # Python dependencies
│   ├── start.sh          # Service startup script
│   └── app/              # Application code with bugs
├── solution/             # Reference solutions
│   ├── solve.sh         # Script that fixes the issue
│   └── reference/       # Working implementation
├── tests/                # Verification tests
│   ├── test.sh          # Main test runner
│   └── test_*.py        # Test cases
└── example_runs/         # Historical agent execution data
    └── <run-id>/
        ├── agent/       # Agent trajectory and commands
        ├── verifier/    # Test results and rewards
        └── result.json  # Final outcome
```

### Key Configuration Files

- **task.toml**: Contains metadata (author, difficulty, category, tags) and resource limits (timeouts, CPU, memory, storage)
- **instruction.md**: User-facing problem description written in natural language
- **Dockerfile**: Defines the isolated execution environment for the task
- **requirements.txt**: Python package dependencies with pinned versions

For complete task format specifications, see https://harborframework.com/docs/task-format

## Task Development Workflow

### Creating a New Task

1. **Create the directory structure**:
   ```bash
   mkdir -p <task-name>/{environment,solution/reference,tests,example_runs}
   ```

2. **Write instruction.md**:
   - Use natural, realistic language (avoid clinical descriptions)
   - Describe symptoms from a user's perspective
   - Include context about when the issue occurs
   - Example: "Users are getting randomly logged out under load" vs "There is a race condition"

3. **Configure task.toml**:
   ```toml
   version = "1.0"

   [metadata]
   author_name = "Your Name"
   author_email = "you@example.com"
   difficulty = "hard"  # easy, medium, hard
   category = "backend-engineering"
   tags = ["python", "fastapi", "redis", "race-condition"]

   [verifier]
   timeout_sec = 180.0

   [agent]
   timeout_sec = 600.0

   [environment]
   build_timeout_sec = 600.0
   cpus = 2
   memory_mb = 4096
   storage_mb = 20480
   ```

4. **Build the environment**:
   - Create Dockerfile with base image and dependencies
   - Add buggy application code to environment/
   - Include start.sh script for service initialization
   - Ensure services expose health check endpoints

5. **Create reference solution**:
   - Place working code in solution/reference/
   - Write solution/solve.sh that applies the fix
   - Test locally to ensure it resolves the issue

6. **Write verification tests**:
   - Create tests/test.sh as the main entry point
   - Use pytest for Python test cases (test_*.py)
   - Tests must be fully automated (no human interaction)
   - Calculate and write reward to /logs/verifier/reward.txt

### Testing Your Task

Before submitting a task for agent evaluation:

```bash
# Build the Docker environment
cd <task-name>/environment
docker build -t <task-name>-test .

# Run the tests manually
docker run -it <task-name>-test bash
cd /tests
./test.sh

# Verify reward was written correctly
cat /logs/verifier/reward.txt  # Should output "0" or "1"
```

### Validation Criteria Checklist

A task is marked **✅ READY FOR PRODUCTION** when:

1. ✓ Docker environment builds successfully
2. ✓ All services start correctly
3. ✓ Tests are discoverable and run via tests/test.sh
4. ✓ Tests pass/fail correctly based on solution quality
5. ✓ Reward is calculated and persisted to /logs/verifier/reward.txt
6. ✓ Agent can solve the task end-to-end
7. ✓ instruction.md has clear, realistic problem statement
8. ✓ Appropriate difficulty level and resource limits

## Build and Test Commands

### Docker Environment Setup

```bash
# Build environment image
docker build -t <task-name> environment/

# Run environment interactively
docker run -it --rm <task-name> bash

# Run with resource limits (matching task.toml)
docker run -it --rm \
  --cpus=2 \
  --memory=4g \
  <task-name> bash
```

### Running Tests

```bash
# Standard test execution
cd tests && ./test.sh

# With verbose output
cd tests && bash -x ./test.sh

# Run specific pytest test
python -m pytest tests/test_outputs.py -v --tb=short

# Check reward file
cat /logs/verifier/reward.txt
```

### Local Development

```bash
# Install dependencies
pip install -r environment/requirements.txt

# Run services locally (if applicable)
cd environment && ./start.sh

# Apply reference solution
cd solution && ./solve.sh

# Run tests against solution
cd tests && ./test.sh
```

## Code Style and Conventions

### Python

- **Style**: Follow PEP 8 conventions
- **Type Hints**: Encouraged for function signatures
- **Async Code**: Use proper async/await patterns, avoid blocking calls
- **Error Handling**: Include appropriate exception handling
- **Logging**: Use Python's logging module, not print statements

Example:
```python
import logging
from typing import Optional

logger = logging.getLogger(__name__)

async def process_job(job_id: str) -> Optional[dict]:
    """Process a job and return its result."""
    try:
        result = await fetch_data(job_id)
        return result
    except Exception as e:
        logger.error(f"Failed to process job {job_id}: {e}")
        return None
```

### Test Naming

- Test files: `test_*.py` for pytest discovery
- Test shell scripts: `test.sh` as main entry point
- Test functions: `test_<behavior>()` descriptive names

### Configuration Files

- **TOML**: Use for task.toml metadata
- **YAML**: For Docker Compose or complex configs (if needed)
- **Shell Scripts**: Use `#!/bin/bash` and `set -e` for fail-fast behavior

### Documentation

- **instruction.md**: Natural language, no code formatting
- **Code Comments**: Explain "why" not "what"
- **Docstrings**: Use for public functions and classes
- **README.md**: Human-focused project overview

## Testing Requirements

### Test Structure

All tests must be executable via `tests/test.sh`:

```bash
#!/bin/bash
set -e

# Start services in background
/workspace/start.sh &
APP_PID=$!

# Create logs directory
mkdir -p /logs/verifier

# Wait for services to be ready
echo "Waiting for service..."
timeout=30
counter=0
until curl -s http://localhost:8000/health > /dev/null 2>&1; do
    sleep 1
    counter=$((counter + 1))
    if [ $counter -ge $timeout ]; then
        echo "Service failed to start"
        echo "0" > /logs/verifier/reward.txt
        exit 1
    fi
done

# Run pytest tests
python -m pytest /tests/test_outputs.py -v --tb=short

# Write reward based on result
if [ $? -eq 0 ]; then
    echo "1" > /logs/verifier/reward.txt
else
    echo "0" > /logs/verifier/reward.txt
fi
```

### Test Requirements

- **Discoverable**: Tests must run via tests/test.sh without manual setup
- **Automated**: No human interaction or input required
- **Deterministic**: Same code should produce same results
- **Fast**: Complete within verifier timeout (typically 180s)
- **Comprehensive**: Cover success cases, edge cases, and failure modes
- **Isolated**: Clean up resources, don't depend on external state

### Reward Calculation

Reward must be written to `/logs/verifier/reward.txt`:

- **1.0** or **1**: All tests passed, perfect solution
- **0.0** or **0**: Tests failed or solution incomplete

Some tasks may use partial rewards (e.g., 0.91 for 10/11 tests passing), but binary 0/1 is preferred.

### Service Readiness Checks

Always wait for services to be ready before running tests:

```bash
# Wait for HTTP service
until curl -s http://localhost:8000/health > /dev/null 2>&1; do
    sleep 1
done

# Wait for Redis
until redis-cli ping > /dev/null 2>&1; do
    sleep 1
done

# Wait for PostgreSQL
until pg_isready -h localhost -p 5432 > /dev/null 2>&1; do
    sleep 1
done
```

## Environment Specifications

### Docker Containers

All tasks run in isolated Docker containers with:

- **Base Images**: Use official images (python:3.11-slim, node:18-alpine, etc.)
- **Dependencies**: Pin versions in requirements.txt or package.json
- **Ports**: Expose necessary ports (8000 for FastAPI, 5432 for PostgreSQL, etc.)
- **Working Directory**: Typically /workspace or /app
- **Entrypoint**: start.sh script that initializes all services

### Resource Limits (from task.toml)

Standard configuration for most tasks:

```toml
[environment]
build_timeout_sec = 600.0  # 10 minutes to build Docker image
cpus = 2                   # 2 CPU cores
memory_mb = 4096          # 4GB RAM
storage_mb = 20480        # 20GB disk space
```

Adjust based on task requirements:
- Database-heavy tasks may need more memory (8192 MB)
- Build-intensive tasks may need longer build timeout (900s)

### Service Initialization

The environment/start.sh script should:

1. Start background services (databases, caches)
2. Wait for services to be ready
3. Initialize databases or load fixtures
4. Start the main application
5. Provide health check endpoints

Example start.sh:

```bash
#!/bin/bash
set -e

# Start Redis in background
redis-server --daemonize yes

# Wait for Redis
until redis-cli ping > /dev/null 2>&1; do
    sleep 1
done

# Start FastAPI application
cd /workspace
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 3 &

echo "All services started successfully"
```

## Agent-Specific Guidance

### Expected Timeout Windows

Configure timeouts based on task complexity:

- **Agent Timeout**: 600 seconds (10 minutes) - standard for most tasks
- **Verifier Timeout**: 180 seconds (3 minutes) - time to run tests
- **Build Timeout**: 600-900 seconds - time to build Docker image

If a task consistently requires more time, adjust in task.toml:

```toml
[agent]
timeout_sec = 900.0  # For complex debugging tasks

[verifier]
timeout_sec = 300.0  # For extensive test suites
```

### Trajectory and Session Data

Agent runs generate detailed execution data in example_runs/:

- **agent/trajectory.json**: Sequence of agent actions and observations
- **agent/command-*/**: Individual command executions with stdout/stderr
- **agent/sessions/**: Cursor IDE session data and debug logs
- **verifier/test-stdout.txt**: Full test output
- **verifier/reward.txt**: Final reward (0.0 to 1.0)
- **result.json**: Summary of run outcome

This data is useful for:
- Debugging task issues
- Understanding agent behavior
- Improving task design
- Training future models

### Performance Metrics to Track

When evaluating agent performance on tasks:

- **Success Rate**: Percentage of runs achieving reward = 1.0
- **Reward**: Average reward across runs
- **Execution Time**: Time from start to test completion
- **Token Usage**: Input/output tokens (and cache hit rate)
- **Command Count**: Number of terminal commands executed
- **Error Rate**: Frequency of failed commands or exceptions

### Status Markers

Tasks use status markers in README.md to indicate readiness:

- **✅ READY FOR PRODUCTION**: Fully validated, agent can solve it
- **⚠️ VALIDATED**: Infrastructure works, agent achieves partial success
- **🚧 IN DEVELOPMENT**: Task is being built and tested
- **❌ DEPRECATED**: Task is no longer maintained or used

### Debugging Agent Failures

If an agent fails to solve a task:

1. **Check environment**: Does Docker build and run correctly?
2. **Check tests**: Do tests pass with the reference solution?
3. **Check instruction.md**: Is the problem description clear?
4. **Review trajectory**: Where did the agent get stuck?
5. **Simplify if needed**: Break complex tasks into smaller subtasks

### Running Evaluations with Harbor

To evaluate an agent on a dataset of tasks:

```bash
# Run Harbor evaluation
harbor evaluate \
  --dataset harbor-datagen \
  --agent claude-code \
  --model claude-sonnet-4 \
  --tasks auth_token_race_condition,fix_async_worker_queue

# View results
harbor results --run-id <run-id>
```

See https://harborframework.com/docs/datasets for full documentation.

## Additional Resources

- **Harbor Framework**: https://harborframework.com/docs/task-format
- **agents.md Specification**: https://agents.md/
- **Anthropic Claude Best Practices**: https://www.anthropic.com/engineering/claude-code-best-practices
- **Repository README**: See README.md for high-level overview and task summaries

## Contributing

When adding new tasks to harbor-datagen:

1. Follow the directory structure and naming conventions
2. Test your task thoroughly before committing
3. Include example_runs/ directory with at least one successful agent run
4. Update README.md with task description and performance metrics
5. Use descriptive commit messages

For questions or issues, contact the task author listed in task.toml metadata.

