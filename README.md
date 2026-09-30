# Student Performance Dashboard — Backend

Full-stack setup: Node.js + Express + MongoDB REST API with a Chart.js frontend.

---

## Folder Structure

```
backend/
├── models/
│   └── Student.js          ← Mongoose schema with virtuals (avgMarks, grade)
├── routes/
│   └── students.js         ← All CRUD + stats endpoints
├── public/
│   └── index.html          ← Frontend (Step 6: served by Express)
├── server.js               ← Express app + CORS + static serving
├── seed.js                 ← Populate MongoDB with 15 sample students
├── .env.example            ← Template — copy to .env
├── package.json
└── README.md
```

---

## Step 1 — Start the Backend

### 1a. Install dependencies

```powershell
cd backend
npm install
```

### 1b. Create your .env file

```powershell
# Windows PowerShell
Copy-Item .env.example .env
```

```bash
# macOS / Linux
cp .env.example .env
```

Edit `.env` if your MongoDB runs on a non-default port or URI:

```
MONGO_URI=mongodb://127.0.0.1:27017/student_dashboard
PORT=5000
CLIENT_ORIGIN=*
```

### 1c. Make sure MongoDB is running

```powershell
# Windows (if installed as a service)
Start-Service MongoDB

# Or start manually
& "C:\Program Files\MongoDB\Server\7.0\bin\mongod.exe" --dbpath C:\data\db
```

### 1d. Seed the database (run once)

```powershell
node seed.js
```

Expected output:
```
✅ Connected to MongoDB
🗑️  Cleared 0 existing records
🌱 Seeded 15 students
👋 Done — MongoDB disconnected
```

Re-running `seed.js` is safe — it clears and re-inserts.

### 1e. Start the server

```powershell
# Production
npm start

# Development (auto-restarts on file change, requires nodemon)
npm run dev
```

Expected output:
```
✅ MongoDB connected: mongodb://127.0.0.1:27017/student_dashboard
🚀 Server running at http://localhost:5000
   API:    http://localhost:5000/api/students
   Health: http://localhost:5000/api/health
```

### 1f. Test with curl

```bash
# Health check
curl http://localhost:5000/api/health

# Get all students
curl http://localhost:5000/api/students

# Get class statistics
curl http://localhost:5000/api/students/stats

# Search by name
curl "http://localhost:5000/api/students?search=sneha"

# Filter by class and grade
curl "http://localhost:5000/api/students?class=10-A&grade=A"

# Sort by attendance descending
curl "http://localhost:5000/api/students?sortBy=attendance&order=desc"

# Get a specific student (replace ID)
curl http://localhost:5000/api/students/REPLACE_WITH_MONGO_ID

# Create a student
curl -X POST http://localhost:5000/api/students \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Student","roll":"R099","class":"10-A","attendance":80,"subjects":{"Maths":70,"Physics":65,"Chemistry":68,"English":72,"CS":75,"Biology":60},"exams":[60,68,72]}'

# Update a student
curl -X PUT http://localhost:5000/api/students/REPLACE_WITH_MONGO_ID \
  -H "Content-Type: application/json" \
  -d '{"name":"Updated Name","roll":"R099","class":"10-A","attendance":85,"subjects":{"Maths":80,"Physics":75,"Chemistry":78,"English":82,"CS":85,"Biology":70},"exams":[70,78,82]}'

# Delete a student
curl -X DELETE http://localhost:5000/api/students/REPLACE_WITH_MONGO_ID
```

---

## Step 2 — CORS (already configured in server.js)

The relevant lines in [`server.js`](server.js):

```js
// ↓ STEP 2 — CORS MIDDLEWARE
app.use(cors({
  origin: process.env.CLIENT_ORIGIN || '*', // '*' allows all origins (dev only)
  methods: ['GET', 'POST', 'PUT', 'DELETE'], // which HTTP verbs are permitted
  allowedHeaders: ['Content-Type'],          // which headers the client may send
}));
```

**For production**, change `.env`:
```
CLIENT_ORIGIN=https://your-production-domain.com
```

---

## Step 3 + 4 — Frontend API Layer

The frontend at `public/index.html` has an `api` object at the top of its `<script>` block.

```js
const BASE_URL = 'http://localhost:5000/api'; // ← change to "/api" for Step 6

const api = {
  getStudents(params)        // GET  /students?search=&class=&grade=&sortBy=&order=
  getStats()                 // GET  /students/stats
  getStudent(id)             // GET  /students/:id   (includes rank)
  createStudent(data)        // POST /students
  updateStudent(id, data)    // PUT  /students/:id
  deleteStudent(id)          // DELETE /students/:id
}
```

