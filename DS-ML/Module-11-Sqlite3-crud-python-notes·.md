# Working with SQLite3 in Python — CRUD Operations

*Day X of my learning journal — Krish Naik's Complete Python Bootcamp*

So today I got into databases, specifically SQLite3. Before this I only knew "databases" as this scary backend thing. Turns out Python has a database engine literally built into it, no installation, no server, nothing. That's SQLite3 for you. Writing this note the way I wish someone had explained it to me on day 1.

---

## 1. What even is SQLite3, and why should I care?

Think of SQLite3 as a **database in a single file**. No server running in the background, no username/password setup, no "start the database service" step. You just point Python at a `.db` file (or even create one on the fly) and you're good to go.

That's exactly why it's the best database to learn CRUD on before jumping into Postgres/MySQL later. Same core SQL concepts, zero setup headache.

`sqlite3` comes pre-installed with Python. You don't `pip install` anything:

```python
import sqlite3
```

That's it. That one import unlocks the whole database.

---

## 2. Connecting to a database

Every interaction with SQLite3 follows the same pattern, and once this clicks, everything else in this note is just "swap the SQL query":

```
connect → get a cursor → execute query → commit (if you changed data) → close
```

Here's the connection step:

```python
def create_database():
    conn = sqlite3.connect('test.db')
    conn.close()
    print("Database created and successfully connected.")

create_database()
```

**The analogy that helped me:** `conn` is like opening the front door to the database building. The `cursor` is the person you actually talk to once you're inside — it's the thing that runs your SQL commands and hands you back results. You don't talk to the building directly, you talk through the cursor.

Important detail: `sqlite3.connect('test.db')` will **create the file if it doesn't exist**. So "creating a database" and "connecting to a database" are basically the same line of code in SQLite3. That confused me at first — I expected some separate "create" step.

---

## 3. Creating a table

Once connected, you create tables the exact same way you'd write raw SQL, just wrapped inside `cursor.execute()`:

```python
def create_table():
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS employees (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL,
            age INTEGER,
            department TEXT
        )
    ''')
    conn.commit()
    conn.close()
    print("Table 'employees' created successfully.")

create_table()
```

Things I made sure to actually understand instead of just copy-pasting:

- `IF NOT EXISTS` — stops the script from throwing an error if you accidentally run it twice. Basically a safety net for re-runs.
- `id INTEGER PRIMARY KEY` — this is the unique identifier for each row. No two employees can have the same `id`.
- `conn.commit()` — this is the part that actually **saves** the change to disk. If you skip this, SQLite quietly does nothing permanent. I forgot this once and spent 10 minutes wondering why my table "disappeared."

---

## 4. The four CRUD operations

CRUD = **C**reate, **R**ead, **U**pdate, **D**elete. This is basically 90% of what any app does to a database. Once I mapped SQL keywords to these four buckets, database work stopped feeling random.

| CRUD | SQL keyword | What it does |
|------|-------------|--------------|
| Create | `INSERT INTO` | adds a new row |
| Read | `SELECT` | fetches rows |
| Update | `UPDATE` | changes existing rows |
| Delete | `DELETE` | removes rows |

### 4.1 Create — inserting data

```python
def insert_employee(id, name, age, department):
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO employees (id, name, age, department)
        VALUES (?, ?, ?, ?)
    ''', (id, name, age, department))
    conn.commit()
    conn.close()
    print("Employee inserted successfully.")

insert_employee(1, 'Alice', 30, 'HR')
insert_employee(2, 'Bob', 25, 'Engineering')
insert_employee(3, 'Charlie', 28, 'Sales')
insert_employee(4, 'David', 35, 'Marketing')
insert_employee(5, 'Eve', 22, 'HR')
```

**This is the single most important habit I picked up from this module:** notice the `?` placeholders instead of just shoving the values directly into the string with f-strings. This is called a **parameterized query**, and it exists for one big reason — **SQL injection**.

