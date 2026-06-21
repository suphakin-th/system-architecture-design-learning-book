# Database — The 3-Layer Model (ANSI/SPARC Architecture)

> "The three-schema architecture separates the users from the physical database. Change the disk storage, and users see nothing. Change the user view, and the disk doesn't care." — ANSI/SPARC Committee, 1975

---

## At a Glance

| | |
|---|---|
| **Standard** | ANSI/SPARC (1975) — the foundation of all modern DBMS |
| **Purpose** | Separate HOW data is stored from WHAT data means from WHO sees what |
| **Also called** | 3-schema architecture, 3-tier DBMS architecture |
| **Key benefit** | Data independence — change one layer without breaking others |

---

## The Problem This Solves

**Without the 3-layer model:**

```
Scenario 1: You change the disk storage format
→ Every application that reads the data BREAKS
→ Need to update 50 application files

Scenario 2: You add a new column to a table
→ Every user view that shows that table BREAKS
→ Need to notify all 200 users

Scenario 3: A junior developer can see salary data for all employees
→ No access control at the data level
→ Security nightmare
```

**With the 3-layer model:**
```
Change disk storage format → Only Internal Layer changes → Applications unaffected ✓
Add new column → Only Conceptual Layer changes → Views adapt, apps unaffected ✓
Junior developer → Gets External View without salary column → Security enforced ✓
```

---

## The Two Different "3-Layer" Models (Don't Confuse Them)

There are actually **two important 3-layer models** for databases. Both matter:

```
Model 1: ANSI/SPARC (Schema Architecture) — How a DBMS organizes itself internally
  External Schema (View Level)
  Conceptual Schema (Logical Level)
  Internal Schema (Physical Level)
  → This is about DATA ABSTRACTION inside the database system

Model 2: Data Modeling Levels — How you DESIGN a database
  Conceptual Model (business concepts, no tech)
  Logical Model (tables, relationships, no DB-specific)
  Physical Model (actual SQL for a specific DB)
  → This is about DATABASE DESIGN PROCESS

Both use 3 layers. Both call them different names. This document covers BOTH.
```

---

## Model 1: ANSI/SPARC — The 3-Schema Architecture

### Layer 1 — External Schema (View Level) — "What each user sees"

The outermost layer. Each user or application gets its own **custom view** of the database — showing only the data they need, in the format they need it.

```
Different users, different views of the SAME underlying data:

HR Manager View:
  employee_view: (emp_id, name, department, hire_date, salary, performance_score)

Payroll System View:
  payroll_view: (emp_id, name, bank_account, salary, tax_rate, deductions)

Project Manager View:
  team_view: (emp_id, name, department, current_project, skills)

Junior Developer View:
  staff_directory: (emp_id, name, department, email, phone)
  -- Note: NO salary, NO bank account, NO performance score

All views come from the SAME data. Each user sees only what they should.
```

**In SQL — External Schema = Views:**

```sql
-- HR Manager gets full employee view (authorized)
CREATE VIEW employee_hr_view AS
SELECT emp_id, name, department, hire_date, salary, performance_score
FROM employees
WHERE department = current_setting('app.user_department');

-- Junior developer gets only the directory (limited)
CREATE VIEW staff_directory AS
SELECT emp_id, name, department, email, phone
FROM employees;
-- salary, bank_account, performance_score NOT included

-- Applications query the VIEW, never the raw table directly
-- Change the underlying table → update the view → app sees nothing
```

**Why it matters:**
- **Security:** Users can't access data beyond their view
- **Simplicity:** Each user sees exactly what they need (no 50-column tables)
- **Flexibility:** Reshape data for each consumer without changing storage

### Layer 2 — Conceptual Schema (Logical Level) — "What data exists and how it relates"

The middle layer. The **single, unified description** of the ENTIRE database — all entities, all relationships, all business rules — independent of how it's stored or who views it.

```
Conceptual Schema answers:
  - WHAT entities exist? (Employee, Department, Project, Salary)
  - WHAT attributes do they have? (emp_id, name, hire_date)
  - HOW do they relate? (Employee BELONGS TO Department, Employee WORKS ON Project)
  - WHAT are the rules? (salary >= 0, emp_id is unique, department must exist)

Does NOT answer:
  - WHERE is it stored? (not physical file names)
  - HOW is it indexed? (not B-tree details)
  - WHAT application sees it? (not user views)
```

**In SQL — Conceptual Schema = Table Definitions + Constraints:**

