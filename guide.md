# KYC Submission System — Intern Build Plan (Python backend)
 
A guided, phase-by-phase plan to build a Know-Your-Customer (KYC) submission system using **HTML, CSS, and vanilla JavaScript on the browser**, and **Python (Flask) + SQLite on the server**.
 
You'll build two things:
 
1. A **public form** where a user enters their name, email, phone, and uploads an ID document (ID card or passport).
2. An **admin dashboard** that lists every submission and lets a reviewer **approve** or **reject** it.
Work through the phases in order. Each phase has a clear objective, precise steps, the tricky code written out for you, and a checkpoint to confirm before moving on. Don't skip the "Verify" steps — they're how you catch bugs early.
 
> **Note on the JavaScript:** the *server* is all Python. The only JavaScript is a small amount in the browser — to validate the form and to make the admin dashboard interactive. Your mentor will walk you through that part; you don't need to know JS deeply to follow it. Everything that stores data, handles files, and makes decisions is Python.
 
---
 
## What you'll learn
 
By the end of this you'll understand:
 
- How a browser form sends data (including a file) to a backend server.
- How a Python (Flask) server receives requests, talks to a database, and sends responses.
- How file uploads are received, validated, and stored safely.
- How the admin page fetches data from your Python API and updates it.
- Basic input validation and why it matters more in fintech than almost anywhere else.
Treat every "why" note seriously — copying code that works is easy; understanding *why* it works is the actual job.
 
---
 
## Tech stack and why
 
| Layer | Tool | Why |
|---|---|---|
| Frontend | HTML + CSS + a little vanilla JS | No frameworks — you see the fundamentals directly. The JS is minimal and explained to you. |
| Server | Python + **Flask** | Flask is the simplest Python web framework: a route is just a Python function. Very little magic to get in your way. |
| File uploads | Flask's built-in `request.files` | No extra library needed — Flask handles uploaded files for you. |
| Database | **SQLite** via Python's built-in `sqlite3` | A real SQL database in a single file, and `sqlite3` ships with Python — nothing to install. |
 
The **only** external dependency is Flask. Everything else (`sqlite3`, `os`, `secrets`, `functools`) is part of Python's standard library. Resist the urge to add more.
 
---
 
## Prerequisites (do this before Phase 0)
 
1. Install **Python 3.10 or later**. Verify in a terminal:
```bash
   python --version
   pip --version
```
   (On some systems the commands are `python3` and `pip3` — use whichever works.)
2. Install a code editor (VS Code is fine).
3. Make sure you can open a terminal in your project folder.
 
If any of these fail, stop and sort it out first — nothing else will work until they do.
 
---
 
## Final project structure
 
This is what you're building toward. Don't create it all at once; each phase adds the relevant files.
 
```
kyc-app/
├── requirements.txt     # lists Flask
├── app.py               # the Flask server (all routes live here)
├── db.py                # database connection + schema setup
├── kyc.db               # the SQLite database file (auto-created)
├── uploads/             # uploaded ID documents (NOT publicly served)
└── public/              # everything the browser can see
    ├── index.html       # the user-facing KYC form
    ├── form.js          # browser JS for the form
    ├── admin.html       # the admin dashboard
    ├── admin.js         # browser JS for the dashboard
    └── styles.css       # shared styling
```
 
**Why `uploads/` is separate from `public/`:** ID documents are sensitive. If they sat inside `public/`, anyone who guessed a filename could download someone's passport. Instead, we serve documents only through a controlled Python route that we can later lock behind a password. Remember this — it's a real-world habit, not a detail.
 
---
 
## Phase 0 — Project setup
 
**Objective:** an empty but runnable Flask project.
 
**Steps:**
 