If I'd written this instead:

```python
cursor.execute(f"INSERT INTO employees VALUES ({id}, '{name}', {age}, '{department}')")
```

...and someone passed a `name` like `'); DROP TABLE employees;--`, congrats, my table is gone. The `?` placeholder tells SQLite "treat this strictly as data, never as part of the SQL command." Never build queries with string formatting. Always use `?` and pass values as a tuple.

### 4.2 Read — querying data

```python
def fetch_all_employees():
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM employees')
    records = cursor.fetchall()
    conn.close()
    for record in records:
        print(record)

fetch_all_employees()
```

Filtering by a condition works exactly the same way, just with a `WHERE` clause and (again) a placeholder:

```python
def fetch_employees_by_department(department):
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM employees WHERE department = ?', (department,))
    records = cursor.fetchall()
    conn.close()
    for record in records:
        print(record)

fetch_employees_by_department('HR')
```

Quick note to self on `fetchall()` vs alternatives:
- `fetchall()` → gives you every matching row as a list of tuples
- `fetchone()` → gives you just the next single row (good when you expect exactly one result)
- `fetchmany(n)` → gives you `n` rows at a time (useful for huge tables so you don't load everything into memory)

More read patterns I practiced — same shape, different `WHERE`:

```python
# employees older than a given age
cursor.execute('SELECT * FROM employees WHERE age > ?', (age,))

# employees whose name starts with a letter (pattern matching)
cursor.execute('SELECT * FROM employees WHERE name LIKE ?', (letter + '%',))
```

`LIKE` + `%` is SQL's version of "starts with / contains." The `%` is a wildcard — `'A%'` means "starts with A," `'%a%'` means "contains a anywhere."

### 4.3 Update — modifying data

```python
def update_employee_department(employee_id, new_department):
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    cursor.execute('''
        UPDATE employees
        SET department = ?
        WHERE id = ?
    ''', (new_department, employee_id))
    conn.commit()
    conn.close()
    print("Employee department updated successfully.")

update_employee_department(1, 'Finance')
```

The one line that will bite you if you're not careful: **the `WHERE` clause.** Forget it, and `UPDATE` applies to *every single row* in the table. I tested this on purpose once on a throwaway `.db` file just to see it happen — genuinely scary, don't skip `WHERE`.

### 4.4 Delete — removing data

```python
def delete_employee(employee_id):
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    cursor.execute('''
        DELETE FROM employees
        WHERE id = ?
    ''', (employee_id,))
    conn.commit()
    conn.close()
    print("Employee deleted successfully.")

delete_employee(5)
```

Same warning as `UPDATE` — `DELETE FROM employees` with no `WHERE` wipes the whole table. `DELETE` and `UPDATE` are basically the two commands where forgetting `WHERE` is the most expensive mistake you can make.

---

## 5. Transactions — the "all or nothing" safety net

This is the concept that made me respect databases a lot more. A **transaction** is a group of operations that either *all* succeed, or *none* of them do. No half-done state.

```python
def insert_multiple_employees(employees):
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    try:
        cursor.executemany('''
            INSERT INTO employees (id, name, age, department)
            VALUES (?, ?, ?, ?)
        ''', employees)
        conn.commit()
        print("All employees inserted successfully.")
    except Exception as e:
        conn.rollback()
        print("Error occurred, transaction rolled back.")
        print(e)
    finally:
        conn.close()
```

I deliberately tested this with a duplicate `id` in the batch (which violates the `PRIMARY KEY` rule):

```python
employees = [
    (6, 'Frank', 40, 'Finance'),
    (7, 'Grace', 29, 'Engineering'),
    (8, 'Hannah', 35, 'Marketing'),
    (9, 'Ivan', 38, 'Sales'),
    (6, 'Jack', 45, 'HR')  # duplicate id — this breaks it
]
insert_multiple_employees(employees)
```

Result: **none** of the 5 rows get inserted, not even the 4 valid ones. That's `conn.rollback()` doing its job — it undoes everything back to the last commit.

**Analogy that stuck with me:** it's like an "undo everything" button that only appears if something in the batch goes wrong. Either the whole batch lands, or the database pretends none of it ever happened.

`executemany()` is also worth remembering separately — it's the batch version of `execute()`. Instead of looping and calling `insert_employee()` five times (five separate connections/commits), you hand it a list of tuples once. Same idea applies to bulk updates:

```python
cursor.executemany('''
    UPDATE employees
    SET age = ?
    WHERE id = ?
''', updates)
```

---

## 6. Relationships — connecting two tables with foreign keys

Real-world data is never one flat table. Employees belong to departments, orders belong to customers, etc. SQLite lets you model that with a **foreign key**.

First, a separate `departments` table:

```python
def create_departments_table():
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS departments (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL
        )
    ''')
    conn.commit()
    conn.close()
```

Then a `department_id` column on `employees` that points back to `departments.id`:

```sql
FOREIGN KEY(department_id) REFERENCES departments(id)
```

**The mental model that helped:** a foreign key is basically a "pointer" — instead of writing `'Finance'` as plain text in every single employee row (and risking typos like `'finance'` vs `'Finance'` scattered everywhere), you store the department **once** in its own table and just reference its `id` everywhere else. One source of truth.

Inserting into both tables together, safely wrapped in a try/except so referential integrity doesn't break halfway:

```python
def insert_department_and_employee(department_id, department_name, employee_id, name, age, department):
    conn = sqlite3.connect('test.db')
    cursor = conn.cursor()
    try:
        cursor.execute('INSERT INTO departments (id, name) VALUES (?, ?)',
                        (department_id, department_name))
        cursor.execute('''
            INSERT INTO employees (id, name, age, department, department_id)
            VALUES (?, ?, ?, ?, ?)
        ''', (employee_id, name, age, department, department_id))
        conn.commit()
        print("Department and employee inserted successfully.")
    except Exception as e:
        conn.rollback()
        print("Error occurred, transaction rolled back.")
    finally:
        conn.close()
```

---

## 7. Indexing — making lookups faster

An **index** is basically a shortcut lookup table SQLite maintains internally so it doesn't have to scan every single row every time you search.

```python
cursor.execute('CREATE INDEX idx_name ON employees(name)')
```

**Analogy:** think of a textbook. Without an index, "finding all mentions of a topic" means flipping through every page one by one (this is called a **full table scan**). With an index, you jump straight to the page numbers listed at the back. Same idea — SQLite keeps a sorted lookup structure on the `name` column so `WHERE name LIKE 'A%'` doesn't need to check every row.

I timed it with `time.time()` before/after adding the index, just to see the difference for myself instead of taking it on faith:

```python
import time

start_time = time.time()
cursor.execute('SELECT * FROM employees WHERE name LIKE ?', (letter + '%',))
records = cursor.fetchall()
end_time = time.time()
print("Time taken: {} seconds".format(end_time - start_time))
```

On a tiny table like this the difference is basically invisible — indexing shows its real value once you're dealing with thousands/millions of rows. But the concept matters more than the number right now.

**Trade-off worth remembering:** indexes speed up reads but slightly slow down writes (every `INSERT`/`UPDATE` now also has to update the index). Don't index every column "just in case" — index columns you actually filter/search on a lot.

---

## 8. Backup and restore

SQLite databases are literally just files, so backing one up can be as simple as copying the file:

```python
import shutil

def backup_database():
    shutil.copy('test.db', 'backup.db')
    print("Database backed up successfully.")

def restore_database():
    shutil.copy('backup.db', 'test.db')
    print("Database restored successfully.")
```

This is the "cheap but effective" way to do it for small local projects. (For production systems you'd usually use SQLite's own `.backup()` API or proper dump/restore tooling so you don't risk copying a file mid-write — but for learning purposes, this file-copy approach is honestly a great way to *feel* why "it's just a file" makes SQLite so simple.)

---

## 9. Working with a more realistic table — sales data

Everything above used the `employees` table, which is great for learning CRUD but doesn't really look like data I'd pull from a real business system. So I redid the same core pattern on a `sales` table instead, just to prove to myself the pattern holds for *any* table shape, not just the one example.

```python
connection = sqlite3.connect('sales_data.db')
cursor = connection.cursor()

cursor.execute('''
    CREATE TABLE IF NOT EXISTS sales (
        id INTEGER PRIMARY KEY,
        date TEXT NOT NULL,
        product TEXT NOT NULL,
        sales INTEGER,
        region TEXT
    )
''')

sales_data = [
    ('2023-01-01', 'Product1', 100, 'North'),
    ('2023-01-02', 'Product2', 200, 'South'),
    ('2023-01-03', 'Product1', 150, 'East'),
    ('2023-01-04', 'Product3', 250, 'West'),
    ('2023-01-05', 'Product2', 300, 'North')
]

cursor.executemany('''
    INSERT INTO sales (date, product, sales, region)
    VALUES (?, ?, ?, ?)
''', sales_data)

connection.commit()

cursor.execute('SELECT * FROM sales')
for row in cursor.fetchall():
    print(row)
```

Nothing new syntax-wise here — same `CREATE TABLE`, same `executemany()` with placeholders I already knew, same `commit()`. The point of redoing it on a different dataset was really just to confirm the CRUD pattern isn't something special about the `employees` example — it's the pattern for *any* table.

One thing worth calling out: `date` is stored as `TEXT` here, not a dedicated date type. SQLite doesn't have a strict native date/datetime column type the way some other databases do — it's common to just store dates as ISO-format text (`'2023-01-01'`) and rely on SQLite's built-in date functions (or just plain string comparison, since ISO format sorts correctly as text) when I need to filter/sort by date.

### 9.1 Gotcha I ran into: querying after `close()`

I deliberately tried this, just to see what happens:

```python
connection.close()

cursor.execute('SELECT * FROM sales')   # this blows up
```

This throws a `sqlite3.ProgrammingError: Cannot operate on a closed database.` Makes sense once I think about it — `close()` isn't a "pause," it's a hard shutdown of that connection. Once it's closed, the `cursor` tied to it is dead too, and there's no bringing it back; I'd have to call `sqlite3.connect()` again to get a fresh connection.

**Lesson for myself:** `close()` should be the very last thing that happens with a connection — after every query and every `commit()` I actually need are already done. This is exactly the kind of bug a context manager (`with sqlite3.connect(...) as conn:`) protects against, since it only closes the connection once you've actually left the `with` block, not a line earlier by accident.

---

## 10. My personal cheat sheet — the pattern that repeats everywhere

Every single function above follows this exact skeleton:

```python
import sqlite3

def do_something():
    conn = sqlite3.connect('test.db')      # 1. connect
    cursor = conn.cursor()                  # 2. get a cursor
    cursor.execute('SQL QUERY', (params,))  # 3. run the query (with ? placeholders!)
    conn.commit()                           # 4. save changes (only needed if you modified data)
    conn.close()                            # 5. close the connection
```

Rules I'm making myself remember going forward:
1. Always use `?` placeholders, never f-strings, for anything going into a query — SQL injection is real.
2. Never forget `WHERE` on `UPDATE`/`DELETE` unless you genuinely mean "all rows."
3. `commit()` after every write, or it doesn't stick.
4. Wrap multi-step writes in `try / except / rollback / finally: close()` so a partial failure never leaves half-written data.
5. `close()` the connection when you're done, every time — leaving connections open is a resource leak.

Next up: probably wrapping this with a context manager (`with sqlite3.connect(...) as conn:`) so I stop manually closing connections everywhere, and eventually SQLAlchemy once I want an ORM instead of raw SQL.
