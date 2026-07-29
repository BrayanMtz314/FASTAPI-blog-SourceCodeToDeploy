# Module 15: PostgreSQL and Alembic - Database Migrations for Production

During the development process, we used SQLite. While it is excellent for prototyping, it is not recommended for production environments where multiple users write to the database simultaneously. In this module, we switch to **PostgreSQL**. 

Additionally, we integrate **Alembic**, a database migration tool. Previously, if we wanted to add a column to a table, we had to delete the entire database and start over. Alembic allows us to implement incremental changes safely without losing existing data.

*(Note: We begin by deleting our local SQLite `blog.db` file. If you have data you want to keep, migrating data from SQLite to Postgres requires a separate export/import process beyond the scope of this module).*

## 1. Setting up PostgreSQL
You must install PostgreSQL on your machine (or run it via Docker). Once installed, we use the command line to create our application's database and user.

* **Accessing the Postgres Shell:**
```bash
# Switch to the postgres system user and start the shell
sudo -i -u postgres
psql
```
* **Creating the User and Database:**
```sql
postgres=# CREATE USER bloguser WITH PASSWORD 'your_password';
postgres=# CREATE DATABASE blog OWNER bloguser;
```
(Type `\q` to exit the shell).

* **Testing the Connection:** You can now log into your new database from your standard terminal:
```bash
psql -h localhost -U bloguser -d blog
```

## 2. Connecting FastAPI to PostgreSQL
We need to swap our database driver and update our configuration.

* **Install the psycopg Driver:**
```bash
uv add "psycopg[binary]"
```

* **Environment Variables (`.env`):** Move the database URL out of the code and into your secure `.env` file.
```txt
DATABASE_URL=postgresql+psycopg://bloguser:your_password@localhost/blog
```

* **Update `database.py`:** Update the engine to use the new URL.
```python
engine = create_async_engine(settings.database_url)
```

## 3. Introducing Alembic (Migrations)
Until now, we used `Base.metadata.create_all` in our lifespan event to build tables. However, this method only creates tables if they don't exist; it cannot alter them (like adding a new column). We need Alembic to manage these changes over time.

* **Installation & Initialization:**
```bash
uv add alembic
# Initialize Alembic configured for async SQLAlchemy
uv run alembic init -t async alembic
```

* **Configuring Alembic (`alembic/env.py`):** We must point Alembic to our database URL and our SQLAlchemy models so it knows what to look at. Add the following:
```python
import models # noqa: F401 (Tells linters to ignore the unused import warning)
from alembic import context
from config import settings
from database import Base

config = context.config
config.set_main_option("sqlalchemy.url", settings.database_url)

target_metadata = Base.metadata
```

## 4. Generating and Applying Migrations
With Alembic configured, we create our very first migration script representing our current database models.

* **Autogenerate the Schema:**
```bash
uv run alembic revision --autogenerate -m "initial schema"
```
**What did this do?** Alembic looked at the empty PostgreSQL database and compared it to your Python `models.py`. It detected that the database was missing the Users, Posts, and Token tables, and automatically wrote a Python script inside the `alembic/versions` folder containing the exact instructions needed to build them.

* **Apply the Migration:**
```bash
uv run alembic upgrade head
```
***(This actually executes the script and creates the tables in PostgreSQL. You can verify this by opening `psql` and typing `\dt` to list your tables).***

* **Check Status:** `Use uv run alembic current` to see which migration version your database is currently on.

## 5. Updating the Schema Without Data Loss
To prove Alembic works, let's add a "likes" column to our existing Posts table.

* **Update `models.py`:**
```python
# server_default="0" is crucial! It tells Postgres to fill existing rows with a 0.
# Without it, adding a new non-nullable column to a table that already has data will cause a crash.
likes: Mapped[int] = mapped_column(Integer, default=0, server_default="0")
```

* **generate and apply:**
```bash
uv run alembic revision --autogenerate -m "add likes to posts" 
uv run alembic upgrade head
```

## 6. Timezone Handling (Postgres vs. SQLite)
PostgreSQL handles timezone-aware datetime objects natively and strictly. Because of this, we can clean up a line of code in our users.py password reset logic:

* **Old Code (`SQLite workaround`):**
```python
if reset_token.expires_at.replace(tzinfo=UTC) < datetime.now(UTC):
```
* **New Code (Postgres native):**
```python
if reset_token.expires_at < datetime.now(UTC):
```

## 7. Migration Management Commands
If you make a mistake, Alembic allows you to travel back in time.

* **View History:** `uv run alembic history` (Shows all past migrations).

* **Rollback:** `uv run alembic downgrade -1` (Undoes the last applied migration).

## 8. Production Best Practices

* **Limitations:** Alembic's `--autogenerate` does not automatically detect all changes out of the box (e.g., if you change a string column length from 30 to 90 characters). Always review the generated script in the `versions/` folder before applying it!

* **The Production Workflow:** You should never run `--autogenerate` on your live production server. The correct workflow is:
    1. Make model changes on your local machine.
    2. Run --autogenerate locally to create the migration file.
    3. Commit the migration file to Git (version control).
    4. Pull the code to your production server and simply run alembic upgrade head.

## Conclusion
By completing this module, you have bridged the gap between a local tutorial project and a real-world software architecture. PostgreSQL provides the performance, strict data integrity, and concurrency required for a live web application. Meanwhile, Alembic acts as "version control for your database," ensuring that as your application evolves—adding new features, columns, and tables—your data remains safe, structured, and perfectly synchronized with your code.

# Return to Readme.md
[**Readme.md**](../README.md)