1. Create and enter the project folder:
```bash
   mkdir kyc-app && cd kyc-app
```
2. (Recommended) Create and activate a virtual environment so this project's packages stay isolated:
```bash
   python -m venv venv
   # Windows:
   venv\Scripts\activate
   # macOS/Linux:
   source venv/bin/activate
```
3. Install Flask and record it:
```bash
   pip install Flask
   pip freeze > requirements.txt
```
4. Create the folders that won't be auto-created:
```bash
   mkdir uploads public
```
5. Create `app.py` with a minimal server to confirm it runs:
```python
   from flask import Flask
 
   app = Flask(__name__)
 
   @app.route('/')
   def home():
       return 'KYC server is alive'
 
   if __name__ == '__main__':
       app.run(port=3000, debug=True)
```
6. Run it:
```bash
   python app.py
```
 
**Verify:** Open `http://localhost:3000` in your browser. You should see "KYC server is alive". Stop the server with `Ctrl+C`.
 
**Checkpoint** Don't continue until the server runs and the page loads.
 
---
 
## Phase 1 — Database schema
 
**Objective:** a SQLite database with one table to hold submissions.
 
**Steps:**
 
1. Create `db.py`:
```python
   import sqlite3
 
   DB_FILE = 'kyc.db'
 
   def get_db():
       """Open a new connection. Rows behave like dictionaries."""
       conn = sqlite3.connect(DB_FILE)
       conn.row_factory = sqlite3.Row
       return conn
 
   def init_db():
       """Create the submissions table if it doesn't already exist."""
       conn = get_db()
       conn.execute("""
           CREATE TABLE IF NOT EXISTS submissions (
               id                     INTEGER PRIMARY KEY AUTOINCREMENT,
               full_name              TEXT NOT NULL,
               email                  TEXT NOT NULL,
               phone                  TEXT NOT NULL,
               document_filename      TEXT NOT NULL,   -- the safe name we store on disk
               document_original_name TEXT,            -- what the user called the file
               document_mime          TEXT,            -- e.g. image/png, application/pdf
               status                 TEXT NOT NULL DEFAULT 'pending',  -- pending | approved | rejected
               created_at             TEXT NOT NULL DEFAULT (datetime('now')),
               reviewed_at            TEXT
           )
       """)
       conn.commit()
       conn.close()
```
 
