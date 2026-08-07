# Module 17: Testing The API - Pytest, Fixtures, and Mocking

By now, we have a complete and deployment-ready application. We've built workflows for image uploads, CRUD operations, authentication, and database migrations. However, best practices demand one final, critical step: **Testing**. 

Testing is how we guarantee our code works correctly and, more importantly, ensures that future updates don't accidentally break existing features (preventing regressions). Testing is not optional in professional environments. In this module, we implement automated testing for our FastAPI application.

## 1. Testing Dependencies
We need to install specific testing libraries into our development environment using the `--dev` flag so they aren't included in our production server builds.

```bash
uv add --dev pytest
uv add --dev "moto[s3]"
```
* `pytest`: The industry-standard testing framework for Python. It allows us to write simple, readable tests and provides a powerful "fixture" system to manage test setups (like creating temporary databases).

* `moto[s3]`: A mocking library for AWS services. It intercepts our application's `boto3` calls and fakes an S3 bucket in our computer's memory. This allows us to test our image upload logic without hitting the internet or spending actual money on AWS.

## 2. Test Directory Structure
We organize our tests into a dedicated `/tests` folder at the root of the project:

* `__init__.py`: Tells Python this folder is a module.

* `conftest.py`: A special pytest file used to share configurations and fixtures across all test files automatically.

* `test_posts.py`: Contains tests specifically for post-related endpoints.

* `test_users.py`: Contains tests specifically for user-related endpoints.

## 3. The Basics of Testing (Demo)
Before testing our complex async API, we created a small `test_demo.py` to understand the mechanics.

```python
from fastapi import FastAPI
from fastapi.testclient import TestClient

demo_app = FastAPI()

@demo_app.get("/")
def demo_home():
    return {"message": "Hello"}

client = TestClient(demo_app)

def test_homepage():
    # 1. Arrange & Act: Make a request to the endpoint
    response = client.get("/")
    # 2. Assert: Verify the outcome is exactly what we expect
    assert response.status_code == 200
    assert response.json() == {"message": "Hello"}
```

**How it works:** The `TestClient` spins up a simulated version of our API. We capture its `response` and use the `assert` keyword to check if a condition is `True`. If an `assert` statement is `False`, the test immediately fails, letting us know our code is broken.

## 4. Configuring Pytest (`conftest.py`)
Testing our actual application is harder because it uses asynchronous code and external services (PostgreSQL and AWS). We use `conftest.py` to set up this environment.

* **Environment Variables:** We use `os.environ` to inject fake AWS credentials and point the app to a special `test_bucket` so it never touches our real configuration.

* **Async Setup:** We configure the `anyio_backend` fixture to tell Pytest to run our async tests using `asyncio`.

### The Database Fixtures
We created a separate PostgreSQL database named `test_blog` specifically for testing so we don't accidentally delete real data.
```bash
sudo -u postgres createdb test_blog -O bloguser
```

To manage this database during tests, we wrote three crucial fixtures in `conftest.py`:

1. `test_engine`: Creates the SQLAlchemy asynchronous engine connected specifically to the `test_blog` database.

2. `setup_database`: Runs exactly once per test session. It builds all our database tables (`Base.metadata.create_all`) before the tests begin and drops them completely when all tests finish.

3. **`db_session` (The Rollback Pattern):** This is the most important fixture. It runs before every single test. It opens a database transaction, yields control to the test, and when the test is done, it intentionally rolls back (cancels) the transaction. This guarantees that every test starts with a completely clean, empty database, preventing data from one test from corrupting another.

### The AWS Mock Fixture
* `mocked_aws`: Uses the `moto` library to trick `boto3`. Because our `test_engine` and database handles async behavior natively, this mock fixture can remain synchronous while successfully faking our S3 environment.

## 5. Helper Functions
To avoid rewriting the same code in every test, we created helper functions in `conftest.py`:

* `create_test_user`: Registers a dummy user in the database.

* `login_user`: Hits the `/token` endpoint and returns a valid JWT.

* `auth_header`: Formats the JWT into a standard `{"Authorization": "Bearer <token>"}` dictionary, making it easy to inject into our test requests.

## 6. Writing Our First Real Test
In `test_posts.py`, we test the pagination logic on an empty database.

```python
import pytest
from httpx import AsyncClient

@pytest.mark.anyio
async def test_get_posts_empty(client: AsyncClient):
    response = await client.get("/api/posts")

    assert response.status_code == 200
    
    data = response.json()
    assert data["posts"] == []
    assert data["total"] == 0
    assert data["has_more"] is False
```
**Explanation of the Test:**
Because of our rollback pattern, we know the database is completely empty when this test runs. We send a `GET` request to `/api/posts`. We assert that the API doesn't crash (status `200`), and we verify that our pagination logic correctly handles an empty dataset by returning an empty list, a total count of `0`, and setting `has_more` to `False`.

## 7. Running Tests  
You can execute your tests from the terminal:

* Run all tests in a file:
```bash
uv run pytest tests/test_posts.py -v
```
* Run a single specifc test:
```bash
uv run pytest tests/test_posts.py::test_get_posts_empty -v
```
(The `-v` flag stands for verbose, which provides a cleaner, more detailed output in the console).

## Conclusion
By completing this module, you have wrapped a safety net around your entire application. You've learned how to isolate test environments using separate databases, mock external cloud services like AWS S3 to save time and money, and implement the rollback pattern to ensure pristine testing conditions. With a solid test suite in place, you can now refactor code, upgrade libraries, or add new features with total confidence, knowing that if anything breaks, Pytest will catch it immediately.

# Return to Readme.md
[**Readme.md**](../README.md)