```sql
-- The conceptual schema: what exists and what rules apply
CREATE TABLE employees (
  emp_id      BIGINT PRIMARY KEY,
  name        VARCHAR(200) NOT NULL,
  department  VARCHAR(100) NOT NULL,
  hire_date   DATE NOT NULL,
  salary      DECIMAL(12,2) NOT NULL CHECK (salary >= 0),
  email       VARCHAR(200) UNIQUE NOT NULL,
  manager_id  BIGINT REFERENCES employees(emp_id)  -- self-referencing relationship
);

CREATE TABLE departments (
  id   BIGINT PRIMARY KEY,
  name VARCHAR(100) NOT NULL UNIQUE,
  budget DECIMAL(15,2) NOT NULL CHECK (budget >= 0)
);

CREATE TABLE employee_departments (
  emp_id      BIGINT REFERENCES employees(emp_id),
  dept_id     BIGINT REFERENCES departments(id),
  start_date  DATE NOT NULL,
  PRIMARY KEY (emp_id, dept_id, start_date)
);

-- The constraints are the business rules: WHAT is valid data?
-- No physical storage decisions here
```

### Layer 3 — Internal Schema (Physical Level) — "How data is stored on disk"

The innermost layer. The **physical implementation details**: how data is stored, where indexes are, how files are organized. Completely hidden from users and applications.

```
Internal Schema answers:
  - HOW are rows stored? (heap files, clustered/unclustered)
  - WHERE are indexes? (B-tree on emp_id, hash on email)
  - HOW are files organized? (one file per tablespace, or multiple)
  - HOW is data compressed? (column-store compression or row-store)
  - WHAT buffer pool strategy? (LRU cache, 4KB pages)

Does NOT answer:
  - WHAT entities exist? (that's conceptual)
  - WHO can see what? (that's external)
```

**In SQL/PostgreSQL — Internal Schema = Physical Storage Settings:**

```sql
-- Index (internal: how to find rows quickly)
CREATE INDEX CONCURRENTLY idx_employees_dept ON employees(department);
CREATE INDEX idx_employees_name_search ON employees USING GIN(to_tsvector('english', name));

-- Tablespace (internal: which disk/partition to use)
CREATE TABLESPACE fast_disk LOCATION '/mnt/ssd/postgres';
ALTER TABLE employees SET TABLESPACE fast_disk;

-- Partitioning (internal: how rows are physically split)
CREATE TABLE employees_2024 PARTITION OF employees
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Storage parameters (internal: page fill factor)
ALTER TABLE employees SET (fillfactor = 90);  -- leave 10% free for updates

-- All invisible to the application querying staff_directory view
-- The app just sees: SELECT * FROM staff_directory WHERE name LIKE 'John%'
```

---

## Data Independence — The Key Benefit

```
Physical Data Independence:
  Change HOW data is stored (add index, move to SSD, change file format)
  → Conceptual and External layers UNCHANGED
  → Applications continue working without modification

Logical Data Independence:
  Change WHAT data is stored (add column, rename table, split table)
  → Internal layer needs updating, but can be done
  → External views can be updated to hide the change from apps
  → Applications may not need to change at all

Example:
  You split the employees table into employees + employee_details
  (performance optimization — less data loaded for common queries)

  Before: employees (emp_id, name, dept, salary, skills, projects, notes)
  After:  employees (emp_id, name, dept, salary)
          employee_details (emp_id, skills, projects, notes)

  Update the view:
  CREATE VIEW employee_hr_view AS
  SELECT e.*, ed.skills, ed.projects
  FROM employees e JOIN employee_details ed ON e.emp_id = ed.emp_id;

  HR Manager application: still queries employee_hr_view → unchanged ✓
  Physical change: table split → hidden ✓
  Logical change: schema changed → managed by view update ✓
```

---

## Model 2: Data Modeling Levels — How to Design a Database

This is what you use when **building** a database from business requirements.

### Step 1 — Conceptual Model (Business Level)

**Who:** Business analysts + domain experts + developers together
**Tool:** Entity-Relationship (ER) diagram — no technical terms
**Goal:** Capture WHAT exists in the business, not HOW to store it

```
Pantip Conceptual Model:
  ENTITIES:
    User        (a person who posts and reads)
    Post        (a forum discussion thread)
    Board       (a category/topic area)
    Comment     (a reply to a post)
    Tag         (a label attached to posts)

  RELATIONSHIPS:
    User CREATES Post          (one User → many Posts)
    Post BELONGS TO Board      (many Posts → one Board)
    User WRITES Comment        (one User → many Comments)
    Comment REPLIES TO Post    (many Comments → one Post)
    Post HAS Tag               (many-to-many: Post ↔ Tag)
    User UPVOTES Post          (many-to-many: User ↔ Post)

No SQL yet. No column names yet. No data types yet.
Just: what are the things? how do they relate?
```

**ER Diagram (text representation):**
```
[USER] ──creates──► [POST] ──belongsTo──► [BOARD]
  │                    │
  └──writes──► [COMMENT] ──repliesTo──► [POST]

[POST] ←──has──► [TAG] (many-to-many junction table needed)
[USER] ←──upvotes──► [POST] (many-to-many junction table needed)
```

