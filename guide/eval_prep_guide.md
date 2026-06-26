# Evaluation Prep Guide
### Git · HTML/CSS/JS · DBMS · Docker

> **Scope:** Covers core concepts, interview-level depth, tricky edge cases, and what a technical panel at a Python-centric company actually expects from trainees.

---

## Table of Contents

1. [Git & Version Control](#1-git--version-control)
2. [HTML & CSS](#2-html--css)
3. [JavaScript & Its Relationship with HTML](#3-javascript--its-relationship-with-html)
4. [DBMS — SQL (Intermediate+)](#4-dbms--sql-intermediate)
5. [DBMS — NoSQL](#5-dbms--nosql)
6. [Docker](#6-docker)

---

---

# 1. Git & Version Control

## 1.1 Core Concepts

**What Git actually is:** Git is a distributed version control system. "Distributed" means every clone is a full copy of the repository — there's no single point of failure. Compare this to centralised VCS (like SVN) where the server is the source of truth.

**The three states of a file in Git:**

```
Working Directory  →  Staging Area (Index)  →  Repository (.git)
    (modified)           (git add)               (git commit)
```

- **Working directory:** Your actual files on disk. Untracked or modified.
- **Staging area (Index):** A snapshot of what *will* go into the next commit. You explicitly choose what to stage.
- **Repository:** The committed history. Stored in `.git/`.

**Why the staging area exists:** It lets you craft precise, logical commits. You might change 5 files but only want to commit changes to 2 of them — you stage only those two.

---

## 1.2 Branching & Merging

**HEAD:** A pointer to the current commit (or branch) you're working from. Detached HEAD means HEAD points directly to a commit, not a branch name.

```bash
# Creating and switching to a branch
git checkout -b feature/login   # old way
git switch -c feature/login     # modern way

# Merging
git checkout main
git merge feature/login         # fast-forward if possible

# Rebasing (alternative to merge)
git rebase main                 # replay feature commits on top of main
```

### Merge vs Rebase — a key interview distinction

| | Merge | Rebase |
|---|---|---|
| History | Preserves both branch histories, creates a merge commit | Rewrites history — commits are replayed as new commits |
| When to use | Integrating completed features, shared branches | Keeping a feature branch up to date with main locally |
| **Golden rule** | Safe on shared/public branches | **Never rebase commits that have been pushed to a shared branch** — it rewrites SHA hashes and causes conflicts for others |

**Fast-forward merge:** When the target branch hasn't diverged, Git just moves the pointer forward — no merge commit is created.

**Merge conflicts:** Occur when two branches change the same line. Git marks them with `<<<<<<<`, `=======`, `>>>>>>>` markers. You manually resolve and then `git add` + `git commit`.

---

## 1.3 Remote Operations

```bash
git remote add origin <url>     # link local repo to remote
git fetch origin                # download remote changes, don't merge
git pull origin main            # fetch + merge (git fetch + git merge)
git push origin feature/login   # push a branch

git pull --rebase origin main   # fetch + rebase instead of merge
```

**`fetch` vs `pull`:** `fetch` is safe — it downloads changes to a remote-tracking branch (`origin/main`) but does not touch your working directory. `pull` is `fetch` + `merge` (or rebase with `--rebase`). Good practice: fetch first, inspect, then merge.

---

## 1.4 Undoing Things — the tricky part

This is where most interviews probe depth.

```bash
# Amend the last commit (message or staged files)
git commit --amend              # WARNING: rewrites the commit SHA

# Unstage a file (keep changes in working dir)
git restore --staged <file>     # modern
git reset HEAD <file>           # old way

# Discard working directory changes (DESTRUCTIVE)
git restore <file>
git checkout -- <file>          # old way
```

### `git reset` — three modes

```bash
git reset --soft HEAD~1    # move HEAD back 1 commit, keep changes STAGED
git reset --mixed HEAD~1   # move HEAD back 1 commit, keep changes in WORKING DIR (default)
git reset --hard HEAD~1    # move HEAD back 1 commit, DISCARD all changes (destructive)
```

**`git revert` vs `git reset`:**
- `git revert <sha>` creates a *new* commit that undoes a previous one. History is preserved. Safe for shared branches.
- `git reset` rewrites history. Never use on commits pushed to shared branches.

**`git stash`:** Temporarily shelves changes without committing.

```bash
git stash                   # stash all tracked changes
git stash push -m "wip: login form"
git stash list
git stash pop               # apply most recent stash and remove it
git stash apply stash@{1}   # apply but keep in stash list
```

---

## 1.5 Useful Inspection Commands

```bash
git log --oneline --graph --all    # visual branch history
git diff                           # unstaged changes
git diff --staged                  # staged vs last commit
git blame <file>                   # who changed each line
git bisect start                   # binary search for bug-introducing commit
git cherry-pick <sha>              # apply a specific commit to current branch
```

---

## 1.6 `.gitignore`

- Patterns are matched relative to the `.gitignore` file location.
- `*.pyc` — ignore all `.pyc` files
- `/build` — ignore `build/` only at root
- `!important.log` — negate a rule (don't ignore this one)
- Once a file is already tracked, adding it to `.gitignore` won't stop tracking it. You need `git rm --cached <file>` first.

---

## 1.7 Tricky Scenarios

**Scenario: You committed to `main` directly instead of a feature branch.**
```bash
git branch feature/oops         # create branch at current position
git reset --hard HEAD~1         # move main back one commit
git switch feature/oops         # now you're on the feature branch with the work
```

**Scenario: You need to pull but have uncommitted changes.**
```bash
git stash
git pull origin main
git stash pop                   # may need to resolve conflicts
```

**Scenario: You pushed a commit with a secret/API key.**
The proper fix is:
1. Rotate the secret immediately (assume it's compromised).
2. Use `git filter-branch` or the BFG Repo Cleaner to remove it from history.
3. Force-push (destructive to collaborators).
The practical answer in most companies: rotate the key, remove it from the next commit, accept that history contains it (audit logs), and restrict access.

---

---

# 2. HTML & CSS

## 2.1 HTML Fundamentals

### Document Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Page Title</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <!-- Visible content -->
    <script src="app.js" defer></script>
</body>
</html>
```

- `<!DOCTYPE html>`: Tells the browser to use standards mode (not quirks mode).
- `<meta charset="UTF-8">`: Without this, special characters may render incorrectly.
- `<meta name="viewport" ...>`: Essential for responsive design. Without it, mobile browsers render at desktop width and scale down.

### Semantic HTML

Semantic elements describe the *meaning* of content, not just its appearance. This matters for accessibility (screen readers), SEO, and maintainability.

```html
<!-- Non-semantic (avoid for structure) -->
<div class="header">...</div>
<div class="nav">...</div>

<!-- Semantic (preferred) -->
<header>...</header>
<nav>...</nav>
<main>
    <article>...</article>
    <aside>...</aside>
</main>
<footer>...</footer>
```

Key semantic elements: `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<footer>`, `<figure>`, `<figcaption>`, `<time>`, `<mark>`.

### Block vs Inline Elements

| Block | Inline |
|---|---|
| Starts on a new line, takes full width | Flows within text, takes only as much width as content |
| `<div>`, `<p>`, `<h1>`-`<h6>`, `<ul>`, `<li>`, `<section>` | `<span>`, `<a>`, `<strong>`, `<em>`, `<img>` |

`<img>` is technically inline, but it's a *replaced element* — its content comes from an external source, so it behaves somewhat like block in certain contexts.

### Forms

```html
<form action="/submit" method="POST">
    <label for="email">Email</label>
    <input type="email" id="email" name="email" required placeholder="you@example.com">
    
    <label for="role">Role</label>
    <select id="role" name="role">
        <option value="dev">Developer</option>
        <option value="qa">QA</option>
    </select>
    
    <textarea name="message" rows="4"></textarea>
    
    <button type="submit">Submit</button>
</form>
```

**`for` and `id` must match** — this is what links the label to its input (for accessibility and click behaviour).

**`name` attribute** is what gets sent to the server. `id` is for CSS/JS targeting. They can be the same value, but they serve different purposes.

---

## 2.2 CSS Fundamentals

### The Box Model

Every element is a rectangular box:

```
+---------------------------+
|        margin             |
|  +---------------------+  |
|  |      border         |  |
|  |  +---------------+  |  |
|  |  |    padding    |  |  |
|  |  |  +---------+  |  |  |
|  |  |  | content |  |  |  |
|  |  |  +---------+  |  |  |
|  |  +---------------+  |  |
|  +---------------------+  |
+---------------------------+
```

**`box-sizing`:**
```css
/* Default — width applies to content only */
box-sizing: content-box;    /* element total width = width + padding + border */

/* Almost always what you want */
box-sizing: border-box;     /* width includes padding and border */

/* Common reset */
*, *::before, *::after { box-sizing: border-box; }
```

### Specificity

When multiple CSS rules match the same element, specificity determines which wins.

**Specificity hierarchy (highest → lowest):**
1. `!important` (avoid — breaks cascade)
2. Inline styles (`style="..."`)
3. ID selectors (`#header`) → **100 points**
4. Class, attribute, pseudo-class (`.btn`, `[type="text"]`, `:hover`) → **10 points**
5. Element, pseudo-element (`div`, `p`, `::before`) → **1 point**

```css
p { color: black; }              /* specificity: 0,0,1 */
.text { color: blue; }           /* specificity: 0,1,0 — wins */
#intro { color: red; }           /* specificity: 1,0,0 — wins over both */
p.text#intro { color: green; }   /* specificity: 1,1,1 — wins */
```

**Tricky:** Two rules with equal specificity — the one that appears *later* in the CSS wins (cascade order).

### Positioning

```css
position: static;    /* default — in normal flow */
position: relative;  /* offset from its own normal position; establishes positioning context */
position: absolute;  /* removed from flow; positioned relative to nearest non-static ancestor */
position: fixed;     /* like absolute, but relative to the viewport */
position: sticky;    /* normal flow until scroll threshold, then sticks */
```

**The "nearest non-static ancestor" trap:** An `absolute` child will look up the DOM tree for the first ancestor that has `position: relative/absolute/fixed/sticky`. If none exists, it's positioned relative to the `<html>` element (effectively the page). Forgetting to set `position: relative` on the parent is a very common bug.

### Flexbox

```css
.container {
    display: flex;
    flex-direction: row;         /* row | column | row-reverse | column-reverse */
    justify-content: center;     /* alignment along main axis */
    align-items: center;         /* alignment along cross axis */
    flex-wrap: wrap;             /* allow items to wrap */
    gap: 16px;                   /* space between items */
}

.item {
    flex: 1;                     /* shorthand for flex-grow: 1, flex-shrink: 1, flex-basis: 0 */
    flex: 0 0 200px;             /* fixed width, don't grow or shrink */
}
```

**`justify-content` vs `align-items`:** `justify-content` works along the *main axis* (by default, horizontal). `align-items` works along the *cross axis* (by default, vertical). These swap when `flex-direction: column`.

### CSS Grid

```css
.container {
    display: grid;
    grid-template-columns: repeat(3, 1fr);  /* 3 equal columns */
    grid-template-columns: 200px 1fr 1fr;   /* fixed first, flexible rest */
    gap: 16px;
}

.item {
    grid-column: 1 / 3;   /* span from column line 1 to 3 (takes up 2 columns) */
    grid-row: 1 / 2;
}
```

### Responsive Design

```css
/* Mobile-first approach */
.container { width: 100%; }

@media (min-width: 768px) {
    .container { max-width: 960px; margin: 0 auto; }
}

@media (min-width: 1024px) {
    .container { max-width: 1200px; }
}
```

### CSS Variables (Custom Properties)

```css
:root {
    --primary-color: #3b82f6;
    --spacing-md: 1rem;
}

.btn {
    background: var(--primary-color);
    padding: var(--spacing-md);
}
```

---

## 2.3 Tricky CSS Scenarios

**Collapsing margins:** The vertical margins of adjacent block elements collapse into a single margin (the larger of the two). This doesn't happen with flexbox/grid children, or with padding.

**`z-index` not working:** `z-index` only works on elements with a `position` other than `static`. Elements also form *stacking contexts* — a child with `z-index: 9999` can't stack above an element outside its parent's stacking context.

**`inline-block` vs `flex` for centering:** Using `text-align: center` on a parent centers inline and inline-block children. For block elements, `margin: 0 auto` works but requires a defined width. Flexbox `justify-content: center` is simpler and more predictable.

---

---

# 3. JavaScript & Its Relationship with HTML

## 3.1 How JS Connects to HTML

JavaScript can be included in HTML in three ways:

```html
<!-- 1. Inline (avoid for anything non-trivial) -->
<button onclick="alert('clicked')">Click</button>

<!-- 2. Internal script tag -->
<script>
    console.log("runs inline");
</script>

<!-- 3. External file (best practice) -->
<script src="app.js"></script>
<script src="app.js" defer></script>
<script src="app.js" async></script>
```

### `defer` vs `async`

| | `defer` | `async` |
|---|---|---|
| Download | Parallel with HTML parsing | Parallel with HTML parsing |
| Execution | After HTML fully parsed, in order | Immediately when downloaded, order not guaranteed |
| When to use | General scripts that need the DOM | Independent scripts (analytics, ads) |

**Rule of thumb:** Use `defer` for your own application scripts. Place `<script>` at the end of `<body>` if you don't use either attribute.

---

## 3.2 The DOM (Document Object Model)

The browser parses HTML and builds the DOM — a tree of objects where every HTML element becomes a node. JavaScript doesn't manipulate HTML text; it manipulates this object tree.

```javascript
// Selecting elements
const el = document.getElementById('myId');
const el = document.querySelector('.myClass');          // first match
const els = document.querySelectorAll('input[type="text"]');  // NodeList

// Reading and writing content
el.textContent = "new text";          // plain text (safe — no XSS risk)
el.innerHTML = "<strong>bold</strong>"; // parsed as HTML (XSS risk if user content)
el.value;                             // for form inputs

// Manipulating attributes
el.setAttribute('disabled', '');
el.removeAttribute('disabled');
el.getAttribute('href');
el.dataset.userId;                    // reads data-user-id attribute

// Manipulating classes
el.classList.add('active');
el.classList.remove('hidden');
el.classList.toggle('open');
el.classList.contains('active');      // returns boolean

// Manipulating styles
el.style.color = 'red';               // inline style (usually better to toggle classes)
```

---

## 3.3 Events

### Adding Event Listeners

```javascript
// Preferred — addEventListener (multiple listeners possible, separates concerns)
const btn = document.querySelector('#submitBtn');
btn.addEventListener('click', function(event) {
    event.preventDefault();   // prevent default browser behaviour (e.g., form submission)
    console.log('clicked');
});

// Arrow function version
btn.addEventListener('click', (e) => {
    console.log(e.target);    // the element that was clicked
});

// Removing a listener (requires named function reference)
function handleClick(e) { ... }
btn.addEventListener('click', handleClick);
btn.removeEventListener('click', handleClick);
```

### Event Bubbling and Delegation

Events bubble up the DOM tree by default. A click on a `<button>` inside a `<div>` also triggers click handlers on the `<div>`, and then on `<body>`, etc.

```javascript
// Event delegation — attach ONE listener to a parent, handle many children
document.querySelector('#todo-list').addEventListener('click', (e) => {
    if (e.target.matches('.delete-btn')) {
        e.target.closest('li').remove();
    }
});
```

**Why delegation matters:** If you have 1000 list items, adding 1000 individual listeners is expensive. One listener on the parent is efficient and also works for dynamically added elements.

`e.stopPropagation()` — prevents the event from bubbling further up.
`e.preventDefault()` — prevents the default browser action (form submit, link navigation, etc.).

---

## 3.4 Manipulating the DOM

```javascript
// Creating and appending elements
const li = document.createElement('li');
li.textContent = 'New item';
li.classList.add('todo-item');
document.querySelector('#list').appendChild(li);

// Inserting relative to another element
parent.insertBefore(newEl, referenceEl);
referenceEl.insertAdjacentElement('beforebegin', newEl);  // before the element itself
referenceEl.insertAdjacentElement('afterend', newEl);     // after the element itself

// Removing elements
el.remove();
parent.removeChild(el);
```

---

## 3.5 Fetch API — JS Communicating with the Server

This is how JavaScript communicates with a backend (like a Django or FastAPI API) without reloading the page.

```javascript
// GET request
fetch('/api/users')
    .then(response => {
        if (!response.ok) {
            throw new Error(`HTTP error: ${response.status}`);
        }
        return response.json();   // parse JSON body
    })
    .then(data => {
        console.log(data);
    })
    .catch(err => console.error(err));

// POST request with async/await
async function createUser(userData) {
    try {
        const response = await fetch('/api/users', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': 'Bearer ' + token
            },
            body: JSON.stringify(userData)
        });

        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.detail);
        }

        return await response.json();
    } catch (err) {
        console.error('Failed to create user:', err);
        throw err;
    }
}
```

**CORS:** When your frontend (running on `localhost:3000`) makes a request to a backend on a different origin (`localhost:8000`), the browser enforces the Same-Origin Policy. The server must send `Access-Control-Allow-Origin` headers. This is a backend configuration — FastAPI and Django DRF both have CORS middleware for this.

---

## 3.6 Core JS Concepts a Panel Will Probe

### `var` vs `let` vs `const`

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function-scoped | Block-scoped | Block-scoped |
| Hoisting | Hoisted and initialised as `undefined` | Hoisted but NOT initialised (Temporal Dead Zone) | Same as `let` |
| Reassignable | Yes | Yes | No (but object contents can mutate) |

**Temporal Dead Zone:** Accessing a `let` or `const` variable before its declaration throws a `ReferenceError`, even though the declaration is hoisted.

### Closures

A closure is a function that "remembers" the variables from its enclosing scope even after that scope has finished executing.

```javascript
function makeCounter() {
    let count = 0;          // this variable is "closed over"
    return function() {
        count++;
        return count;
    };
}

const counter = makeCounter();
counter();  // 1
counter();  // 2
```

**Classic closure trap in loops:**
```javascript
// BUG: every timeout logs 3
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100);
}

// FIX 1: use let (block-scoped, each iteration gets its own i)
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100);
}

// FIX 2: IIFE to capture the value
for (var i = 0; i < 3; i++) {
    ((j) => setTimeout(() => console.log(j), 100))(i);
}
```

### `this` Keyword

`this` refers to the object that is executing the current function. Its value depends on *how* the function is called.

```javascript
const obj = {
    name: 'Alice',
    greet() {
        console.log(this.name);   // 'Alice' — called as method
    },
    greetArrow: () => {
        console.log(this.name);   // undefined — arrow functions inherit this from enclosing scope
    }
};

// Explicitly setting this
obj.greet.call({ name: 'Bob' });   // 'Bob'
obj.greet.apply({ name: 'Bob' }); // same as call, args passed as array
const boundGreet = obj.greet.bind({ name: 'Charlie' });
boundGreet();   // 'Charlie'
```

### Promises and Async/Await

```javascript
// Promise chain
fetch('/api/data')
    .then(r => r.json())
    .then(data => processData(data))
    .catch(err => handleError(err))
    .finally(() => hideSpinner());

// Async/await (syntactic sugar over promises)
async function loadData() {
    try {
        const response = await fetch('/api/data');
        const data = await response.json();
        return processData(data);
    } catch (err) {
        handleError(err);
    } finally {
        hideSpinner();
    }
}

// Running multiple requests in parallel
const [users, posts] = await Promise.all([
    fetch('/api/users').then(r => r.json()),
    fetch('/api/posts').then(r => r.json())
]);
```

### ES6+ Features You Should Know

```javascript
// Destructuring
const { name, age } = user;
const [first, ...rest] = array;

// Spread operator
const newArr = [...arr1, ...arr2];
const newObj = { ...obj1, extra: 'value' };

// Template literals
const msg = `Hello, ${user.name}! You have ${count} notifications.`;

// Optional chaining (prevents TypeError on null/undefined)
const city = user?.address?.city;     // undefined if any part is null/undefined

// Nullish coalescing (only falls back for null/undefined, not falsy values like 0 or "")
const name = user.name ?? 'Anonymous';

// Array methods
const doubled = arr.map(x => x * 2);
const evens = arr.filter(x => x % 2 === 0);
const sum = arr.reduce((acc, x) => acc + x, 0);
const found = arr.find(x => x.id === 5);
const exists = arr.some(x => x.active);
const allActive = arr.every(x => x.active);
```

---

---

# 4. DBMS — SQL (Intermediate+)

## 4.1 Relational Model Refresher

A **relation** (table) is a set of tuples (rows). Each column has a defined domain (data type). Key concepts:

- **Primary Key:** Uniquely identifies each row. Cannot be NULL.
- **Foreign Key:** References the primary key of another table. Enforces referential integrity.
- **Candidate Key:** Any column (or set of columns) that could serve as a primary key.
- **Surrogate vs Natural Key:** Surrogate (auto-increment `id`) is artificial; Natural key is a real-world attribute (e.g., email). Surrogates are generally safer because natural keys can change.

---

## 4.2 Normalisation

The process of organising data to reduce redundancy and improve integrity.

**1NF:** Atomic values — no repeating groups or arrays in a cell. Each cell holds one value.

**2NF:** 1NF + no partial dependencies. Every non-key attribute must depend on the *entire* primary key (only relevant when the PK is composite).

**3NF:** 2NF + no transitive dependencies. Non-key attributes must not depend on other non-key attributes.

```
Violation example (3NF): 
Table: Order(order_id, customer_id, customer_city)
customer_city depends on customer_id, not on order_id.
Fix: Move customer_city to a Customer table.
```

**BCNF (Boyce-Codd Normal Form):** Stricter version of 3NF. Every determinant must be a candidate key.

**In practice:** Most production schemas aim for 3NF. Denormalisation is sometimes intentional for read performance.

---

## 4.3 SQL — Beyond the Basics

### JOINs

```sql
-- INNER JOIN: only matching rows from both tables
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: all rows from left table, matching from right (NULL if no match)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- SELF JOIN: join a table with itself
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

**Tricky join behaviour:** `NULL = NULL` is `FALSE` in SQL. If a foreign key is NULL, it will not match any row in an INNER JOIN.

### Aggregate Functions and GROUP BY

```sql
SELECT department, AVG(salary), MAX(salary), COUNT(*)
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000;   -- HAVING filters AFTER grouping; WHERE filters BEFORE
```

**`WHERE` vs `HAVING`:** `WHERE` cannot reference aggregate functions. `HAVING` can. If you need to filter on a non-aggregate column, prefer `WHERE` — it filters before the grouping, which is more efficient.

### Subqueries and CTEs

```sql
-- Subquery in WHERE
SELECT name FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated subquery (references outer query — runs once per row, can be slow)
SELECT name FROM employees e1
WHERE salary > (
    SELECT AVG(salary) FROM employees e2
    WHERE e2.department = e1.department   -- references outer query
);

-- CTE (Common Table Expression) — more readable, often same performance as subquery
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary, d.avg_salary
FROM employees e
JOIN dept_avg d ON e.department = d.department
WHERE e.salary > d.avg_salary;
```

### Window Functions

Window functions perform calculations across rows *related to the current row* without collapsing the result set like GROUP BY does.

```sql
SELECT
    name,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_in_dept,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS overall_rank,
    LAG(salary, 1) OVER (ORDER BY hire_date) AS prev_salary
FROM employees;
```

**`RANK()` vs `ROW_NUMBER()` vs `DENSE_RANK()`:**
- `ROW_NUMBER()`: Unique consecutive integers. Ties get different numbers.
- `RANK()`: Ties get the same rank, next rank skips (1, 1, 3).
- `DENSE_RANK()`: Ties get the same rank, no skipping (1, 1, 2).

### Indexes

An index is a data structure (usually a B-tree) that speeds up reads at the cost of write performance and storage.

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX idx_composite ON orders(user_id, created_at);  -- composite index
```

**When indexes are used:**
- Equality lookups: `WHERE user_id = 5`
- Range queries: `WHERE created_at > '2024-01-01'`
- ORDER BY and GROUP BY on indexed columns
- JOINs on foreign keys

**When indexes are NOT used (or misused):**
- Functions applied to the column: `WHERE YEAR(created_at) = 2024` — the index on `created_at` won't be used. Rewrite as range query.
- Low cardinality columns: An index on a boolean column is often less efficient than a full table scan.
- Leading column rule: A composite index `(a, b, c)` supports queries on `a`, `(a, b)`, or `(a, b, c)` — but NOT on `b` alone or `c` alone.

### Transactions and ACID

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If anything fails, ROLLBACK; restores both updates
```

**ACID:**
- **Atomicity:** All or nothing. If one statement fails, the whole transaction rolls back.
- **Consistency:** The database goes from one valid state to another. Constraints are enforced.
- **Isolation:** Concurrent transactions don't see each other's intermediate state.
- **Durability:** Committed transactions survive crashes (written to disk/WAL).

### Isolation Levels and Concurrency Anomalies

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible |
| SERIALIZABLE | Prevented | Prevented | Prevented |

- **Dirty Read:** Reading a row modified by another uncommitted transaction.
- **Non-Repeatable Read:** Reading the same row twice in a transaction and getting different values (another transaction committed a change in between).
- **Phantom Read:** Running the same query twice and getting a different set of rows (another transaction inserted/deleted rows in between).

---

## 4.4 Query Optimisation

**`EXPLAIN` / `EXPLAIN ANALYZE`:** Shows the query execution plan. Use this to understand if indexes are being used.

**Common performance patterns:**
- Select only the columns you need (`SELECT *` is fine for exploration, not for production queries on large tables).
- Filter early — reduce the dataset as early as possible in the query.
- Avoid `SELECT DISTINCT` as a shortcut for bad joins; fix the root cause instead.
- `EXISTS` is often faster than `IN` for subqueries because it short-circuits on first match.

```sql
-- IN with subquery — evaluates all rows
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders);

-- EXISTS — stops at first match
SELECT * FROM users u WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

---

## 4.5 Tricky SQL Scenarios

**NULL handling:** `NULL` is not a value — it represents the *absence* of a value. Any comparison with `NULL` using `=` returns `NULL` (falsy). Use `IS NULL` and `IS NOT NULL`.

```sql
-- This returns NO rows even if NULL values exist
SELECT * FROM users WHERE middle_name = NULL;   -- WRONG

-- Correct
SELECT * FROM users WHERE middle_name IS NULL;
```

**`COALESCE` and `NULLIF`:**
```sql
COALESCE(a, b, c)   -- returns first non-NULL value
NULLIF(a, b)        -- returns NULL if a = b, else returns a
```

---

---

# 5. DBMS — NoSQL

## 5.1 Why NoSQL Exists

Relational databases enforce strict schemas, require normalisation, and scale primarily vertically. NoSQL databases emerged to address:
- **Schema flexibility:** Data shape varies between records (e.g., different product attributes in an e-commerce store).
- **Horizontal scalability:** Distribute data across many machines.
- **High write throughput:** Sacrifice some consistency for speed.
- **Specific access patterns:** Key-value lookups, graph traversals, time-series data.

---

## 5.2 CAP Theorem

A distributed database can guarantee at most **two** of:
- **Consistency (C):** Every read returns the most recent write.
- **Availability (A):** Every request gets a response (not necessarily the latest data).
- **Partition Tolerance (P):** The system continues operating when network partitions occur.

Since network partitions are a reality, systems choose CP or AP:
- **CP (e.g., MongoDB with strong consistency settings, HBase):** Sacrifice availability during partitions.
- **AP (e.g., Cassandra, CouchDB):** Return data that might be stale during partitions.

**BASE vs ACID:**
- ACID is what SQL databases guarantee.
- BASE (Basically Available, Soft state, Eventually consistent) is what many NoSQL systems offer.

---

## 5.3 Document Databases (MongoDB)

Data is stored as JSON-like documents (BSON in MongoDB). Documents in a collection don't need the same structure.

```javascript
// A document
{
    "_id": ObjectId("..."),
    "name": "Alice",
    "email": "alice@example.com",
    "address": {              // embedded document
        "city": "Mumbai",
        "pin": "400001"
    },
    "orders": [              // embedded array
        { "product": "Book", "qty": 2 },
        { "product": "Pen", "qty": 10 }
    ]
}
```

### Querying

```javascript
// Find documents
db.users.find({ "address.city": "Mumbai" })
db.users.find({ age: { $gt: 25, $lte: 40 } })
db.users.find({ tags: { $in: ["python", "backend"] } })

// Update
db.users.updateOne(
    { _id: ObjectId("...") },
    { $set: { "address.city": "Pune" }, $push: { tags: "django" } }
)

// Aggregation pipeline
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $group: { _id: "$user_id", total: { $sum: "$amount" } } },
    { $sort: { total: -1 } },
    { $limit: 10 }
])
```

### Embedding vs Referencing

| Embedding (subdocuments) | Referencing (foreign keys) |
|---|---|
| Denormalised — data lives together | Normalised — requires a second query or `$lookup` |
| Fast reads for related data | Avoids data duplication |
| Good when the embedded data is always accessed with the parent | Good when the referenced data is large, shared, or independently queried |
| MongoDB document size limit: **16 MB** | More flexible for many-to-many relationships |

### Indexes in MongoDB

```javascript
db.users.createIndex({ email: 1 })         // ascending single-field index
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ city: 1, age: -1 }) // compound index
db.products.createIndex({ name: "text" })  // full-text search index
```

---

## 5.4 Key-Value Stores (Redis)

Redis stores data as key-value pairs in memory, making it extremely fast. It's commonly used for:
- **Caching:** Store expensive query results with a TTL (time to live).
- **Session storage:** Fast per-request session lookup.
- **Rate limiting:** Increment a counter per IP per minute.
- **Pub/Sub messaging:** Lightweight message broker.
- **Queues:** Redis lists as job queues (used by Celery).

```python
import redis
r = redis.Redis(host='localhost', port=6379, db=0)

r.set('user:1:name', 'Alice')
r.get('user:1:name')               # b'Alice'
r.setex('session:abc', 3600, 'user_id=1')  # expires in 1 hour

r.incr('page:home:views')
r.lpush('job_queue', 'send_email')
r.rpop('job_queue')
```

**Redis is not a primary database** in most architectures — it's a cache or ephemeral store. Data persistence is optional (RDB snapshots, AOF logs).

---

## 5.5 SQL vs NoSQL — What a Panel Wants to Hear

Don't say "NoSQL is better for big data" — that's a non-answer. The right answer is context-dependent:

**Choose SQL when:**
- You have complex relationships and need joins.
- You need strong ACID guarantees (financial transactions, inventory).
- The schema is well-defined and stable.
- You need ad-hoc querying (analytics).

**Choose NoSQL when:**
- The data structure varies significantly per record.
- You're storing hierarchical or document-style data where embedding is natural.
- You need to scale writes horizontally across many nodes.
- You're building a cache, session store, or message queue.

**Often both are used together:** SQL as the primary store for relational data, Redis as a cache layer, MongoDB for a product catalog or content repository.

---

---

# 6. Docker

## 6.1 The Core Problem Docker Solves

"It works on my machine" — Docker solves this by packaging an application *and its environment* together. This environment includes the OS libraries, runtime, dependencies, and configuration.

**Virtual Machine vs Container:**
| | Virtual Machine | Container |
|---|---|---|
| Isolation | Full OS virtualisation | Process-level isolation via Linux namespaces & cgroups |
| Size | GBs (full OS image) | MBs (shares host OS kernel) |
| Startup time | Minutes | Seconds or less |
| Performance overhead | Higher | Near-native |

---

## 6.2 Core Concepts

- **Image:** A read-only template. A blueprint for a container. Built from a `Dockerfile`. Images are layered.
- **Container:** A running instance of an image. Isolated process with its own filesystem, network, and process space.
- **Registry:** A storage and distribution system for images. Docker Hub is the default public registry. You can have private registries (Azure Container Registry, AWS ECR).
- **Dockerfile:** A text file with instructions that Docker executes layer by layer to build an image.

---

## 6.3 Essential Docker Commands

```bash
# Image operations
docker pull python:3.11-slim          # download an image
docker images                         # list local images
docker rmi python:3.11-slim           # remove an image
docker build -t myapp:1.0 .           # build from Dockerfile in current dir

# Container operations
docker run python:3.11-slim           # create and start a container
docker run -it python:3.11-slim bash  # interactive terminal
docker run -d -p 8000:8000 myapp:1.0  # detached, port mapping
docker run --name my-container myapp  # give it a name
docker run --rm myapp:1.0             # remove container when it stops

# Flags you'll use constantly
# -d         detached mode (background)
# -p 8000:8000  host_port:container_port
# -e DB_URL=...  set environment variable
# -v /host/path:/container/path  mount a volume
# --name     name the container
# --rm       auto-remove when stopped

# Managing running containers
docker ps                             # list running containers
docker ps -a                          # all containers (including stopped)
docker stop my-container
docker start my-container
docker restart my-container
docker rm my-container                # remove stopped container
docker exec -it my-container bash     # open shell in running container
docker logs my-container              # view output
docker logs -f my-container           # follow logs in real time
```

---

## 6.4 Writing a Dockerfile

```dockerfile
# Start from an official base image
FROM python:3.11-slim

# Set working directory inside the container
WORKDIR /app

# Copy dependency file first (for caching — see below)
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code
COPY . .

# Environment variable
ENV PORT=8000

# Expose the port the app listens on (documentation only — doesn't publish the port)
EXPOSE 8000

# The command to run when the container starts
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Layer Caching — Critical Concept

Docker builds images layer by layer. Each instruction (`FROM`, `COPY`, `RUN`, etc.) creates a layer. If a layer hasn't changed, Docker uses the cached version.

**The rule:** Put things that change least at the top, things that change most at the bottom.

```dockerfile
# BAD: Copying all code before installing dependencies
COPY . .
RUN pip install -r requirements.txt   # re-runs every time ANY file changes

# GOOD: Copy requirements first, install, then copy code
COPY requirements.txt .
RUN pip install -r requirements.txt   # cached until requirements.txt changes
COPY . .                              # code changes don't invalidate the pip layer
```

### `CMD` vs `ENTRYPOINT`

```dockerfile
# CMD: default command, easily overridden at runtime
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

# Run with override: docker run myapp python manage.py migrate
# The CMD is replaced entirely.

# ENTRYPOINT: the main executable, not easily overridden
ENTRYPOINT ["uvicorn", "main:app"]
CMD ["--host", "0.0.0.0", "--port", "8000"]   # CMD becomes default args to ENTRYPOINT

# Common pattern: ENTRYPOINT sets the program, CMD sets default args
```

---

## 6.5 Docker Volumes

Containers are ephemeral — when a container is removed, its filesystem is gone. Volumes persist data.

```bash
# Named volume (Docker manages the location)
docker run -v mydata:/app/data myapp

# Bind mount (maps a host directory into the container — useful for development)
docker run -v $(pwd)/data:/app/data myapp

# Read-only bind mount
docker run -v $(pwd)/config:/app/config:ro myapp
```

**Development pattern with bind mounts:** Mount your source code directory into the container so changes are immediately reflected without rebuilding the image.

```bash
docker run -d -p 8000:8000 -v $(pwd):/app myapp
```

---

## 6.6 Docker Networking

Containers run in isolated networks. By default, a container can't be reached from outside unless you publish ports with `-p`.

```bash
docker network create mynetwork
docker run --network mynetwork --name db postgres
docker run --network mynetwork --name app myapp
# 'app' container can now reach 'db' by hostname 'db'
```

**Port mapping:** `-p 8000:8000` means "forward host port 8000 to container port 8000". The container internally listens on its own port; Docker NATs external requests.

**Important:** Your app must listen on `0.0.0.0` (all interfaces), not `127.0.0.1` (localhost only). Listening on localhost inside a container means it's unreachable from outside that container.

```python
# FastAPI — correct for Docker
uvicorn.run(app, host="0.0.0.0", port=8000)

# This would be unreachable from the host
uvicorn.run(app, host="127.0.0.1", port=8000)
```

---

## 6.7 Docker Compose

Docker Compose defines and runs multi-container applications from a single YAML file. It's the standard way to run local development environments.

```yaml
# docker-compose.yml
version: "3.9"

services:
  web:
    build: .                          # build from Dockerfile in current dir
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
    volumes:
      - .:/app                        # bind mount for development
    depends_on:
      - db
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

```bash
docker compose up              # start all services
docker compose up -d           # detached mode
docker compose up --build      # rebuild images before starting
docker compose down            # stop and remove containers
docker compose down -v         # also remove volumes
docker compose logs web        # logs for a specific service
docker compose exec web bash   # shell into a running service
```

**`depends_on`:** Controls start order, but does NOT wait for the service to be *ready* (e.g., Postgres takes a few seconds to accept connections after starting). For production-grade setups, use a health check or a wait script.

---

## 6.8 `.dockerignore`

Like `.gitignore` but for Docker's build context. Prevents unnecessary files from being sent to the Docker daemon.

```
.git
__pycache__
*.pyc
*.pyo
.env
.venv
venv/
*.log
node_modules/
```

Without `.dockerignore`, Docker sends your entire directory (including `.git` and `node_modules`) to the build context, slowing builds significantly.

---

## 6.9 Tricky Docker Scenarios

**Scenario: Container starts but the app can't connect to the database.**

Common causes:
1. The DB container is still initialising — `depends_on` doesn't wait for readiness.
2. The app is connecting to `localhost` instead of the service name (`db`).
3. The port mapping is correct externally, but the app should use the container port internally.

**Scenario: Image is very large.**
Use `python:3.11-slim` or `python:3.11-alpine` instead of `python:3.11`. Use multi-stage builds to avoid copying build-time dependencies into the final image. Use `.dockerignore`.

**Scenario: Code changes aren't reflected.**
If using bind mounts in development, the code updates immediately. But if you forgot the bind mount and just rebuilt the image — you need to also recreate the container (`docker compose up --build`), not just restart it.

**Scenario: Environment variables with secrets.**
Never hardcode secrets in the `Dockerfile` or `docker-compose.yml`. Use `.env` files (and add `.env` to `.gitignore`) or Docker secrets for production.

```bash
# .env file
DATABASE_URL=postgresql://...
SECRET_KEY=...

# docker-compose.yml
env_file:
  - .env
```

---

*End of Guide — Good luck with the evaluation!*
