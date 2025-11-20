# Harbor Data gen

This directory contains Harbor format tasks that are generated, validated, built, and reviewed by multiple Claude Code agents. This is a proof of concept of using agents to scale the data generation process for evaluation as well as trianing. 

## Tasks

### auth_token_race_condition

**Difficulty**: Hard
**Category**: Backend Engineering
**Tags**: python, fastapi, redis, race-condition, concurrency, distributed-systems, jwt, authentication

**Description**: Fix a race condition in a FastAPI authentication gateway where concurrent token refresh requests cause random user logouts. The `/auth/refresh` endpoint has non-atomic read-validate-delete-create operations that fail when multiple gateway instances process the same refresh token simultaneously.

**Bug**: The token refresh logic in `auth_gateway/token_service.py` (lines 84-136) uses separate Redis GET, DELETE, and SET operations without atomicity, allowing race conditions when concurrent requests process the same token.

**Solution**: Implement atomic operations using Redis Lua scripts or GETDEL to ensure only one request can successfully use a refresh token.

**Test Coverage**: 3 comprehensive tests
1. Concurrent refresh with same token (10 requests) - validates atomicity
2. Sequential token rotation (5 iterations) - validates correctness
3. Multi-user stress test (100 concurrent requests, 20 users) - validates system under load

**Environment**:
- Python 3.11 with FastAPI
- Redis for token storage
- JWT-based authentication
- Multi-worker gateway instances

**Agent Timeout**: 600 seconds
**Verifier Timeout**: 180 seconds

**Performance (Claude Code + Sonnet 4.5)**:
- Success Rate: 100% (1/1 test run)
- Reward: 1.0 (all tests passed)
- Execution Time: ~7-8 minutes
- Token Usage: ~1.7M input (99.9% cached), ~20K output

**Status**: ✅ READY FOR PRODUCTION

---

### fix_async_worker_queue

**Difficulty**: Hard
**Category**: Backend Engineering
**Tags**: python, fastapi, async, concurrency, debugging, workers, race-condition

**Description**: Debug a broken asynchronous job queue system with multiple interconnected bugs in FastAPI and asyncio workers.

**Bugs**: Four intentional issues in `app/worker_queue.py`:
1. Workers never start (initialization flaw)
2. Worker event loop broken (control flow prevents job retrieval)
3. Jobs not queued (API accepts but drops them)
4. Status never updates (state transitions missing)

**Solution**: Fix all four bugs to enable proper async job processing with state management.

**Test Coverage**: 11 comprehensive tests
1. Health endpoint responds
2. Workers initialize correctly
3. Job submission works
4. Status endpoint functional
5. 404 handling for missing jobs
6. Jobs actually process
7. Timestamps set correctly
8. Concurrent processing works
9. Queue management correct
10. Data integrity maintained
11. State transitions (PENDING → PROCESSING → COMPLETED)

**Environment**:
- Python 3.11 with FastAPI
- AsyncIO worker pool
- In-memory job store
- Queue-based job distribution

**Agent Timeout**: 600 seconds
**Verifier Timeout**: 180 seconds

**Performance (Claude Code + Sonnet 4.5)**:
- Success Rate: 91% (10/11 tests - 1 run)
- Reward: 0.0 (needs 11/11 for 1.0)
- Execution Time: ~3-4 minutes
- Token Usage: ~500K input (cached), ~15K output

**Bugs Fixed by Agent**:
- ✅ Worker initialization (100%)
- ✅ Event loop control flow (100%)
- ✅ Job queueing logic (100%)
- ⚠️ State management (10/11 tests pass)

**Example Runs**: See `example_runs/` directory
- `partial_success_2025-11-19/` - Agent fixed 3.5/4 bugs, 91% test pass rate

**Status**: ⚠️ VALIDATED - Infrastructure perfect, agent achieves 91% (nearly complete)


---

## Validation Criteria

For a task to be marked as "READY FOR PRODUCTION":

1. Docker environment builds successfully
2. All services start correctly
3. Tests are discoverable and run
4. Tests pass/fail correctly based on solution quality
5. Reward is calculated and persisted correctly
6. Agent can solve the task end-to-end
7. Clear, realistic problem statement
8. Appropriate difficulty level