### Step 2 — Logical Model (Technical Design, DB-Agnostic)

**Who:** Database designers / backend developers
**Tool:** Relational schema (table names, columns, data types, keys)
**Goal:** Translate business concepts into table structure — but NOT for any specific DB

```sql
-- Logical model: table structure, relationships, constraints
-- No storage details, no indexes, no partitions

users (
  id: INTEGER [PK]
  username: VARCHAR(50) [NOT NULL, UNIQUE]
  email: VARCHAR(200) [NOT NULL, UNIQUE]
  display_name: VARCHAR(100) [NOT NULL]
  created_at: TIMESTAMP [NOT NULL]
  status: ENUM('active','banned','deleted') [NOT NULL, DEFAULT 'active']
)

boards (
  id: INTEGER [PK]
  name: VARCHAR(100) [NOT NULL, UNIQUE]
  description: TEXT
  parent_id: INTEGER [FK → boards.id]  -- for sub-boards
)

posts (
  id: BIGINT [PK]
  title: VARCHAR(300) [NOT NULL]
  body: TEXT [NOT NULL]
  user_id: INTEGER [NOT NULL, FK → users.id]
  board_id: INTEGER [NOT NULL, FK → boards.id]
  created_at: TIMESTAMP [NOT NULL]
  view_count: INTEGER [NOT NULL, DEFAULT 0]
  status: ENUM('published','draft','deleted') [NOT NULL]
)

comments (
  id: BIGINT [PK]
  body: TEXT [NOT NULL]
  user_id: INTEGER [NOT NULL, FK → users.id]
  post_id: BIGINT [NOT NULL, FK → posts.id]
  parent_id: BIGINT [FK → comments.id]  -- for nested replies
  created_at: TIMESTAMP [NOT NULL]
)

post_tags (  -- junction table for many-to-many
  post_id: BIGINT [NOT NULL, FK → posts.id]
  tag_id: INTEGER [NOT NULL, FK → tags.id]
  PRIMARY KEY (post_id, tag_id)
)

post_upvotes (  -- junction table for many-to-many
  user_id: INTEGER [NOT NULL, FK → users.id]
  post_id: BIGINT [NOT NULL, FK → posts.id]
  voted_at: TIMESTAMP [NOT NULL]
  PRIMARY KEY (user_id, post_id)
)
```

### Step 3 — Physical Model (DB-Specific Implementation)

**Who:** Database administrators + senior developers
**Tool:** Actual SQL DDL for the specific database system
**Goal:** Optimize for the specific DBMS, access patterns, and scale

```sql
-- Physical model: actual PostgreSQL DDL with storage decisions

-- Users table: PostgreSQL-specific
CREATE TABLE users (
  id           BIGSERIAL PRIMARY KEY,  -- PostgreSQL: BIGSERIAL = BIGINT + sequence
  username     VARCHAR(50)  NOT NULL,
  email        VARCHAR(200) NOT NULL,
  display_name VARCHAR(100) NOT NULL,
  created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),  -- timezone-aware
  status       VARCHAR(10)  NOT NULL DEFAULT 'active'
    CHECK (status IN ('active', 'banned', 'deleted'))
) TABLESPACE pg_default;

-- Physical decisions: BIGSERIAL vs UUID, TIMESTAMPTZ vs TIMESTAMP, CHECK vs ENUM

-- Indexes (physical: not in logical model)
CREATE UNIQUE INDEX idx_users_username ON users(username);
CREATE UNIQUE INDEX idx_users_email ON users(LOWER(email));  -- case-insensitive unique
CREATE INDEX idx_users_status ON users(status) WHERE status = 'active';  -- partial index

-- Posts table: partitioned by board for Pantip's board-heavy traffic
CREATE TABLE posts (
  id          BIGSERIAL,
  title       VARCHAR(300) NOT NULL,
  body        TEXT NOT NULL,
  user_id     BIGINT NOT NULL REFERENCES users(id),
  board_id    INTEGER NOT NULL,  -- shard key
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  view_count  INTEGER NOT NULL DEFAULT 0,
  status      VARCHAR(10) NOT NULL DEFAULT 'published'
    CHECK (status IN ('published', 'draft', 'deleted')),
  PRIMARY KEY (id, board_id)  -- board_id in PK enables partition pruning
) PARTITION BY LIST (board_id);

-- Physical: create partitions per high-traffic board cluster
CREATE TABLE posts_cluster_a PARTITION OF posts
FOR VALUES IN (1, 5, 12, 15);  -- hot boards: General, IT, Entertainment

CREATE TABLE posts_cluster_b PARTITION OF posts
FOR VALUES IN (20, 25, 30, 35);  -- medium boards

-- Indexes on partitioned table
CREATE INDEX idx_posts_user ON posts(user_id);
CREATE INDEX idx_posts_board_created ON posts(board_id, created_at DESC);
-- Physical: DESC index for newest-first pagination

-- Full-text search (physical: Thai tokenizer)
CREATE INDEX idx_posts_search ON posts
USING GIN (to_tsvector('simple', title || ' ' || body));
-- Note: 'simple' tokenizer for Thai (English tokenizer won't work for Thai text)
```