All functions are `async`/`await` and throw on non-2xx responses.
The UI wires these to every interactive element (search, filter, sort, add, edit, delete, row click).

---

## Step 5 — Common Connection Errors & Fixes

### Error: `TypeError: Failed to fetch` or `net::ERR_CONNECTION_REFUSED`

**Cause**: The backend is not running or is on a different port.

**Fix**:
1. Start the backend: `npm start` inside the `backend/` folder.
2. Confirm the port in `.env` matches `BASE_URL` in `index.html`.
3. Check nothing else is using port 5000: `netstat -ano | findstr :5000` (Windows).

---

### Error: `Access to fetch at 'http://localhost:5000/api/...' from origin 'null' has been blocked by CORS policy`

**Cause**: Opening `index.html` directly from the filesystem (`file://`) sends `Origin: null`, which some CORS configs reject.

**Fix A** (quickest — dev only): Ensure `CLIENT_ORIGIN=*` in `.env` so Express allows `null` origins.

```
# .env
CLIENT_ORIGIN=*
```

**Fix B** (recommended): Use Step 6 — serve the frontend from Express so both share `http://localhost:5000` and there is no cross-origin request at all.

**Fix C**: Use VS Code Live Server (right-click `index.html` → "Open with Live Server"). Live Server serves from `http://127.0.0.1:5500`, which is a proper origin that CORS accepts. Add it to `.env`:
```
CLIENT_ORIGIN=http://127.0.0.1:5500
```

---

### Error: `MongoServerSelectionError: connect ECONNREFUSED 127.0.0.1:27017`

**Cause**: MongoDB is not running.

**Fix** (Windows):
```powershell
Start-Service MongoDB
# or
& "C:\Program Files\MongoDB\Server\7.0\bin\mongod.exe" --dbpath C:\data\db
```

**Fix** (macOS/Linux):
```bash
brew services start mongodb-community   # macOS Homebrew
sudo systemctl start mongod             # Linux systemd
```

---

### Error: `MongooseError: Operation students.find() buffering timed out`

**Cause**: Mongoose connected but the DB is slow to respond, or the connection string is wrong.

**Fix**: Double-check `MONGO_URI` in `.env`. The default for a local install is:
```
MONGO_URI=mongodb://127.0.0.1:27017/student_dashboard
```

---

### Error: `E11000 duplicate key error … index: students.roll`

**Cause**: You tried to create or update a student with a `roll` number that already exists. The `roll` field has a `unique` index in MongoDB.

**Fix**: Use a different roll number. To list existing rolls:
```bash
curl http://localhost:5000/api/students | python -m json.tool
```

---

### Frontend shows stale data after add/edit/delete

**Cause**: `statsCache` was not cleared, or a refresh was not awaited.

**Fix**: The frontend calls `statsCache = null` before every `refreshAll()`. If you add custom logic, always null the cache and `await refreshAll()`.

---

## Step 6 — Serve Frontend from Backend (no CORS needed)

Everything is already set up. Here's what was done:

### 1. `server.js` serves the `public/` folder

```js
// ↓ STEP 6 — serve frontend from Express (same origin = no CORS)
app.use(express.static(path.join(__dirname, 'public')));
```

### 2. Change BASE_URL in `public/index.html`

Open `public/index.html` and change line 1 of the `<script>` section:

```js
// Before (standalone frontend pointing at backend):
const BASE_URL = 'http://localhost:5000/api';

// After (served from Express — relative URL, no CORS):
const BASE_URL = '/api';
```

### 3. Folder structure for Step 6

```
backend/
├── public/
│   └── index.html    ← your frontend goes here
├── server.js
└── ...
```

### 4. Open the app

Start the server (`npm start`) then open:

```
http://localhost:5000
```

Express serves `public/index.html` automatically for `/` and falls back to it for any non-API route.

---

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/health` | Server health check |
| GET | `/api/students` | List students (supports search, class, grade, sortBy, order, page, limit) |
| GET | `/api/students/stats` | Class statistics + top 5 + at-risk |
| GET | `/api/students/:id` | Single student with rank |
| POST | `/api/students` | Create student |
| PUT | `/api/students/:id` | Update student |
| DELETE | `/api/students/:id` | Delete student |