**Why these columns:** `status` drives the whole admin workflow (it starts as `pending`). We store both the original filename (for display) and our own generated filename (what's actually on disk) — you'll see why in Phase 4. Timestamps let the admin sort and audit submissions.
 
**Why we open a fresh connection each time (`get_db`):** SQLite connections don't like being shared across different requests/threads. Opening one per request and closing it is the simplest pattern that's always correct.
 
**Verify:** In `app.py`, import and call `init_db()` once at startup:
```python
from db import init_db
init_db()
```
Run `python app.py`, then confirm the file `kyc.db` now exists in your folder. (Optional: install the "SQLite Viewer" VS Code extension to inspect it.)
 
**Checkpoint** The database file is created and the server still runs.
 
---
 
## Phase 2 — The submission API (text fields only)
 
**Objective:** a Python route that accepts a submission and saves it. We'll add the file in Phase 4 — first get the text fields working.
 
**Steps:**
 
1. Build out `app.py`:
```python
   from flask import Flask, request, jsonify, send_from_directory
   from db import init_db, get_db
 
   app = Flask(__name__, static_folder='public', static_url_path='')
   init_db()
 
   # Serve the user form at the root URL
   @app.route('/')
   def index():
       return send_from_directory('public', 'index.html')
 
   # Create a submission (text fields only for now)
   @app.route('/api/submissions', methods=['POST'])
   def create_submission():
       full_name = request.form.get('full_name')
       email = request.form.get('email')
       phone = request.form.get('phone')
 
       if not full_name or not email or not phone:
           return jsonify(error='All fields are required.'), 400
 
       conn = get_db()
       cur = conn.execute(
           """INSERT INTO submissions (full_name, email, phone, document_filename)
              VALUES (?, ?, ?, ?)""",
           (full_name, email, phone, 'placeholder')
       )
       conn.commit()
       new_id = cur.lastrowid
       conn.close()
       return jsonify(id=new_id), 201
 
   if __name__ == '__main__':
       app.run(port=3000, debug=True)
```
 
**Why the `?` placeholders (and never f-strings) in SQL:** never paste user input directly into a SQL string. The `?` placeholders are *parameterised queries* — they stop **SQL injection**, a classic attack where a malicious value rewrites your query. This is non-negotiable.
 
**Why `static_folder='public', static_url_path=''`:** this tells Flask to serve your `public/` files directly, so `styles.css`, `form.js`, and `admin.html` are reachable in the browser without writing a route for each one.
 
**Verify:** With the server running, test the route from a second terminal:
```bash
curl -X POST http://localhost:3000/api/submissions \
  -F "full_name=Test User" -F "email=t@test.com" -F "phone=650000000"
```
You should get back something like `{"id":1}`. Run it once more and confirm the id increments.
 
**Checkpoint** A row is being inserted and you get an id back.
 
---
 
## Phase 3 — The user-facing form
 
**Objective:** an HTML form that submits to your Python API.
 
**Steps:**
 
1. Create `public/styles.css` with simple, clean styling. Keep it minimal — a centred card, readable inputs, a clear button. (This is yours to design; aim for tidy, not fancy.)
2. Create `public/index.html`:
```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
     <meta charset="UTF-8" />
     <meta name="viewport" content="width=device-width, initial-scale=1.0" />
     <title>KYC Verification</title>
     <link rel="stylesheet" href="styles.css" />
   </head>
   <body>
     <main class="card">
       <h1>Identity Verification</h1>
       <p>Please complete the form and upload a valid ID card or passport.</p>
 
       <form id="kyc-form">
         <label>Full name
           <input type="text" name="full_name" required />
         </label>
 
         <label>Email
           <input type="email" name="email" required />
         </label>
 
         <label>Phone number
           <input type="tel" name="phone" required />
         </label>
 
         <label>ID document (JPG, PNG, or PDF)
           <input type="file" name="document" accept=".jpg,.jpeg,.png,.pdf" required />
         </label>
 
         <button type="submit">Submit</button>
       </form>
 
       <p id="message" class="message"></p>
     </main>
 
     <script src="form.js"></script>
   </body>
   </html>
```
 
3. Create `public/form.js`. It collects the fields into `FormData` and sends them to your Python route. (The file input is already in the HTML, but we'll only *send* the file in Phase 4 — for now just the text fields.)
```js
   const form = document.getElementById('kyc-form');
   const message = document.getElementById('message');
 
   form.addEventListener('submit', async (e) => {
     e.preventDefault();
     message.textContent = 'Submitting...';
 
     const formData = new FormData();
     formData.append('full_name', form.full_name.value.trim());
     formData.append('email', form.email.value.trim());
     formData.append('phone', form.phone.value.trim());
 
     try {
       const res = await fetch('/api/submissions', {
         method: 'POST',
         body: formData, // do NOT set a Content-Type header — the browser sets it correctly
       });
       if (!res.ok) {
         const err = await res.json();
         throw new Error(err.error || 'Submission failed');
       }
       form.reset();
       message.textContent = 'Submitted successfully. Thank you!';
     } catch (err) {
       message.textContent = err.message;
     }
   });
```
 
**Why `FormData` (and not JSON):** `FormData` lets us send text fields *and* a file in the same request later, with no rewrite. On the Python side, these arrive in `request.form` (text) and `request.files` (the file). Your mentor will explain the JS, but this is the shape it takes.
 
**Verify:** Reload `http://localhost:3000`, fill in the form, submit. You should see the success message, and a new row should appear in the database. (The file isn't saved yet — that's next.)
 
**Checkpoint** The form submits and saves the text fields.
 
---
 
## Phase 4 — Handling the file upload
 
**Objective:** actually receive, validate, and store the uploaded document.
 
This is the most important phase for a KYC system. Read every "why" note here carefully.
 
**Steps:**
 
1. At the top of `app.py`, add the imports and configuration:
```python
   import os
   import secrets
 
   UPLOAD_FOLDER = 'uploads'
   ALLOWED_MIME = {'image/jpeg', 'image/png', 'application/pdf'}
   ALLOWED_EXT = {'.jpg', '.jpeg', '.png', '.pdf'}
   MAX_SIZE = 5 * 1024 * 1024  # 5 MB
 
   os.makedirs(UPLOAD_FOLDER, exist_ok=True)
   app.config['MAX_CONTENT_LENGTH'] = MAX_SIZE  # Flask rejects bigger requests automatically
```
 
2. Replace the submission route so it also handles the file:
```python
   @app.route('/api/submissions', methods=['POST'])
   def create_submission():
       full_name = request.form.get('full_name')
       email = request.form.get('email')
       phone = request.form.get('phone')
       file = request.files.get('document')
 
       if not full_name or not email or not phone or not file:
           return jsonify(error='All fields and a document are required.'), 400
 
       # Validate the file type by BOTH its declared type and its extension
       ext = os.path.splitext(file.filename)[1].lower()
       if file.mimetype not in ALLOWED_MIME or ext not in ALLOWED_EXT:
           return jsonify(error='Only JPG, PNG, or PDF files are allowed.'), 400
 
       # Generate our OWN safe filename — never trust the user's
       safe_name = secrets.token_hex(16) + ext
       file.save(os.path.join(UPLOAD_FOLDER, safe_name))
 
       conn = get_db()
       cur = conn.execute(
           """INSERT INTO submissions
                (full_name, email, phone, document_filename, document_original_name, document_mime)
              VALUES (?, ?, ?, ?, ?, ?)""",
           (full_name, email, phone, safe_name, file.filename, file.mimetype)
       )
       conn.commit()
       new_id = cur.lastrowid
       conn.close()
       return jsonify(id=new_id), 201
```
 
3. Add an error handler so an oversized file returns a clean message instead of an ugly error page:
```python
   @app.errorhandler(413)
   def too_large(e):
       return jsonify(error='File is too large (max 5 MB).'), 413
```
 
4. Update `public/form.js` — just **one new line** to attach the file to the `FormData`:
```js
   formData.append('document', form.document.files[0]);
```
   Add it right after the three `formData.append(...)` lines for the text fields. Nothing else in `form.js` changes.
 
**Why we rename the file with random bytes (`secrets.token_hex`):** never trust the user's filename. It could collide with another file, contain dangerous characters, or be crafted to overwrite something. Generating our own random name avoids all of that. We keep the original name in the database only for display.
 
**Why the size limit and the type filter:** a KYC endpoint is a target. Without a size limit, someone can upload a huge file and exhaust your disk. Without a type filter, someone can upload an executable or script. We check both the MIME type *and* the extension because either one alone can be faked.
 
**Verify:** Submit the form with a real image or PDF. Confirm (a) the success message, (b) a randomly-named file appears in `uploads/`, and (c) the database row now has the real filename and mime type. Then test the guards: try uploading a `.txt` file (should be rejected) and a file over 5 MB (should be rejected with a clean message).
 
**Checkpoint** Real files upload, get stored safely, and bad files are rejected.
 
---
 
## Phase 5 — The admin dashboard (read-only first)
 
**Objective:** a page that lists every submission. We'll add approve/reject in Phase 6.
 
**Steps:**
 
1. Add two routes to `app.py` — one to list submissions, one to serve a document securely:
```python
   # List all submissions (newest first)
   @app.route('/api/submissions', methods=['GET'])
   def list_submissions():
       conn = get_db()
       rows = conn.execute(
           'SELECT * FROM submissions ORDER BY created_at DESC'
       ).fetchall()
       conn.close()
       return jsonify([dict(row) for row in rows])
 
   # Serve a single document through a controlled route
   @app.route('/api/submissions/<int:sub_id>/document', methods=['GET'])
   def get_document(sub_id):
       conn = get_db()
       row = conn.execute(
           'SELECT document_filename FROM submissions WHERE id = ?', (sub_id,)
       ).fetchone()
       conn.close()
       if row is None:
           return jsonify(error='Not found'), 404
       return send_from_directory(UPLOAD_FOLDER, row['document_filename'])
```
 
2. Create `public/admin.html`:
```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
     <meta charset="UTF-8" />
     <meta name="viewport" content="width=device-width, initial-scale=1.0" />
     <title>KYC Admin Dashboard</title>
     <link rel="stylesheet" href="styles.css" />
   </head>
   <body>
     <main class="dashboard">
       <h1>KYC Submissions</h1>
       <table id="submissions-table">
         <thead>
           <tr>
             <th>Name</th><th>Email</th><th>Phone</th>
             <th>Document</th><th>Status</th><th>Submitted</th><th>Actions</th>
           </tr>
         </thead>
         <tbody id="table-body"></tbody>
       </table>
     </main>
     <script src="admin.js"></script>
   </body>
   </html>
```
 
3. Create `public/admin.js` to fetch and render the rows:
```js
   const tableBody = document.getElementById('table-body');
 
   async function loadSubmissions() {
     const res = await fetch('/api/submissions');
     const submissions = await res.json();
 
     tableBody.innerHTML = '';
     submissions.forEach((s) => {
       const row = document.createElement('tr');
       row.innerHTML = `
         <td>${escapeHtml(s.full_name)}</td>
         <td>${escapeHtml(s.email)}</td>
         <td>${escapeHtml(s.phone)}</td>
         <td><a href="/api/submissions/${s.id}/document" target="_blank">View</a></td>
         <td class="status status-${s.status}">${s.status}</td>
         <td>${s.created_at}</td>
         <td class="actions" data-id="${s.id}"></td>
       `;
       tableBody.appendChild(row);
     });
   }
 
   // Prevents stored XSS — never inject user text as raw HTML
   function escapeHtml(str) {
     return String(str)
       .replace(/&/g, '&amp;').replace(/</g, '&lt;')
       .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
   }
 
   loadSubmissions();
```
 
**Why `escapeHtml`:** the name/email/phone came from a stranger. If a user typed `<script>...</script>` as their name and you injected it straight into the page, it would run in the admin's browser — a **stored XSS** attack. Escaping neutralises it. In a KYC tool, the admin is a high-value target, so this matters. (Your mentor will explain this JS; the takeaway is the *principle*: never display untrusted input as raw HTML.)
 
**Verify:** Open `http://localhost:3000/admin.html`. Your earlier test submissions should appear in the table, each with a working "View" link that opens the uploaded document.
 
**Checkpoint** The dashboard lists all submissions and documents open correctly.
 
---
 
## Phase 6 — Approve / reject
 
**Objective:** let the admin change a submission's status.
 
**Steps:**
 
1. Add the status-update route to `app.py`:
```python
   @app.route('/api/submissions/<int:sub_id>/status', methods=['PATCH'])
   def update_status(sub_id):
       data = request.get_json(silent=True) or {}
       status = data.get('status')
 
       if status not in ('approved', 'rejected'):
           return jsonify(error='Invalid status.'), 400
 
       conn = get_db()
       cur = conn.execute(
           """UPDATE submissions
                 SET status = ?, reviewed_at = datetime('now')
               WHERE id = ?""",
           (status, sub_id)
       )
       conn.commit()
       changed = cur.rowcount
       conn.close()
 
       if changed == 0:
           return jsonify(error='Not found'), 404
       return jsonify(ok=True)
```
 
**Why we only accept `approved` or `rejected`:** never let the client send an arbitrary status string into your database. Whitelist the allowed values and reject anything else.
 
2. In `public/admin.js`, fill the actions cell with buttons and wire them up:
```js
   // Inside the submissions.forEach loop, AFTER appending the row:
   const actionsCell = row.querySelector('.actions');
   if (s.status === 'pending') {
     actionsCell.appendChild(makeButton('Approve', s.id, 'approved'));
     actionsCell.appendChild(makeButton('Reject', s.id, 'rejected'));
   } else {
     actionsCell.textContent = '—';
   }
 
   // Add these two functions at the bottom of admin.js:
   function makeButton(label, id, status) {
     const btn = document.createElement('button');
     btn.textContent = label;
     btn.className = 'btn-' + status;
     btn.addEventListener('click', () => updateStatus(id, status));
     return btn;
   }
 
   async function updateStatus(id, status) {
     const res = await fetch(`/api/submissions/${id}/status`, {
       method: 'PATCH',
       headers: { 'Content-Type': 'application/json' },
       body: JSON.stringify({ status }),
     });
     if (res.ok) {
       loadSubmissions(); // refresh the table to show the new status
     } else {
       alert('Could not update status.');
     }
   }
```
 
3. Add status colours in `styles.css` (e.g. green for approved, red for rejected, grey for pending) so the dashboard is readable at a glance.
**Verify:** On the dashboard, approve one submission and reject another. Confirm (a) the status updates and the buttons disappear, (b) reloading the page keeps the new status (it's saved in the DB, not just on screen).
 
**Checkpoint** Approve and reject work and persist.
 
---
 
## Phase 7 — Lock the admin routes (basic auth)
 
**Objective:** the dashboard and its data shouldn't be open to the whole world. Add a simple password gate in Python.
 
> This is a *learning-level* gate, not production security. Note that clearly when you present the project.
 
**Steps:**
 
1. At the top of `app.py`, add:
```python
   from functools import wraps
   from flask import Response
 
   def admin_auth(f):
       @wraps(f)
       def wrapper(*args, **kwargs):
           auth = request.authorization
           if auth and auth.username == 'admin' and auth.password == 'change-this-password':
               return f(*args, **kwargs)
           return Response(
               'Authentication required', 401,
               {'WWW-Authenticate': 'Basic realm="KYC Admin"'}
           )
       return wrapper
```
 
2. Add `@admin_auth` to each admin-only route — the list route, the document route, and the status route. Leave the public submission route (`POST /api/submissions`) open. For example:
```python
   @app.route('/api/submissions', methods=['GET'])
   @admin_auth
   def list_submissions():
       ...
```
   Do the same for `get_document` and `update_status`.
 
**Why the document route is protected too:** the uploaded files *are* the most sensitive data in the system. There's no reason an unauthenticated visitor should reach them. The browser remembers your login for the same realm, so the dashboard's "View" links keep working once you're in.
 
**Verify:** Open the dashboard — the browser should prompt for a username and password. Enter `admin` / `change-this-password` to get in. Wrong credentials should be rejected. The public form should still work without any login.
 
**Checkpoint** The admin side requires a login; the public form still works without one.
 
---
 
## Definition of done (acceptance criteria)
 
The project is complete when **all** of these are true:
 
- A user can fill the form and upload a JPG, PNG, or PDF, and gets a clear success message.
- Invalid files (wrong type, over 5 MB) and incomplete forms are rejected with a readable message — the server never crashes.
- Uploaded documents are stored in `uploads/` with random filenames and are **not** reachable by guessing a public URL.
- The admin dashboard lists every submission, newest first, and each document opens via its "View" link.
- The admin can approve or reject a submission, and the change survives a page reload.
- The admin routes are behind a password.
- All user text is escaped before being shown on the dashboard, and all database queries use parameterised (`?`) statements.
- The whole thing runs from a clean copy with `pip install -r requirements.txt` then `python app.py`.
---
 
## Stretch goals (only after the above is solid)
 
Pick these up if you finish early — each is a small, real improvement:
 
1. **Search/filter** the dashboard by status (pending / approved / rejected).
2. **Email and phone format validation** on both the browser *and* the Python side (never trust the browser alone — re-check on the server).
3. A **rejection reason** field the admin fills in when rejecting (add a column, accept it in the PATCH route).
4. **Image preview** in the dashboard for image documents (PDFs just keep the link).
5. **Pagination** if there are many submissions.
---
 
## How to work and check in with me
 
- **Commit your work after every passing checkpoint** with a clear message (e.g. `feat: handle file upload with validation`). I want to see incremental commits, not one giant one at the end.
- **At each checkpoint, message me a one-line status** ("Phase 4 done, files uploading and validating"). If you're stuck for more than ~30 minutes, ask — being stuck is fine, staying stuck silently is not.
- The backend is all Python, which you know. When you hit the small JavaScript bits (the form submit and the dashboard buttons), don't worry about mastering JS — follow the steps and ask me to explain anything that's unclear. The "why" notes about security are the most valuable part of this project, so flag any you don't fully follow.
Take it one phase at a time. Don't move on with a broken checkpoint behind you.