---

## Summary: The Two 3-Layer Models Side by Side

```
ANSI/SPARC (HOW DBMS works):           Data Modeling (HOW you design):
─────────────────────────────           ──────────────────────────────
External Schema (Views)                 [not directly equivalent]
  ↕ logical independence
Conceptual Schema (Logical tables)  ←→  Logical Model (DB-agnostic tables)
  ↕ physical independence
Internal Schema (Storage, indexes)  ←→  Physical Model (DB-specific SQL)

                                        Conceptual Model (ER diagram)
                                          ↑ this extra step = purely business language
```

---

## In Clean Architecture Terms

```
ANSI/SPARC maps to Clean Architecture layers:

External Schema (Views)
  = Interface Adapters — each view is shaped for its consumer
  = Different DTOs for different controllers/consumers

Conceptual Schema (Logical tables)
  = Entities — the domain model, independent of storage details
  = IOrderRepository (interface) — defines what operations exist

Internal Schema (Indexes, storage)
  = Frameworks & Drivers — PostgreSQL, MySQL, physical implementation
  = OrderRepositoryPostgres — concrete adapter, storage decisions

Data Modeling Maps to Clean Architecture Design Process:
  Conceptual Model (business concepts)
    = Step 1: Define your domain entities (no DB yet)

  Logical Model (table structure)
    = Step 2: Design the schema (still DB-agnostic)

  Physical Model (actual DB SQL)
    = Step 3: Implement the Infrastructure layer adapter

The discipline: NEVER let physical decisions influence conceptual model.
The Order entity should not know about BIGSERIAL or TIMESTAMPTZ.
That's the physical model's concern.
```

---

## Real-World Application: Design a Pantip Post Feed

**Step 1 — Conceptual (Business language):**
```
"A user can write posts in a board.
A post has a title, a body, and belongs to one board.
Other users can comment on the post.
Users can upvote posts but only once per post."
```

**Step 2 — Logical (Table design, DB-agnostic):**
```
posts (id PK, title, body, user_id FK→users, board_id FK→boards, created_at, status)
comments (id PK, body, user_id FK→users, post_id FK→posts, parent_id FK→comments)
post_upvotes (user_id FK→users, post_id FK→posts, PK: both) -- junction, UNIQUE enforced
```

**Step 3 — Physical (PostgreSQL specific + Pantip's traffic pattern):**
```sql
-- Partition posts by board_id (Pantip's traffic is board-concentrated)
-- Index: board_id + created_at DESC (most common query: "latest posts in board")
-- Partial index: WHERE status = 'published' (drafts don't need indexing)
-- Thai full-text search: GIN index with 'simple' tokenizer
-- post_upvotes: composite PK enforces one vote per user per post (DB-level constraint)
```

---

## Key Takeaways

1. **ANSI/SPARC gives data independence** — change storage without breaking apps; change views without touching disk
2. **External Schema = security boundary** — users only see what they should see (SQL Views)
3. **Conceptual Schema = the truth** — one consistent picture of what data exists and what rules apply
4. **Internal Schema = performance decisions** — indexes, partitions, tablespaces, storage
5. **Data modeling process** (Conceptual→Logical→Physical) is how you BUILD a database correctly
6. **Never jump to physical first** — juniors write `CREATE TABLE` before they understand the domain; seniors write ER diagrams first
7. **Both maps to Clean Architecture** — Conceptual = Entities, Logical = Interfaces, Physical = Adapters/Infrastructure

---

**Sources:**
- [Three-Level DBMS Architecture — Medium](https://medium.com/@saisindujaa200603/three-level-dbms-architecture-external-conceptual-internal-a2efe89c7f3c)
- [ANSI-SPARC Architecture — Wikipedia](https://en.wikipedia.org/wiki/ANSI-SPARC_Architecture)
- [The 3-level DBMS schema architecture — TheServerSide](https://www.theserverside.com/tip/The-3-level-DBMS-schema-architecture)
- [Conceptual, Logical, Physical Data Models — DEV Community](https://dev.to/alexmercedcoder/conceptual-logical-and-physical-data-models-explained-h10)
- [Three-Level Architecture of DBMS — GeeksforGeeks](https://www.geeksforgeeks.org/dbms/introduction-of-3-tier-architecture-in-dbms-set-2/)
