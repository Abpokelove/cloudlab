# Cloud Computing Lab Cheat Sheet (v2)

Covers: Exp 2A, 2B, 2C, 3, 4, 5, 6, 8, 9, 10, 11A, 11B, 12 (Exp 1 and 7 omitted).

**Rules for every experiment**
- One experiment = one folder = its own `package.json`. Never mix projects.
- Replace every `YOUR_...` placeholder. Never publish keys, tokens, passwords or `.env` files.
- Code is written **generically**: each experiment has a small `CONFIG` block at the top. In the exam, change only those lines (resource name, fields, API URL) to match the question.
- Commands are labelled **Ubuntu** (also WSL) or **Windows (PowerShell)**. Never paste shell commands into a `scala>` or JS console.
- Cloud services (Firebase, Dropbox, Atlas, Auth0, NASA) need internet and an account. Local fallbacks are given where possible.

---

## 0) One-Time Installation (do this before the lab)

### 0.1 Install table

| Tool | Ubuntu | Windows (PowerShell, as Administrator) |
|---|---|---|
| Git | `sudo apt install -y git` | `winget install Git.Git` |
| Node.js LTS + npm | see below | `winget install OpenJS.NodeJS.LTS` |
| JDK 11 | `sudo apt install -y openjdk-11-jdk` | `winget install EclipseAdoptium.Temurin.11.JDK` |
| JDK 17 | `sudo apt install -y openjdk-17-jdk` | `winget install EclipseAdoptium.Temurin.17.JDK` |
| Maven | `sudo apt install -y maven` | Download zip from maven.apache.org, extract to `C:\maven`, add `C:\maven\bin` to PATH |
| Python 3 + pip | `sudo apt install -y python3 python3-pip` | `winget install Python.Python.3.12` |
| Postman | `sudo snap install postman` | `winget install Postman.Postman` |
| VS Code | `sudo snap install code --classic` | `winget install Microsoft.VisualStudioCode` |
| Eclipse IDE (Java) | `sudo snap install eclipse --classic` | Installer from eclipse.org/downloads (choose "Eclipse IDE for Java Developers") |
| NetBeans | `sudo snap install netbeans --classic` | Installer from netbeans.apache.org/front/main/download |
| Linux tools (WSL) | already Linux | `wsl --install -d Ubuntu` then reboot |

Ubuntu Node.js LTS:
```bash
sudo apt update
sudo apt install -y curl wget unzip build-essential
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
```

Windows: close and reopen PowerShell after each install so PATH refreshes. If `npm` is blocked by "running scripts is disabled":
```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

### 0.2 Verify everything
```bash
git --version
node -v          # v18 or newer is required (built-in fetch)
npm -v
java -version
javac -version
mvn -v
python3 --version
```
Windows: use `python --version` instead of `python3`.

### 0.3 Switch Java version (Hadoop needs 11, Spark 3.5 works with 11/17, CloudSim: see Exp 11A)
```bash
sudo update-alternatives --config java
sudo update-alternatives --config javac
```
Windows: set `JAVA_HOME` in *System Properties -> Environment Variables* to the JDK folder, and put `%JAVA_HOME%\bin` first in PATH.

### 0.4 Standard Node project skeleton (use for 2A, 2B, 2C, 4, 5)
```bash
mkdir my-project && cd my-project
npm init -y
npm install express
npm install --save-dev nodemon        # auto-restart on save
npm pkg set scripts.start="node server.js"
npm pkg set scripts.dev="nodemon server.js"
```
Run with `npm start` (or `npm run dev`). Stop with `Ctrl + C`.

`.gitignore` for every project:
```text
node_modules/
.env
```

### 0.5 Free a busy port
Ubuntu:
```bash
lsof -i :3000
kill -9 <PID>
```
Windows:
```powershell
netstat -ano | findstr :3000
Stop-Process -Id <PID> -Force
```

### 0.6 Test an API without Postman
Ubuntu / Git Bash:
```bash
curl http://localhost:3000/api/users
curl -X POST http://localhost:3000/api/users -H "Content-Type: application/json" -d '{"name":"Sam","email":"sam@example.com"}'
```
Windows PowerShell (works everywhere, no curl needed):
```powershell
Invoke-RestMethod http://localhost:3000/api/users
Invoke-RestMethod -Method Post -Uri http://localhost:3000/api/users -ContentType "application/json" -Body '{"name":"Sam","email":"sam@example.com"}'
Invoke-RestMethod -Method Delete -Uri http://localhost:3000/api/users/1
```

---

## 1) Experiment 2A: RESTful API with Express.js

### Aim
Explore RESTful API design with Express.js (GET, POST, PUT, PATCH, DELETE, status codes, validation, search, pagination).

### Links
[Node.js](https://nodejs.org/en/download) | [Express](https://expressjs.com/en/starter/installing.html) | [Postman](https://www.postman.com/downloads/)

### Setup
```bash
mkdir api-demo && cd api-demo
npm init -y
npm install express
```

### `app.js` (generic: change only the CONFIG block)
```js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// ================= CONFIG: edit for any topic =================
const RESOURCE = 'users';                    // URL becomes /api/users
const FIELDS   = ['name', 'email', 'age'];   // every allowed field
const REQUIRED = ['name', 'email'];          // fields that must be present on create
let items = [
  { id: 1, name: 'Alice', email: 'alice@example.com', age: 21 },
  { id: 2, name: 'Bob',   email: 'bob@example.com',   age: 22 }
];
// ==============================================================

let nextId = items.length ? Math.max(...items.map(i => i.id)) + 1 : 1;
const base = `/api/${RESOURCE}`;

app.use(express.json());

// Logger middleware
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()}  ${req.method} ${req.originalUrl}`);
  next();
});

// Keep only allowed fields (stops clients overwriting "id")
const pick = body => Object.fromEntries(
  Object.entries(body || {}).filter(([k]) => FIELDS.includes(k))
);

app.get('/', (req, res) => res.send(`REST API is running. Try ${base}`));

// READ ALL  (supports ?q=text search, ?page=1&limit=10, ?sort=name)
app.get(base, (req, res) => {
  let list = [...items];
  const { q, sort } = req.query;
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || list.length || 1;

  if (q) {
    const term = String(q).toLowerCase();
    list = list.filter(i => Object.values(i).some(v => String(v).toLowerCase().includes(term)));
  }
  if (sort && FIELDS.includes(sort)) {
    list.sort((a, b) => (a[sort] > b[sort] ? 1 : -1));
  }
  const start = (page - 1) * limit;
  res.status(200).json({ total: list.length, page, data: list.slice(start, start + limit) });
});

// READ ONE
app.get(`${base}/:id`, (req, res) => {
  const item = items.find(i => i.id === parseInt(req.params.id));
  if (!item) return res.status(404).json({ message: `${RESOURCE} not found` });
  res.status(200).json(item);
});

// CREATE
app.post(base, (req, res) => {
  const data = pick(req.body);
  const missing = REQUIRED.filter(f => data[f] === undefined || data[f] === '');
  if (missing.length) return res.status(400).json({ message: `Missing: ${missing.join(', ')}` });
  const item = { id: nextId++, ...data };
  items.push(item);
  res.status(201).json(item);
});

// REPLACE (PUT: all required fields must be sent)
app.put(`${base}/:id`, (req, res) => {
  const index = items.findIndex(i => i.id === parseInt(req.params.id));
  if (index === -1) return res.status(404).json({ message: `${RESOURCE} not found` });
  const data = pick(req.body);
  const missing = REQUIRED.filter(f => data[f] === undefined || data[f] === '');
  if (missing.length) return res.status(400).json({ message: `Missing: ${missing.join(', ')}` });
  items[index] = { id: items[index].id, ...data };
  res.status(200).json(items[index]);
});

// PARTIAL UPDATE (PATCH: send only the changed fields)
app.patch(`${base}/:id`, (req, res) => {
  const index = items.findIndex(i => i.id === parseInt(req.params.id));
  if (index === -1) return res.status(404).json({ message: `${RESOURCE} not found` });
  items[index] = { ...items[index], ...pick(req.body) };
  res.status(200).json(items[index]);
});

// DELETE
app.delete(`${base}/:id`, (req, res) => {
  const index = items.findIndex(i => i.id === parseInt(req.params.id));
  if (index === -1) return res.status(404).json({ message: `${RESOURCE} not found` });
  const [deleted] = items.splice(index, 1);
  res.status(200).json({ message: 'Deleted', deleted });
});

// 404 for unknown routes, then error handler
app.use((req, res) => res.status(404).json({ error: 'Endpoint not found' }));
app.use((err, req, res, next) => {
  console.error(err);
  res.status(err.status || 500).json({ error: err.message || 'Server error' });
});

app.listen(PORT, () => console.log(`Server running at http://localhost:${PORT}`));
```
Run: `node app.js`

### Test table (Postman: Body -> raw -> JSON)
| Operation | Method + URL | Body |
|---|---|---|
| List | `GET http://localhost:3000/api/users` | none |
| Search / page | `GET .../api/users?q=ali&page=1&limit=5&sort=name` | none |
| Read one | `GET .../api/users/1` | none |
| Create | `POST .../api/users` | `{"name":"Sam","email":"sam@example.com","age":20}` |
| Replace | `PUT .../api/users/1` | `{"name":"Asha","email":"asha@example.com","age":23}` |
| Partial update | `PATCH .../api/users/1` | `{"age":24}` |
| Delete | `DELETE .../api/users/1` | none |

Expected status codes: 200 OK, 201 Created, 400 Bad Request, 404 Not Found.

### Adapting to another question (example: "products API")
Change only: `RESOURCE = 'products'`, `FIELDS = ['name','price','stock']`, `REQUIRED = ['name','price']`, and the sample `items`.

### If it fails
- `Cannot find module 'express'`: run `npm install express` inside the project folder.
- `EADDRINUSE`: `PORT=3001 node app.js` (PowerShell: `$env:PORT=3001; node app.js`).
- `req.body` undefined: keep `app.use(express.json())` above the routes and send `Content-Type: application/json`.
- Data resets on restart: expected (memory only). Use Exp 5 for persistence.

---

## 2) Experiment 2B: Consuming Third-Party APIs with Express

### Aim
Integrate third-party APIs into an Express app, keeping the API key on the server.

### Links
[NASA API key](https://api.nasa.gov/) | [Open-Meteo (no key)](https://open-meteo.com/en/docs) | [OpenWeatherMap](https://openweathermap.org/api) | [JSONPlaceholder (no key)](https://jsonplaceholder.typicode.com/)

### Setup
```bash
mkdir api-consumer && cd api-consumer
npm init -y
npm install express dotenv
mkdir public
```

### `.env`
```env
PORT=5000
NASA_API_KEY=DEMO_KEY
OWM_API_KEY=YOUR_OPENWEATHER_KEY
```
`DEMO_KEY` works for NASA with low rate limits. OpenWeather is optional.

### `server.js` (generic: add a new API by adding one entry to `SERVICES`)
```js
require('dotenv').config();
const express = require('express');
const path = require('path');
const app = express();
const PORT = process.env.PORT || 5000;

// ================= CONFIG: one entry per third-party API =================
const SERVICES = {
  nasa: {                                   // needs key (kept server-side)
    url: 'https://api.nasa.gov/planetary/apod',
    fixed: { api_key: process.env.NASA_API_KEY || 'DEMO_KEY', thumbs: 'true' },
    allow: ['date']                         // query params the browser may send
  },
  weather: {                                // no key needed
    url: 'https://api.open-meteo.com/v1/forecast',
    fixed: { current_weather: 'true' },
    allow: ['latitude', 'longitude']
  },
  openweather: {                            // needs key
    url: 'https://api.openweathermap.org/data/2.5/weather',
    fixed: { appid: process.env.OWM_API_KEY, units: 'metric' },
    allow: ['q']
  },
  posts: {                                  // fake REST API for practice
    url: 'https://jsonplaceholder.typicode.com/posts',
    fixed: {},
    allow: ['userId', '_limit']
  }
};
// =========================================================================

app.use(express.static(path.join(__dirname, 'public')));

app.get('/api/:service', async (req, res) => {
  const svc = SERVICES[req.params.service];
  if (!svc) return res.status(404).json({ error: 'Unknown service' });

  try {
    const url = new URL(svc.url);
    for (const [k, v] of Object.entries(svc.fixed)) if (v) url.searchParams.set(k, v);
    for (const k of svc.allow) if (req.query[k]) url.searchParams.set(k, req.query[k]);

    const response = await fetch(url);               // Node 18+
    const data = await response.json();
    if (!response.ok) {
      return res.status(response.status).json({ error: data.error?.message || data.message || data.reason || 'Upstream API error' });
    }
    res.json(data);
  } catch (err) {
    console.error('Upstream failed:', err.message);
    res.status(502).json({ error: 'Unable to reach the third-party API' });
  }
});

app.listen(PORT, () => console.log(`Open http://localhost:${PORT}`));
```

### `public/index.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Third-Party API Explorer</title>
  <style>
    body { max-width: 860px; margin: 30px auto; padding: 0 16px; font: 16px/1.5 Arial, sans-serif; color: #222; }
    select, input, button { padding: 8px 10px; font: inherit; margin: 4px 4px 4px 0; }
    button { background: #333; color: #fff; border: 0; border-radius: 4px; cursor: pointer; }
    pre { background: #f5f5f5; padding: 12px; overflow-x: auto; white-space: pre-wrap; }
    img { max-width: 100%; margin-top: 12px; }
  </style>
</head>
<body>
  <h1>Third-Party API Explorer</h1>
  <select id="service">
    <option value="nasa">NASA APOD (param: date)</option>
    <option value="weather">Open-Meteo (params: latitude, longitude)</option>
    <option value="openweather">OpenWeather (param: q = city)</option>
    <option value="posts">JSONPlaceholder (param: _limit)</option>
  </select>
  <input id="params" placeholder="e.g. date=2024-01-01" value="date=2024-01-01" size="34">
  <button onclick="callApi()">Call API</button>
  <p id="msg"></p>
  <img id="img" hidden alt="result image">
  <pre id="out"></pre>
  <script>
    const presets = {
      nasa: 'date=2024-01-01',
      weather: 'latitude=16.3067&longitude=80.4365',
      openweather: 'q=Guntur',
      posts: '_limit=3'
    };
    document.getElementById('service').onchange = e =>
      document.getElementById('params').value = presets[e.target.value];

    async function callApi() {
      const s = document.getElementById('service').value;
      const p = document.getElementById('params').value;
      const msg = document.getElementById('msg'), out = document.getElementById('out'), img = document.getElementById('img');
      msg.textContent = 'Loading...'; out.textContent = ''; img.hidden = true;
      try {
        const res = await fetch(`/api/${s}?${p}`);
        const data = await res.json();
        if (!res.ok) throw new Error(data.error || 'Request failed');
        out.textContent = JSON.stringify(data, null, 2);
        if (data.media_type === 'image') { img.src = data.url; img.hidden = false; }
        msg.textContent = 'Success';
      } catch (e) { msg.textContent = e.message; }
    }
  </script>
</body>
</html>
```

### Run and verify
```bash
node server.js
```
Open `http://localhost:5000` (never double-click the HTML file). API-only check:
```bash
curl "http://localhost:5000/api/nasa?date=2024-01-01"
curl "http://localhost:5000/api/weather?latitude=16.3&longitude=80.4"
```

### If it fails
- `fetch is not defined`: Node < 18. Upgrade (see 0.1).
- NASA 429: wait, or put your own key in `.env`.
- `Cannot GET /`: file must be exactly `public/index.html`.
- OpenWeather 401: key not activated yet (takes up to ~2 hours after signup).
- Changed `.env`: restart the server.

---

## 3) Experiment 2C: CRUD API with Validation (Joi) and Postman Testing

### Aim
Build a CRUD API with Express, validate input with Joi, and test with Postman.

### Links
[Joi](https://joi.dev/api/) | [Postman](https://www.postman.com/downloads/)

### Setup
```bash
mkdir movie-api && cd movie-api
npm init -y
npm install express joi
```

### `app.js` (generic: edit the schema + sample data)
```js
const express = require('express');
const Joi = require('joi');
const app = express();
const PORT = process.env.PORT || 8080;
app.use(express.json());

// ================= CONFIG: edit for any topic =================
const RESOURCE = 'movies';
const schema = Joi.object({
  title:       Joi.string().min(2).max(100).required(),
  genre:       Joi.string().required(),
  director:    Joi.string().required(),
  releaseYear: Joi.number().integer().min(1900).max(2100).required(),
  rating:      Joi.number().min(0).max(10).required(),
  status:      Joi.string().valid('Watched', 'Favourite', 'Planned').required()
});
let items = [
  { id: 1, title: 'Interstellar', genre: 'Science Fiction', director: 'Christopher Nolan', releaseYear: 2014, rating: 9,   status: 'Watched' },
  { id: 2, title: 'The Martian',  genre: 'Science Fiction', director: 'Ridley Scott',      releaseYear: 2015, rating: 8.5, status: 'Favourite' }
];
// ==============================================================

let nextId = Math.max(0, ...items.map(i => i.id)) + 1;
const base = `/api/${RESOURCE}`;
const validate = (body, partial = false) =>
  (partial ? schema.fork(Object.keys(schema.describe().keys), s => s.optional()) : schema)
    .validate(body, { abortEarly: false, stripUnknown: true });

const findIndex = req => items.findIndex(i => i.id === parseInt(req.params.id));

app.get('/', (req, res) => res.send(`Welcome to the ${RESOURCE} API`));

app.get(base, (req, res) => {
  let list = items;
  for (const [k, v] of Object.entries(req.query)) {          // filter: ?genre=Sci&status=Watched
    list = list.filter(i => String(i[k] ?? '').toLowerCase().includes(String(v).toLowerCase()));
  }
  res.status(200).json(list);
});

app.get(`${base}/:id`, (req, res) => {
  const i = findIndex(req);
  if (i === -1) return res.status(404).json({ message: 'Not found' });
  res.status(200).json(items[i]);
});

app.post(base, (req, res) => {
  const { error, value } = validate(req.body);
  if (error) return res.status(400).json({ errors: error.details.map(d => d.message) });
  const item = { id: nextId++, ...value };
  items.push(item);
  res.status(201).json(item);
});

app.put(`${base}/:id`, (req, res) => {
  const i = findIndex(req);
  if (i === -1) return res.status(404).json({ message: 'Not found' });
  const { error, value } = validate(req.body);
  if (error) return res.status(400).json({ errors: error.details.map(d => d.message) });
  items[i] = { id: items[i].id, ...value };
  res.status(200).json(items[i]);
});

app.patch(`${base}/:id`, (req, res) => {
  const i = findIndex(req);
  if (i === -1) return res.status(404).json({ message: 'Not found' });
  const { error, value } = validate(req.body, true);
  if (error) return res.status(400).json({ errors: error.details.map(d => d.message) });
  items[i] = { ...items[i], ...value };
  res.status(200).json(items[i]);
});

app.delete(`${base}/:id`, (req, res) => {
  const i = findIndex(req);
  if (i === -1) return res.status(404).json({ message: 'Not found' });
  const [deleted] = items.splice(i, 1);
  res.status(200).json({ message: 'Deleted successfully', deleted });
});

app.use((req, res) => res.status(404).json({ error: 'API endpoint not found' }));
app.listen(PORT, () => console.log(`API running at http://localhost:${PORT}`));
```
Run: `node app.js`

### Postman steps
1. Open Postman -> **New -> HTTP Request**.
2. For POST/PUT/PATCH: **Body -> raw -> JSON**.
3. Save requests into a **Collection** ("Movie API") and screenshot each response.

| Test | Request | Expected |
|---|---|---|
| List | `GET http://localhost:8080/api/movies` | 200 + array |
| Filter | `GET .../api/movies?genre=Science` | 200 + filtered |
| One | `GET .../api/movies/1` | 200 |
| Create | `POST .../api/movies` (body below) | 201 |
| Invalid create | `POST` with `{"title":"A"}` | 400 + error list |
| Update | `PUT .../api/movies/1` (full body) | 200 |
| Patch | `PATCH .../api/movies/1` `{"rating":9.5}` | 200 |
| Delete | `DELETE .../api/movies/1` | 200 |
| Missing | `GET .../api/movies/99` | 404 |

POST body:
```json
{
  "title": "Avatar",
  "genre": "Sci-Fi",
  "director": "James Cameron",
  "releaseYear": 2009,
  "rating": 8.8,
  "status": "Favourite"
}
```

### Postman test script (paste in the request's **Tests** tab)
```js
pm.test("Status is 201", () => pm.response.to.have.status(201));
pm.test("Has id", () => pm.expect(pm.response.json()).to.have.property("id"));
```

### curl / PowerShell backup
```bash
curl -X POST http://localhost:8080/api/movies -H "Content-Type: application/json" -d '{"title":"Avatar","genre":"Sci-Fi","director":"James Cameron","releaseYear":2009,"rating":8.8,"status":"Favourite"}'
```
```powershell
Invoke-RestMethod -Method Post -Uri http://localhost:8080/api/movies -ContentType "application/json" -Body '{"title":"Avatar","genre":"Sci-Fi","director":"James Cameron","releaseYear":2009,"rating":8.8,"status":"Favourite"}'
```

### If it fails
- `Cannot find module 'joi'`: `npm install joi`.
- Always 400: PUT needs every required field; use PATCH for partial changes.
- Status field rejected: it must match the `valid(...)` list; edit the list in the schema.

---

## 4) Experiment 3: Firebase (Google Auth + Firestore CRUD + Storage + Hosting)

### Aim
Integrate Google Authentication, Firestore CRUD, file upload to Firebase Storage, and deploy with Firebase Hosting.

### Links
[Firebase Console](https://console.firebase.google.com/) | [CLI reference](https://firebase.google.com/docs/cli) | [Web setup](https://firebase.google.com/docs/web/setup) | [Hosting](https://firebase.google.com/docs/hosting)

### Console steps (browser)
1. **Add project** -> give a name -> Create.
2. **Project overview -> `</>` Web app** -> register -> copy the `firebaseConfig` object.
3. **Build -> Authentication -> Get started -> Sign-in method -> Google -> Enable** -> choose a support email -> Save.
4. **Authentication -> Settings -> Authorized domains**: make sure `localhost` is listed (add `127.0.0.1` if you use it).
5. **Build -> Firestore Database -> Create database** (production mode, nearest region).
6. **Build -> Storage -> Get started**. Note: new Storage buckets may require the **Blaze (pay-as-you-go)** plan. If Storage is blocked, demo Auth + Firestore + Hosting and say Storage needs billing.
7. Publish the **Rules** shown below.

### Install and initialise (Ubuntu and Windows are identical)
```bash
mkdir firebase-student && cd firebase-student
npm init -y
npm install --save-dev firebase-tools
npx firebase login                  # opens browser; use "npx firebase login --no-localhost" on a remote VM
npx firebase init hosting
```
Answers during `firebase init hosting`:
- Use an existing project -> select yours
- Public directory: `public`
- Single-page app: **No**
- Automatic builds with GitHub: **No**
- Overwrite `public/index.html`: **No** (you will paste your own)

### Firestore rules (Firestore -> Rules -> Publish)
```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /students/{studentId} {
      allow create: if request.auth != null
        && request.resource.data.ownerUid == request.auth.uid;
      allow read, delete: if request.auth != null
        && resource.data.ownerUid == request.auth.uid;
      allow update: if request.auth != null
        && resource.data.ownerUid == request.auth.uid
        && request.resource.data.ownerUid == request.auth.uid;
    }
  }
}
```

### Storage rules (Storage -> Rules -> Publish)
```text
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /student-files/{userId}/{fileName} {
      allow read, delete: if request.auth != null && request.auth.uid == userId;
      allow create, update: if request.auth != null
        && request.auth.uid == userId
        && request.resource.size < 5 * 1024 * 1024;
    }
  }
}
```

### `public/index.html` (generic: change `COLLECTION` and the field list in `FIELDS`)
Paste your own values from **Project settings -> General -> Your apps -> SDK setup**.
```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Student Profile Manager</title>
  <style>
    * { box-sizing: border-box; }
    body { margin: 0; padding: 28px 16px; color: #242424; background: #fff; font: 16px/1.5 Arial, sans-serif; }
    main { max-width: 760px; margin: auto; }
    .card { border: 1px solid #ddd; border-radius: 8px; padding: 20px; margin: 16px 0; }
    label { display: block; margin: 12px 0 4px; }
    input { width: 100%; max-width: 520px; padding: 9px; border: 1px solid #aaa; border-radius: 4px; font: inherit; }
    button { margin: 12px 8px 0 0; padding: 9px 14px; color: #fff; background: #333; border: 0; border-radius: 4px; cursor: pointer; }
    button.secondary { color: #222; background: #eee; }
    .row { border-top: 1px solid #eee; padding: 12px 0; }
  </style>
</head>
<body>
<main>
  <h1>Student Profile Manager</h1>
  <section class="card">
    <button id="signIn">Sign in with Google</button>
    <button id="signOut" class="secondary" hidden>Sign out</button>
    <p id="account">Not signed in</p>
  </section>
  <section class="card">
    <h2>Student details</h2>
    <form id="studentForm">
      <input id="studentId" type="hidden">
      <label for="name">Name</label><input id="name" required maxlength="80">
      <label for="email">Email</label><input id="email" type="email" required maxlength="120">
      <label for="course">Course</label><input id="course" required maxlength="80">
      <label for="attachment">Optional attachment (max 5 MB)</label><input id="attachment" type="file">
      <button type="submit">Save profile</button>
      <button id="cancelEdit" type="button" class="secondary" hidden>Cancel edit</button>
    </form>
  </section>
  <section class="card">
    <h2>Saved profiles</h2>
    <button id="refresh" class="secondary">Refresh list</button>
    <div id="students"></div>
  </section>
  <p id="status" role="status"></p>
</main>
<script type="module">
  import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js';
  import { getAuth, GoogleAuthProvider, signInWithPopup, signOut, onAuthStateChanged }
    from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js';
  import { getFirestore, collection, addDoc, getDocs, doc, updateDoc, deleteDoc, query, where }
    from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js';
  import { getStorage, ref, uploadBytes, getDownloadURL, deleteObject }
    from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-storage.js';

  // ================= CONFIG =================
  const firebaseConfig = {
    apiKey: 'YOUR_API_KEY',
    authDomain: 'YOUR_PROJECT_ID.firebaseapp.com',
    projectId: 'YOUR_PROJECT_ID',
    storageBucket: 'YOUR_STORAGE_BUCKET',
    messagingSenderId: 'YOUR_SENDER_ID',
    appId: 'YOUR_APP_ID'
  };
  const COLLECTION = 'students';                 // must match the name used in the Firestore rules
  // ==========================================

  const app = initializeApp(firebaseConfig);
  const auth = getAuth(app), db = getFirestore(app), storage = getStorage(app);
  const provider = new GoogleAuthProvider();
  const $ = id => document.getElementById(id);
  const form = $('studentForm'), status = $('status');
  let currentUser = null;

  const reportError = e => { console.error(e); status.textContent = e.message || 'Operation failed (see console).'; };
  const clearForm = () => { form.reset(); $('studentId').value = ''; $('cancelEdit').hidden = true; };

  $('signIn').onclick  = async () => { try { await signInWithPopup(auth, provider); } catch (e) { reportError(e); } };
  $('signOut').onclick = async () => { try { await signOut(auth); clearForm(); } catch (e) { reportError(e); } };
  $('cancelEdit').onclick = clearForm;
  $('refresh').onclick = loadStudents;

  onAuthStateChanged(auth, async user => {
    currentUser = user;
    $('signIn').hidden = Boolean(user);
    $('signOut').hidden = !user;
    $('account').textContent = user ? `Signed in as ${user.email}` : 'Not signed in';
    $('students').replaceChildren();
    if (user) await loadStudents();
  });

  async function loadStudents() {
    if (!currentUser) { status.textContent = 'Sign in first.'; return; }
    try {
      const q = query(collection(db, COLLECTION), where('ownerUid', '==', currentUser.uid));
      const snap = await getDocs(q);
      const list = $('students');
      list.replaceChildren();
      snap.forEach(d => {
        const s = d.data();
        const row = document.createElement('div');
        row.className = 'row';
        const p = document.createElement('p');
        p.textContent = `${s.name} — ${s.email} — ${s.course}`;
        row.append(p);
        if (s.fileUrl) {
          const a = document.createElement('a');
          a.href = s.fileUrl; a.textContent = 'Open attachment'; a.target = '_blank'; a.rel = 'noreferrer';
          row.append(a);
        }
        const edit = document.createElement('button');
        edit.textContent = 'Edit';
        edit.onclick = () => {
          $('studentId').value = d.id; $('name').value = s.name; $('email').value = s.email; $('course').value = s.course;
          $('cancelEdit').hidden = false;
        };
        const del = document.createElement('button');
        del.textContent = 'Delete'; del.className = 'secondary';
        del.onclick = async () => {
          if (!confirm(`Delete ${s.name}?`)) return;
          try {
            await deleteDoc(doc(db, COLLECTION, d.id));
            if (s.filePath) await deleteObject(ref(storage, s.filePath));
            status.textContent = 'Deleted.';
            await loadStudents();
          } catch (e) { reportError(e); }
        };
        row.append(edit, del);
        list.append(row);
      });
      status.textContent = snap.empty ? 'No profiles yet.' : `Loaded ${snap.size} profile(s).`;
    } catch (e) { reportError(e); }
  }

  form.onsubmit = async event => {
    event.preventDefault();
    if (!currentUser) { status.textContent = 'Sign in first.'; return; }
    try {
      const id = $('studentId').value;
      const data = {
        name: $('name').value.trim(),
        email: $('email').value.trim(),
        course: $('course').value.trim(),
        ownerUid: currentUser.uid
      };
      const file = $('attachment').files[0];
      if (file) {
        if (file.size > 5 * 1024 * 1024) throw new Error('File must be under 5 MB.');
        const safe = file.name.replace(/[^a-zA-Z0-9._-]/g, '_');
        const filePath = `student-files/${currentUser.uid}/${Date.now()}-${safe}`;
        const r = ref(storage, filePath);
        await uploadBytes(r, file);
        data.filePath = filePath;
        data.fileUrl = await getDownloadURL(r);
      }
      if (id) { await updateDoc(doc(db, COLLECTION, id), data); status.textContent = 'Updated.'; }
      else    { await addDoc(collection(db, COLLECTION), { ...data, createdAt: new Date().toISOString() }); status.textContent = 'Saved.'; }
      clearForm();
      await loadStudents();
    } catch (e) { reportError(e); }
  };
</script>
</body>
</html>
```

### Test locally and deploy
```bash
npx firebase serve --only hosting          # older CLI
npx firebase emulators:start --only hosting   # newer CLI (default http://127.0.0.1:5000)
npx firebase deploy --only hosting
```
Open the printed `https://YOUR_PROJECT_ID.web.app` URL. Verify: sign in, create, refresh, edit, upload a file, delete.

### If it fails
- `auth/unauthorized-domain`: add the exact hostname to Authorized domains.
- `Missing or insufficient permissions`: publish the rules; confirm `ownerUid` equals the signed-in user.
- Storage `unauthorized` or bucket error: copy `storageBucket` exactly; enable Storage (may need Blaze plan).
- Popup blocked: allow popups, or switch to `signInWithRedirect`.
- `firebase: command not found`: always use `npx firebase ...`.

---

## 5) Experiment 4: Dropbox Storage with Express.js

### Aim
Upload a file to Dropbox, list files, create a temporary download link, and delete a file. The token stays on the server.

### Links
[App Console](https://www.dropbox.com/developers/apps) | [SDK](https://github.com/dropbox/dropbox-sdk-js) | [API Explorer](https://dropbox.github.io/dropbox-api-v2-explorer/)

### Create the app and token
1. App Console -> **Create app** -> **Scoped access** -> **App folder** -> name it.
2. **Permissions** tab: tick `files.metadata.read`, `files.content.read`, `files.content.write`, then **Submit**.
3. **Settings** tab -> **Generated access token** -> **Generate**. Copy it. These tokens are short-lived (about 4 hours); generate a new one if you get a 401.

### Setup
```bash
mkdir dropbox-express && cd dropbox-express
npm init -y
npm install express dotenv dropbox multer
mkdir public
```
`.env`:
```env
DROPBOX_ACCESS_TOKEN=paste_token_here
PORT=3000
```

### `server.js`
```js
require('dotenv').config();
const express = require('express');
const path = require('path');
const multer = require('multer');
const { Dropbox } = require('dropbox');

const app = express();
const port = Number(process.env.PORT) || 3000;
const token = process.env.DROPBOX_ACCESS_TOKEN;
if (!token) { console.error('Set DROPBOX_ACCESS_TOKEN in .env'); process.exit(1); }

const dbx = new Dropbox({ accessToken: token, fetch: globalThis.fetch });
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 10 * 1024 * 1024 } });

app.use(express.json());                      // REQUIRED for the DELETE route (reads req.body)
app.use(express.static(path.join(__dirname, 'public')));

// LIST
app.get('/api/files', async (req, res) => {
  try {
    const r = await dbx.filesListFolder({ path: '' });
    res.json(r.result.entries.filter(e => e['.tag'] === 'file').map(f => ({ name: f.name, path: f.path_display, size: f.size })));
  } catch (e) {
    console.error(e);
    res.status(502).json({ error: 'Could not list files. Check token and permissions.' });
  }
});

// UPLOAD
app.post('/api/upload', upload.single('file'), async (req, res) => {
  if (!req.file) return res.status(400).json({ error: 'Choose a file first.' });
  const safeName = path.basename(req.file.originalname).replace(/[^a-zA-Z0-9._ -]/g, '_');
  try {
    const r = await dbx.filesUpload({ path: `/${safeName}`, contents: req.file.buffer, mode: { '.tag': 'overwrite' } });
    res.status(201).json({ message: 'Upload complete.', name: r.result.name });
  } catch (e) {
    console.error(e);
    res.status(502).json({ error: 'Upload failed. Check token, permissions and connection.' });
  }
});

// DOWNLOAD (temporary link, valid ~4 hours)
app.get('/api/download', async (req, res) => {
  const p = typeof req.query.path === 'string' ? req.query.path : '';
  if (!p.startsWith('/')) return res.status(400).json({ error: 'File path required.' });
  try {
    const r = await dbx.filesGetTemporaryLink({ path: p });
    res.json({ url: r.result.link });
  } catch (e) {
    console.error(e);
    res.status(502).json({ error: 'Could not create a download link.' });
  }
});

// DELETE
app.delete('/api/files', async (req, res) => {
  const p = typeof req.body?.path === 'string' ? req.body.path : '';
  if (!p.startsWith('/')) return res.status(400).json({ error: 'File path required.' });
  try {
    await dbx.filesDeleteV2({ path: p });
    res.json({ message: 'File deleted.' });
  } catch (e) {
    console.error(e);
    res.status(502).json({ error: 'Could not delete the file.' });
  }
});

app.use((err, req, res, next) => {
  if (err instanceof multer.MulterError && err.code === 'LIMIT_FILE_SIZE')
    return res.status(413).json({ error: 'File too large (10 MB limit).' });
  console.error(err);
  res.status(500).json({ error: 'Unexpected server error.' });
});

app.listen(port, () => console.log(`Open http://localhost:${port}`));
```

### `public/index.html`
```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Dropbox File Cabinet</title>
  <style>
    body { max-width: 760px; margin: 32px auto; padding: 0 18px; font: 16px/1.5 Arial, sans-serif; color: #242424; }
    section { margin: 20px 0; padding: 18px; border: 1px solid #ddd; border-radius: 8px; }
    button { margin: 6px 6px 6px 0; padding: 8px 13px; color: #fff; background: #333; border: 0; border-radius: 4px; cursor: pointer; }
    button.delete { color: #222; background: #eee; }
    li { padding: 8px 0; border-bottom: 1px solid #eee; }
  </style>
</head>
<body>
  <h1>Dropbox File Cabinet</h1>
  <section>
    <h2>Upload</h2>
    <form id="uploadForm"><input id="file" type="file" required><button type="submit">Upload to Dropbox</button></form>
    <p>Max 10 MB. Same name replaces the old file.</p>
  </section>
  <section>
    <h2>Files</h2>
    <button id="refresh" type="button">Refresh list</button>
    <ul id="files"></ul>
  </section>
  <p id="status" role="status"></p>
  <script>
    const statusLine = document.getElementById('status');
    async function request(url, options) {
      const r = await fetch(url, options);
      const data = await r.json();
      if (!r.ok) throw new Error(data.error || `Request failed (${r.status})`);
      return data;
    }
    async function loadFiles() {
      try {
        const files = await request('/api/files');
        const list = document.getElementById('files');
        list.replaceChildren();
        for (const f of files) {
          const li = document.createElement('li');
          const open = document.createElement('button'); open.textContent = 'Open/download';
          open.onclick = async () => {
            try { const d = await request(`/api/download?path=${encodeURIComponent(f.path)}`); window.open(d.url, '_blank', 'noopener'); }
            catch (e) { statusLine.textContent = e.message; }
          };
          const del = document.createElement('button'); del.textContent = 'Delete'; del.className = 'delete';
          del.onclick = async () => {
            if (!confirm(`Delete ${f.name}?`)) return;
            try {
              await request('/api/files', { method: 'DELETE', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ path: f.path }) });
              statusLine.textContent = 'Deleted.'; await loadFiles();
            } catch (e) { statusLine.textContent = e.message; }
          };
          li.append(document.createTextNode(`${f.name} `), open, del);
          list.append(li);
        }
        statusLine.textContent = files.length ? `${files.length} file(s).` : 'No files yet.';
      } catch (e) { statusLine.textContent = e.message; }
    }
    document.getElementById('uploadForm').onsubmit = async ev => {
      ev.preventDefault();
      const file = document.getElementById('file').files[0];
      if (!file) return;
      const fd = new FormData(); fd.append('file', file);
      statusLine.textContent = 'Uploading...';
      try { const r = await request('/api/upload', { method: 'POST', body: fd }); statusLine.textContent = r.message; ev.target.reset(); await loadFiles(); }
      catch (e) { statusLine.textContent = e.message; }
    };
    document.getElementById('refresh').onclick = loadFiles;
    loadFiles();
  </script>
</body>
</html>
```

### Run and verify
```bash
node server.js
```
Open `http://localhost:3000`: upload -> refresh -> open -> delete. Check **Dropbox -> Apps -> your app** as an independent proof.

### Check the token without the app (Ubuntu)
```bash
curl -X POST https://api.dropboxapi.com/2/files/list_folder \
  --header "Authorization: Bearer YOUR_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"path":""}'
```

### If it fails
- 401 / `expired_access_token`: generate a new token, update `.env`, restart.
- 403 / missing scope: tick the scopes, **Submit**, then generate a NEW token.
- `Cannot DELETE`/empty body: `express.json()` must be present (it is in this version).
- `LIMIT_FILE_SIZE`: use a file under 10 MB.

---

## 6) Experiment 5: MongoDB with CRUD and Node.js (CodeSnipper)

### Aim
Build an app that saves, displays, edits and deletes code snippets in MongoDB. Atlas (cloud) or local MongoDB both work.

### Links
[Atlas](https://www.mongodb.com/cloud/atlas) | [Compass](https://www.mongodb.com/try/download/compass) | [Mongoose](https://mongoosejs.com/docs/guide.html)

### Option A: MongoDB Atlas (cloud)
1. Create a free **M0** cluster.
2. **Database Access -> Add New Database User** (username + password; this is NOT your Atlas login).
3. **Network Access -> Add IP Address -> Add Current IP**.
4. **Database -> Connect -> Drivers -> Node.js** -> copy the URI. Replace `<password>`; append the DB name `/codesnipper` before `?`. URL-encode special characters in the password (`@` -> `%40`).

### Option B: Local MongoDB (offline fallback)
Ubuntu 22.04 / 24.04:
```bash
sudo apt install -y gnupg curl
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
mongosh --eval "db.runCommand({ ping: 1 })"
```
Windows PowerShell (Administrator):
```powershell
winget install MongoDB.Server
winget install MongoDB.Shell
Get-Service MongoDB          # should be Running
mongosh --eval "db.runCommand({ ping: 1 })"
```
Local URI: `mongodb://127.0.0.1:27017/codesnipper`

### Setup
```bash
mkdir mongodb-codesnipper && cd mongodb-codesnipper
npm init -y
npm install express mongoose dotenv
mkdir public
```
`.env`:
```env
MONGODB_URI=mongodb+srv://USERNAME:PASSWORD@cluster0.xxxxx.mongodb.net/codesnipper?retryWrites=true&w=majority
PORT=5000
```

### `server.js` (generic: change `MODEL_NAME` and the schema)
```js
require('dotenv').config();
const express = require('express');
const mongoose = require('mongoose');
const path = require('path');

const app = express();
const port = Number(process.env.PORT) || 5000;
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// ================= CONFIG =================
const MODEL_NAME = 'Snippet';          // collection becomes "snippets"
const schema = new mongoose.Schema({
  title:     { type: String, required: true, trim: true, maxlength: 100 },
  language:  { type: String, required: true, trim: true, maxlength: 40 },
  code:      { type: String, required: true, maxlength: 50000 },
  createdAt: { type: Date, default: Date.now }
});
const FIELDS = ['title', 'language', 'code'];   // fields the client may send
// ==========================================
const Model = mongoose.model(MODEL_NAME, schema);
const pick = b => Object.fromEntries(FIELDS.filter(f => b?.[f] !== undefined).map(f => [f, b[f]]));

app.get('/api/snippets', async (req, res, next) => {
  try {
    const filter = req.query.q ? { title: new RegExp(String(req.query.q).replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'i') } : {};
    res.json(await Model.find(filter).sort({ createdAt: -1 }));
  } catch (e) { next(e); }
});

app.get('/api/snippets/:id', async (req, res, next) => {
  try {
    const doc = await Model.findById(req.params.id);
    if (!doc) return res.status(404).json({ error: 'Not found.' });
    res.json(doc);
  } catch (e) { next(e); }
});

app.post('/api/snippets', async (req, res, next) => {
  try { res.status(201).json(await Model.create(pick(req.body))); }
  catch (e) { next(e); }
});

app.put('/api/snippets/:id', async (req, res, next) => {
  try {
    const doc = await Model.findByIdAndUpdate(req.params.id, pick(req.body), { new: true, runValidators: true });
    if (!doc) return res.status(404).json({ error: 'Not found.' });
    res.json(doc);
  } catch (e) { next(e); }
});

app.delete('/api/snippets/:id', async (req, res, next) => {
  try {
    const doc = await Model.findByIdAndDelete(req.params.id);
    if (!doc) return res.status(404).json({ error: 'Not found.' });
    res.json({ message: 'Deleted.', doc });
  } catch (e) { next(e); }
});

app.use((err, req, res, next) => {
  console.error(err.message);
  if (err.name === 'ValidationError') return res.status(400).json({ error: err.message });
  if (err.name === 'CastError') return res.status(400).json({ error: 'Invalid ID.' });
  res.status(500).json({ error: 'Server error. See terminal.' });
});

(async () => {
  if (!process.env.MONGODB_URI) throw new Error('MONGODB_URI missing in .env');
  await mongoose.connect(process.env.MONGODB_URI, { serverSelectionTimeoutMS: 10000 });
  console.log('Connected to MongoDB');
  app.listen(port, () => console.log(`Open http://localhost:${port}`));
})().catch(e => { console.error('Startup failed:', e.message); process.exit(1); });
```

### `public/index.html`
```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>CodeSnipper</title>
  <style>
    * { box-sizing: border-box; }
    body { max-width: 850px; margin: 30px auto; padding: 0 18px; font: 16px/1.5 Arial, sans-serif; color: #242424; }
    section, article { border: 1px solid #ddd; border-radius: 8px; padding: 18px; margin: 16px 0; }
    label { display: block; margin: 10px 0 4px; }
    input, textarea { width: 100%; padding: 9px; border: 1px solid #aaa; border-radius: 4px; font: inherit; }
    textarea { min-height: 130px; font-family: Consolas, monospace; }
    button { margin: 10px 8px 0 0; padding: 8px 13px; color: #fff; background: #333; border: 0; border-radius: 4px; cursor: pointer; }
    button.secondary { color: #222; background: #eee; }
    pre { overflow-x: auto; padding: 12px; background: #f6f6f6; white-space: pre-wrap; }
  </style>
</head>
<body>
  <h1>CodeSnipper</h1>
  <section>
    <h2 id="formHeading">Add a snippet</h2>
    <form id="snippetForm">
      <input id="snippetId" type="hidden">
      <label for="title">Title</label><input id="title" required maxlength="100">
      <label for="language">Language</label><input id="language" required maxlength="40" placeholder="JavaScript, Java, Python...">
      <label for="code">Code</label><textarea id="code" required></textarea>
      <button type="submit">Save snippet</button>
      <button id="cancel" type="button" class="secondary" hidden>Cancel edit</button>
    </form>
  </section>
  <section>
    <h2>Saved snippets</h2>
    <input id="search" placeholder="Search by title...">
    <button id="refresh" type="button" class="secondary">Refresh</button>
    <div id="output"></div>
  </section>
  <p id="status" role="status"></p>
  <script>
    const $ = id => document.getElementById(id);
    const form = $('snippetForm'), statusLine = $('status');
    async function api(url, options = {}) {
      const r = await fetch(url, options);
      const data = await r.json();
      if (!r.ok) throw new Error(data.error || `Request failed (${r.status})`);
      return data;
    }
    function clearForm() { form.reset(); $('snippetId').value = ''; $('formHeading').textContent = 'Add a snippet'; $('cancel').hidden = true; }

    async function loadSnippets() {
      try {
        const q = encodeURIComponent($('search').value.trim());
        const snippets = await api('/api/snippets' + (q ? `?q=${q}` : ''));
        const out = $('output'); out.replaceChildren();
        if (!snippets.length) out.textContent = 'No snippets found.';
        for (const s of snippets) {
          const card = document.createElement('article');
          const h = document.createElement('h3'); h.textContent = `${s.title} (${s.language})`;
          const pre = document.createElement('pre'); pre.textContent = s.code;
          const edit = document.createElement('button'); edit.textContent = 'Edit';
          edit.onclick = () => {
            $('snippetId').value = s._id; $('title').value = s.title; $('language').value = s.language; $('code').value = s.code;
            $('formHeading').textContent = 'Edit snippet'; $('cancel').hidden = false; window.scrollTo({ top: 0, behavior: 'smooth' });
          };
          const del = document.createElement('button'); del.textContent = 'Delete'; del.className = 'secondary';
          del.onclick = async () => {
            if (!confirm(`Delete "${s.title}"?`)) return;
            try { await api(`/api/snippets/${s._id}`, { method: 'DELETE' }); statusLine.textContent = 'Deleted.'; await loadSnippets(); }
            catch (e) { statusLine.textContent = e.message; }
          };
          card.append(h, pre, edit, del); out.append(card);
        }
      } catch (e) { statusLine.textContent = e.message; }
    }

    form.onsubmit = async ev => {
      ev.preventDefault();
      const id = $('snippetId').value;
      const body = { title: $('title').value.trim(), language: $('language').value.trim(), code: $('code').value };
      try {
        await api(id ? `/api/snippets/${id}` : '/api/snippets', { method: id ? 'PUT' : 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) });
        statusLine.textContent = id ? 'Updated.' : 'Created.';
        clearForm(); await loadSnippets();
      } catch (e) { statusLine.textContent = e.message; }
    };
    $('cancel').onclick = clearForm;
    $('refresh').onclick = loadSnippets;
    $('search').oninput = loadSnippets;
    loadSnippets();
  </script>
</body>
</html>
```

### Run and test
```bash
node server.js
```
Open `http://localhost:5000`. Verify in Compass or `mongosh`:
```bash
mongosh "YOUR_MONGODB_URI"
> use codesnipper
> db.snippets.find().pretty()
> db.snippets.countDocuments()
```

### If it fails
- `bad auth`: use the Database Access user, URL-encode the password.
- `ENOTFOUND` / `ECONNREFUSED` / selection timeout: Atlas IP allowlist, internet, or local `mongod` not running (`sudo systemctl status mongod`).
- `MONGODB_URI missing`: `.env` must sit beside `server.js` and not be named `.env.txt`.
- `Failed to fetch`: open via `http://localhost:5000`, keep the server running.
- Atlas unreachable in the lab: switch the URI to the local one (Option B).

---

## 7) Experiment 6: Auth0 SSO (React SPA + protected Express API)

### Aim
Use Auth0 as the identity provider for a React app (login, profile, logout), call a protected API with an access token, and show SSO across two apps.

### Links
[Auth0 Dashboard](https://manage.auth0.com/) | [React quickstart](https://auth0.com/docs/quickstart/spa/react) | [Node API quickstart](https://auth0.com/docs/quickstart/backend/nodejs)

### Dashboard configuration
1. **Applications -> Create Application -> Single Page Web Applications** (name `SPA One`).
2. Copy **Domain** and **Client ID**. Never use a client secret in React.
3. **Settings** tab, set and **Save**:
   - Allowed Callback URLs: `http://localhost:5173`
   - Allowed Logout URLs: `http://localhost:5173`
   - Allowed Web Origins: `http://localhost:5173`
4. **Applications -> APIs -> Create API**: Identifier `https://api.local`, algorithm RS256.
5. Second app for SSO: create `SPA Two` with the same settings but `http://localhost:5174`.

### Create the React app
```bash
npm create vite@latest auth0-spa -- --template react
cd auth0-spa
npm install
npm install @auth0/auth0-react
```
(If Vite asks questions: framework **React**, variant **JavaScript**.)

`.env.local` in the project root (restart `npm run dev` after any change):
```env
VITE_AUTH0_DOMAIN=YOUR_TENANT.us.auth0.com
VITE_AUTH0_CLIENT_ID=YOUR_SPA_CLIENT_ID
VITE_AUTH0_AUDIENCE=https://api.local
```
Domain has no `https://`.

### `src/main.jsx` (replace)
```jsx
import ReactDOM from 'react-dom/client';
import App from './App';
import { Auth0Provider } from '@auth0/auth0-react';
import './style.css';

ReactDOM.createRoot(document.getElementById('root')).render(
  <Auth0Provider
    domain={import.meta.env.VITE_AUTH0_DOMAIN}
    clientId={import.meta.env.VITE_AUTH0_CLIENT_ID}
    authorizationParams={{
      redirect_uri: window.location.origin,
      audience: import.meta.env.VITE_AUTH0_AUDIENCE,
      scope: 'openid profile email'
    }}
  >
    <App />
  </Auth0Provider>
);
```

### `src/App.jsx` (replace)
```jsx
import { useState } from 'react';
import { useAuth0 } from '@auth0/auth0-react';

export default function App() {
  const { loginWithRedirect, logout, user, isAuthenticated, isLoading, error, getAccessTokenSilently } = useAuth0();
  const [result, setResult] = useState('');

  async function callApi() {
    try {
      const token = await getAccessTokenSilently();
      const res = await fetch('http://localhost:4000/api/profile', { headers: { Authorization: `Bearer ${token}` } });
      const data = await res.json();
      if (!res.ok) throw new Error(data.error || `API returned ${res.status}`);
      setResult(JSON.stringify(data, null, 2));
    } catch (e) { setResult(e.message); }
  }

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Auth error: {error.message}</p>;

  return (
    <main className="card">
      <h1>Auth0 SSO Lab</h1>
      {isAuthenticated ? (
        <>
          <img src={user.picture} alt="profile" width="64" height="64" style={{ borderRadius: '50%' }} />
          <h2>Welcome, {user.name}</h2>
          <p>{user.email}</p>
          <button onClick={callApi}>Call protected API</button>
          <button className="secondary" onClick={() => logout({ logoutParams: { returnTo: window.location.origin } })}>Sign out</button>
        </>
      ) : (
        <button onClick={() => loginWithRedirect()}>Sign in</button>
      )}
      <pre>{result}</pre>
    </main>
  );
}
```

### `src/style.css` (new)
```css
* { box-sizing: border-box; }
body { margin: 0; padding: 30px 16px; color: #242424; background: #fff; font: 16px/1.5 Arial, sans-serif; }
.card { max-width: 620px; margin: 30px auto; padding: 24px; border: 1px solid #ddd; border-radius: 8px; }
button { margin: 8px 8px 8px 0; padding: 9px 14px; color: #fff; background: #333; border: 0; border-radius: 4px; cursor: pointer; }
button.secondary { color: #222; background: #eee; }
pre { min-height: 28px; overflow-x: auto; padding: 12px; background: #f5f5f5; white-space: pre-wrap; }
```

### Protected API (separate folder)
```bash
mkdir api && cd api
npm init -y
npm install express cors dotenv express-oauth2-jwt-bearer
```
`api/.env`:
```env
AUTH0_DOMAIN=YOUR_TENANT.us.auth0.com
AUTH0_AUDIENCE=https://api.local
```
`api/server.js`:
```js
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const { auth } = require('express-oauth2-jwt-bearer');

const app = express();
const { AUTH0_DOMAIN, AUTH0_AUDIENCE } = process.env;
if (!AUTH0_DOMAIN || !AUTH0_AUDIENCE) throw new Error('Set AUTH0_DOMAIN and AUTH0_AUDIENCE in api/.env');

app.use(cors({ origin: ['http://localhost:5173', 'http://localhost:5174'] }));
const checkJwt = auth({ audience: AUTH0_AUDIENCE, issuerBaseURL: `https://${AUTH0_DOMAIN}/` });

app.get('/api/public', (req, res) => res.json({ message: 'Anyone can read this' }));
app.get('/api/profile', checkJwt, (req, res) =>
  res.json({ message: 'Access token verified', subject: req.auth.payload.sub }));

app.use((err, req, res, next) => res.status(err.status || 500).json({ error: err.message }));
app.listen(4000, () => console.log('Protected API at http://localhost:4000'));
```

### Run (three terminals)
```bash
# Terminal 1: API
cd auth0-spa/api && node server.js
# Terminal 2: SPA One
cd auth0-spa && npm run dev
# Terminal 3: SPA Two for the SSO demo (copy the folder, put SPA Two's client ID in its .env.local)
cp -r auth0-spa auth0-spa-two          # Windows: Copy-Item -Recurse auth0-spa auth0-spa-two
cd auth0-spa-two && npm run dev -- --port 5174
```

### SSO demo script
1. Open `http://localhost:5173` -> Sign in (Auth0 login page).
2. Open `http://localhost:5174` in the **same browser** -> Sign in. Auth0 should not ask for the password again (the tenant session is reused).
3. Sign out in one app, then show the other still has its own session until it logs out. Screenshot each step.

### If it fails
- `Callback URL mismatch`: add the exact URL (including port) to all three lists and Save.
- `Service not found: https://api.local`: audience in `.env.local` must equal the API identifier.
- API 401: sign out/in to get a fresh token; check both audience values.
- CORS: API `origin` must include the exact SPA URL.
- Blank page after `.env.local` edit: restart Vite.
- Auth0 needs internet; no offline fallback.

---

# Cloud Lab Cheat Sheet v2: Part 2 (Exp 8 to 12, Ubuntu VM)

Assumed numbering: **Exp 8A** OpenMP, **Exp 8B** MPI, **Exp 9** Hadoop 3.3.6, **Exp 10** Apache Spark, **Exp 11A** CloudSim 5.0 Setup, **Exp 11B** CloudSim FCFS Resource Allocation, **Exp 12** CloudAnalyst.

Target: Ubuntu 22.04 / 24.04 VM (also covers native Windows/WSL where relevant). Every block is copy-paste. Run in a normal terminal (Ctrl+Alt+T). Never paste shell commands into `scala>`, `>>>` or Eclipse.

---

## A) Ubuntu VM survival kit (read first)

### A1. Check the VM before you start

```bash
nproc        # CPU cores (need 2+ to see OpenMP/MPI speedup)
free -h      # RAM (Hadoop/Spark want 4 GB; run one thing at a time on 2 GB)
df -h /      # keep at least 10 GB free
ping -c 2 archive.ubuntu.com
```

VirtualBox: shut the VM down, then Settings -> System -> Processor = 2 to 4 CPUs, Motherboard -> RAM = 4096 MB or more.

### A2. "Could not get lock /var/lib/dpkg/lock-frontend ... held by process" (CRITICAL UBUNTU VM ISSUE)

**Why this happens:** When Ubuntu boots inside a VM, background services (`unattended-upgr`, `apt-daily.service`, `packagekitd`, or the desktop "Software Updater") immediately wake up and acquire exclusive locks on `/var/lib/dpkg/lock-frontend` and `/var/lib/apt/lists/lock`. If you run `sudo apt install` while this is happening, you will get locked out and error code 100.

#### 🔥 EMERGENCY 10-SECOND LOCK NUKER (Copy-paste this entire single line into the terminal):
```bash
sudo killall -9 apt apt-get dpkg unattended-upgrade packagekitd 2>/dev/null; sudo rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock /var/lib/apt/lists/lock; sudo dpkg --configure -a; sudo apt --fix-broken install -y; sudo apt update
```

#### 🛡️ PERMANENT PREVENTION (Run this once right after booting the VM so it NEVER locks again during your lab):
```bash
# 1. Stop background apt updaters completely
sudo systemctl stop unattended-upgrades packagekit apt-daily.service apt-daily-upgrade.service 2>/dev/null
sudo systemctl disable --now unattended-upgrades packagekit apt-daily.timer apt-daily-upgrade.timer 2>/dev/null

# 2. Disable automatic download/upgrade in config files
sudo sed -i 's/"1"/"0"/g' /etc/apt/apt.conf.d/20auto-upgrades 2>/dev/null || true
sudo sed -i 's/"1"/"0"/g' /etc/apt/apt.conf.d/10periodic 2>/dev/null || true

# 3. Always allow apt to wait up to 5 minutes rather than crashing instantly
echo 'APT::Acquire::Retries "3";' | sudo tee /etc/apt/apt.conf.d/80retries
echo 'DPkg::Lock::Timeout "300";' | sudo tee /etc/apt/apt.conf.d/80locktimeout
```

#### Step-by-Step Diagnostics (if you want to inspect manually):
```bash
# 1. See who is currently holding the lock
ps aux | grep -E 'apt|dpkg|unattended|packagekit' | grep -v grep
sudo fuser -v /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/lib/apt/lists/lock

# 2. Kill the process holding the lock by PID
sudo kill -9 <PID>

# 3. Remove stale lock files ONLY if no legitimate apt process is running
sudo rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock /var/lib/apt/lists/lock

# 4. Repair dpkg database
sudo dpkg --configure -a
sudo apt --fix-broken install -y
sudo apt update
```

#### Other apt/system corner failures in VM:
- `dpkg was interrupted, you must manually run 'sudo dpkg --configure -a'` -> run `sudo dpkg --configure -a` then `sudo apt update`.
- `Unable to locate package X` -> enable universe repository:
  ```bash
  sudo apt install -y software-properties-common && sudo add-apt-repository -y universe && sudo apt update
  ```
- `Temporary failure resolving 'archive.ubuntu.com'` -> VM has lost DNS/network.
  VirtualBox fix: VM Settings -> Network -> Adapter 1 -> Attached to: **NAT** (ensure "Cable Connected" is checked).
  Inside VM run:
  ```bash
  sudo systemctl restart systemd-resolved NetworkManager
  echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" | sudo tee /etc/resolv.conf
  ping -c 2 8.8.8.8
  ```
- `You have held broken packages` -> `sudo apt update && sudo apt --fix-broken install -y`.
- `No space left on device` -> VM virtual disk filled up:
  ```bash
  df -h /
  sudo apt clean
  sudo journalctl --vacuum-size=50M
  rm -rf ~/.cache/* /tmp/*
  ```
- `certificate ... not yet valid` / SSL errors -> VM clock skew after pausing/resuming VM:
  ```bash
  sudo timedatectl set-ntp true
  sudo systemctl restart systemd-timesyncd
  date    # verify correct date and time
  ```
- `snap ... has changes in progress` -> `snap changes`, then `sudo snap abort <ID>` and retry.
- `Permission denied` in `/opt` or `/usr/local` -> `sudo chown -R $USER:$USER <folder>`.
- Shell environment variables missing in new terminal -> always append to `~/.bashrc` and run `source ~/.bashrc`.
- Windows to Linux CRLF error (`\r: command not found` or syntax errors in scripts):
  ```bash
  sudo apt install -y dos2unix
  dos2unix <filename>
  # Or with sed:
  sed -i 's/\r$//' <filename>
  ```

### A3. Install the common tools once

```bash
sudo apt update
sudo apt install -y build-essential wget curl unzip tar git lsof nano openjdk-11-jdk
```

### A4. Switch Java without menus (use JDK 11 for everything in this part)

```bash
ls /usr/lib/jvm
sudo update-alternatives --set java  /usr/lib/jvm/java-11-openjdk-amd64/bin/java
sudo update-alternatives --set javac /usr/lib/jvm/java-11-openjdk-amd64/bin/javac
java -version && javac -version     # both must say 11.x
```

ARM VM (Apple silicon): replace `amd64` with `arm64` everywhere in this sheet. Check with `dpkg --print-architecture`.

---

## B) Exp 8A: OpenMP (shared-memory parallelism in C)

### Aim

Parallelise loops with OpenMP: parallel region, `parallel for` with `reduction`, race condition vs fix, `sections`, and measure speedup.

### Install and verify

```bash
sudo apt install -y build-essential
gcc --version
echo | gcc -fopenmp -dM -E - | grep -i _OPENMP     # prints a #define _OPENMP 2018xx line
```

### `omp_demo.c` (change only the CONFIG block)

```c
#include <stdio.h>
#include <math.h>
#include <omp.h>

// ================= CONFIG =================
#define N        50000000L   // loop size (raise it if speedup is not visible)
#define THREADS  4           // threads to use
#define ITER     200000      // iterations for the race-condition demo
// ==========================================

int main(void) {
    omp_set_num_threads(THREADS);
    printf("Cores: %d | Max threads: %d\n", omp_get_num_procs(), omp_get_max_threads());

    /* 1. Parallel region */
    #pragma omp parallel
    printf("Hello from thread %d of %d\n", omp_get_thread_num(), omp_get_num_threads());

    /* 2. Serial vs parallel for + reduction */
    double t0 = omp_get_wtime(), s1 = 0.0;
    for (long i = 1; i <= N; i++) s1 += sqrt((double)i);
    double t1 = omp_get_wtime(), s2 = 0.0;
    #pragma omp parallel for reduction(+:s2)
    for (long i = 1; i <= N; i++) s2 += sqrt((double)i);
    double t2 = omp_get_wtime();
    printf("Serial   : %.4f s  sum=%.2f\n", t1 - t0, s1);
    printf("Parallel : %.4f s  sum=%.2f\n", t2 - t1, s2);
    printf("Speedup  : %.2fx\n", (t1 - t0) / (t2 - t1));

    /* 3. Race condition vs fixes */
    long bad = 0, atom = 0, good = 0;
    #pragma omp parallel for
    for (int i = 0; i < ITER; i++) bad++;                 // WRONG: data race
    #pragma omp parallel for
    for (int i = 0; i < ITER; i++) {
        #pragma omp atomic
        atom++;                                           // correct, slower
    }
    #pragma omp parallel for reduction(+:good)
    for (int i = 0; i < ITER; i++) good++;                // correct, fastest
    printf("Expected %d | unsafe=%ld atomic=%ld reduction=%ld\n", ITER, bad, atom, good);

    /* 4. Sections (task parallelism) */
    #pragma omp parallel sections
    {
        #pragma omp section
        printf("Section A on thread %d\n", omp_get_thread_num());
        #pragma omp section
        printf("Section B on thread %d\n", omp_get_thread_num());
    }
    return 0;
}
```

### Compile and run

```bash
gcc -fopenmp -O2 omp_demo.c -o omp_demo -lm
./omp_demo
```

The flags must be exactly in this order (`-lm` last). Expected: threads print in random order (normal), parallel time lower than serial, `unsafe` count usually below 200000, `atomic` and `reduction` exactly 200000.

Speedup table for the record (tries 1, 2, 4, 8 threads):

```bash
for t in 1 2 4 8; do
  sed -i "s/#define THREADS .*/#define THREADS  $t/" omp_demo.c
  gcc -fopenmp -O2 omp_demo.c -o omp_demo -lm && echo "== $t threads" && ./omp_demo | grep -E 'Serial|Parallel|Speedup'
done
```

### If it fails

- `fatal error: omp.h: No such file or directory` -> `sudo apt install -y build-essential`, check `gcc --version`.
- `undefined reference to GOMP_parallel` or `omp_get_thread_num` -> `-fopenmp` missing at compile/link.
- `undefined reference to sqrt` -> put `-lm` after the source file.
- Every line says "thread 0 of 1" -> `-fopenmp` missing (pragmas are silently ignored).
- Speedup about 1x or below 1x -> `nproc` shows 1 core (fix VM CPUs, see A1), or N too small (raise N).
- Setting `OMP_NUM_THREADS=2` has no effect -> the code calls `omp_set_num_threads(THREADS)`, which overrides it. Change `THREADS` instead.
- `Permission denied` when running -> `chmod +x omp_demo`; `command not found` -> use `./omp_demo`.
- Strange characters/compile errors after copying from Windows -> `sed -i 's/\r$//' omp_demo.c`.

---

## C) Exp 8B: MPI (message passing, OpenMPI)

### Aim

Run several processes that communicate: rank/size, Send/Recv, Bcast, Scatter, Reduce, and a parallel pi calculation.

### Install and verify

```bash
sudo apt install -y openmpi-bin libopenmpi-dev
mpicc --version
mpirun --version       # Open MPI 4.x
```

### `mpi_demo.c`

```c
#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

// ================= CONFIG =================
#define PER_PROC 4            // elements per process in Scatter
#define STEPS    100000000L   // intervals for the pi integration
// ==========================================

int main(int argc, char **argv) {
    int rank, size, len;
    char host[MPI_MAX_PROCESSOR_NAME];
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Get_processor_name(host, &len);
    printf("Hello from rank %d of %d on %s\n", rank, size, host);
    MPI_Barrier(MPI_COMM_WORLD);

    /* 1. Point-to-point */
    if (size >= 2) {
        if (rank == 0) {
            int msg = 100;
            MPI_Send(&msg, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);
            printf("Rank 0 sent %d to rank 1\n", msg);
        } else if (rank == 1) {
            int msg;
            MPI_Recv(&msg, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            printf("Rank 1 received %d from rank 0\n", msg);
        }
    }

    /* 2. Broadcast */
    int value = (rank == 0) ? 42 : 0;
    MPI_Bcast(&value, 1, MPI_INT, 0, MPI_COMM_WORLD);
    printf("Rank %d has broadcast value %d\n", rank, value);

    /* 3. Scatter + Reduce */
    int *all = NULL, local[PER_PROC], part = 0, total = 0;
    if (rank == 0) {
        all = malloc(size * PER_PROC * sizeof(int));
        for (int i = 0; i < size * PER_PROC; i++) all[i] = i + 1;
    }
    MPI_Scatter(all, PER_PROC, MPI_INT, local, PER_PROC, MPI_INT, 0, MPI_COMM_WORLD);
    for (int i = 0; i < PER_PROC; i++) part += local[i];
    MPI_Reduce(&part, &total, 1, MPI_INT, MPI_SUM, 0, MPI_COMM_WORLD);
    if (rank == 0) {
        int n = size * PER_PROC;
        printf("Sum 1..%d = %d (expected %d)\n", n, total, n * (n + 1) / 2);
        free(all);
    }

    /* 4. Pi by numerical integration */
    double t0 = MPI_Wtime(), h = 1.0 / (double)STEPS, local_sum = 0.0, pi = 0.0;
    for (long i = rank; i < STEPS; i += size) {
        double x = h * ((double)i + 0.5);
        local_sum += 4.0 / (1.0 + x * x);
    }
    local_sum *= h;
    MPI_Reduce(&local_sum, &pi, 1, MPI_DOUBLE, MPI_SUM, 0, MPI_COMM_WORLD);
    if (rank == 0) printf("Pi = %.12f  time = %.3f s with %d processes\n", pi, MPI_Wtime() - t0, size);

    MPI_Finalize();
    return 0;
}
```

### Compile and run

```bash
mpicc mpi_demo.c -o mpi_demo -lm
mpirun -np 4 ./mpi_demo
# more processes than cores (needed on a 2-core VM for -np 4 or more):
mpirun --oversubscribe -np 8 ./mpi_demo
# compare timings for the record
for p in 1 2 4; do mpirun --oversubscribe -np $p ./mpi_demo | grep Pi; done
```

### If it fails

- `mpicc: command not found` -> `sudo apt install -y openmpi-bin libopenmpi-dev`, then open a new terminal.
- `There are not enough slots available in the system` -> add `--oversubscribe`, or lower `-np` to the `nproc` value.
- `mpirun has detected an attempt to run as root` -> you are in `sudo -i`/root shell. Use your normal user. Last resort: `mpirun --allow-run-as-root -np 4 ./mpi_demo`.
- `mpicc` works but `mpirun` errors / version mismatch (MPICH and OpenMPI both installed) -> `sudo update-alternatives --config mpirun` and `--config mpicc`, pick the same MPI for both.
- `error while loading shared libraries: libmpi.so` -> `sudo ldconfig`, or reinstall `libopenmpi-dev`.
- Warnings about `vader`, `CMA`, `Read -1, expected` -> `export OMPI_MCA_btl_vader_single_copy_mechanism=none` then rerun.
- Hangs forever -> a Send without a matching Recv (rank numbers), or you ran with `-np 1` while the code needs 2 ranks. Press Ctrl+C.
- Network/interface warnings and slow start in a VM -> `mpirun --mca btl_tcp_if_include lo -np 4 ./mpi_demo`.
- Output lines mixed up or in random order -> normal for MPI (processes print independently).
- Segmentation fault in Scatter -> `PER_PROC` and the receive buffer size must match.

---

## D) Exp 9: Hadoop 3.3.6 (single node, HDFS + YARN + WordCount)

### Aim

Set up a pseudo-distributed Hadoop cluster on one Ubuntu VM, store files in HDFS and run a MapReduce WordCount.

### D1. Java and SSH

```bash
sudo apt update
sudo apt install -y openjdk-11-jdk openssh-server openssh-client wget tar
sudo systemctl enable --now ssh
systemctl status ssh --no-pager | head -n 5      # must show active (running)
```

Do NOT install `pdsh` (it breaks start-dfs.sh).

### D2. Passwordless SSH to localhost

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -P "" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ssh-keyscan -H localhost 0.0.0.0 127.0.0.1 >> ~/.ssh/known_hosts 2>/dev/null
ssh localhost 'echo SSH OK'       # must print SSH OK without asking for a password
```

### D3. Download and place Hadoop

```bash
cd ~
wget -c https://archive.apache.org/dist/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -xzf hadoop-3.3.6.tar.gz
sudo mv hadoop-3.3.6 /usr/local/hadoop
sudo chown -R $USER:$USER /usr/local/hadoop
```

`gzip: stdin: unexpected end of file` means a partial download: `rm hadoop-3.3.6.tar.gz` and run `wget -c` again (or download the same URL in Firefox).

### D4. Environment (run this block ONCE; running twice duplicates lines)

```bash
cat >> ~/.bashrc <<'EOF'
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export PDSH_RCMD_TYPE=ssh
EOF
source ~/.bashrc

cat >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh <<EOF
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-$(dpkg --print-architecture)
export HADOOP_SSH_OPTS="-o StrictHostKeyChecking=no"
EOF

ls $JAVA_HOME 2>/dev/null; ls /usr/lib/jvm      # confirm the java-11 folder name matches
hadoop version                                  # first line: Hadoop 3.3.6
```

JAVA\_HOME is set inside `hadoop-env.sh` on purpose: the daemons are started over SSH and do not read `.bashrc`.

### D5. Configuration files (copy the whole block)

```bash
mkdir -p ~/hadoopdata/namenode ~/hadoopdata/datanode ~/hadoopdata/tmp
cd $HADOOP_HOME/etc/hadoop

cat > core-site.xml <<EOF
<?xml version="1.0"?>
<configuration>
  <property><name>fs.defaultFS</name><value>hdfs://localhost:9000</value></property>
  <property><name>hadoop.tmp.dir</name><value>$HOME/hadoopdata/tmp</value></property>
</configuration>
EOF

cat > hdfs-site.xml <<EOF
<?xml version="1.0"?>
<configuration>
  <property><name>dfs.replication</name><value>1</value></property>
  <property><name>dfs.namenode.name.dir</name><value>file:$HOME/hadoopdata/namenode</value></property>
  <property><name>dfs.datanode.data.dir</name><value>file:$HOME/hadoopdata/datanode</value></property>
</configuration>
EOF

cat > mapred-site.xml <<'EOF'
<?xml version="1.0"?>
<configuration>
  <property><name>mapreduce.framework.name</name><value>yarn</value></property>
  <property><name>mapreduce.application.classpath</name><value>/usr/local/hadoop/share/hadoop/mapreduce/*:/usr/local/hadoop/share/hadoop/mapreduce/lib/*</value></property>
  <property><name>yarn.app.mapreduce.am.env</name><value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value></property>
  <property><name>mapreduce.map.env</name><value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value></property>
  <property><name>mapreduce.reduce.env</name><value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value></property>
</configuration>
EOF

cat > yarn-site.xml <<'EOF'
<?xml version="1.0"?>
<configuration>
  <property><name>yarn.nodemanager.aux-services</name><value>mapreduce_shuffle</value></property>
  <property><name>yarn.nodemanager.env-whitelist</name><value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value></property>
  <property><name>yarn.nodemanager.vmem-check-enabled</name><value>false</value></property>
</configuration>
EOF

cd ~
```

### D6. Format (ONLY the first time) and start

```bash
hdfs namenode -format          # look for: has been successfully formatted
start-dfs.sh
start-yarn.sh
jps
```

`jps` must list: **NameNode, DataNode, SecondaryNameNode, ResourceManager, NodeManager** (plus Jps). Web pages (Firefox in the VM): http://localhost:9870 (HDFS) and http://localhost:8088 (YARN). After every VM reboot just run `start-dfs.sh` and `start-yarn.sh` again. NEVER format again unless something is broken (it deletes your HDFS data).

### D7. HDFS commands and WordCount (example jar)

```bash
mkdir -p ~/wc_input
printf 'hello cloud hello hadoop\nbig data in the cloud\n' > ~/wc_input/a.txt

hdfs dfs -mkdir -p /user/$USER
hdfs dfs -put -f ~/wc_input /user/$USER/input
hdfs dfs -ls /user/$USER/input

hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar wordcount /user/$USER/input /user/$USER/output
hdfs dfs -cat /user/$USER/output/part-r-00000
```

Re-running needs a fresh output folder: `hdfs dfs -rm -r /user/$USER/output`. Other handy commands: `hdfs dfs -get`, `-cat`, `-rm`, `-du -h`, `hdfs dfsadmin -report`.

### D8. Your own WordCount.java (if the question asks you to write the code)

```java
import java.io.IOException;
import java.util.StringTokenizer;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {
  public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();
    public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString().toLowerCase().replaceAll("[^a-z0-9\\s]", " "));
      while (itr.hasMoreTokens()) { word.set(itr.nextToken()); context.write(word, one); }
    }
  }
  public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    private IntWritable result = new IntWritable();
    public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable v : values) sum += v.get();
      result.set(sum);
      context.write(key, result);
    }
  }
  public static void main(String[] args) throws Exception {
    Job job = Job.getInstance(new Configuration(), "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

```bash
mkdir -p wc_classes
javac -classpath "$(hadoop classpath)" -d wc_classes WordCount.java
jar -cvf wc.jar -C wc_classes/ .
hadoop jar wc.jar WordCount /user/$USER/input /user/$USER/output2
hdfs dfs -cat /user/$USER/output2/part-r-00000
```

### D9. Stop, and a no-daemon fallback

```bash
stop-yarn.sh
stop-dfs.sh
```

If the cluster refuses to start and time is short, run WordCount locally (no HDFS/YARN needed):

```bash
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar wordcount -D mapreduce.framework.name=local -D fs.defaultFS=file:/// ~/wc_input ~/wc_output_local
cat ~/wc_output_local/part-r-00000
```

### If it fails (Hadoop)

- `ssh: connect to host localhost port 22: Connection refused` -> `sudo systemctl enable --now ssh`.
- `Permission denied (publickey)` -> `chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys; chmod go-w ~`, then redo D2.
- `pdsh@...: rcmd: socket: Permission denied` -> `export PDSH_RCMD_TYPE=ssh` (or `sudo apt remove -y pdsh`), then `start-dfs.sh`.
- `Are you sure you want to continue connecting` loops/prompts at start-dfs -> redo the `ssh-keyscan` line in D2.
- `Error: JAVA_HOME is not set` / `does not exist` -> `nano $HADOOP_HOME/etc/hadoop/hadoop-env.sh`, fix the `JAVA_HOME=` line to a folder shown by `ls /usr/lib/jvm`.
- `hadoop: command not found` -> `source ~/.bashrc`; check `echo $HADOOP_HOME`.
- `jps: command not found` -> you have only the JRE: `sudo apt install -y openjdk-11-jdk`.
- NameNode missing in `jps` -> `tail -n 40 $HADOOP_HOME/logs/hadoop-*-namenode-*.log`. Usually not formatted (run `hdfs namenode -format`) or port 9000 busy.
- DataNode missing, log says `Incompatible clusterIDs` (happens after a second format) -> clean restart:

```bash
stop-yarn.sh; stop-dfs.sh
rm -rf ~/hadoopdata/namenode/* ~/hadoopdata/datanode/* ~/hadoopdata/tmp/*
hdfs namenode -format
start-dfs.sh; start-yarn.sh; jps
```

- `Address already in use` / old daemons still alive -> `jps | grep -E 'NameNode|DataNode|SecondaryNameNode|ResourceManager|NodeManager' | awk '{print $1}' | xargs -r kill -9`, then start again. For a foreign process: `sudo lsof -i :9000` and kill that PID.
- `Name node is in safe mode` -> wait 30 s after start, else `hdfs dfsadmin -safemode leave`.
- `Connection refused ... localhost:9000` -> NameNode is down: `start-dfs.sh`, check `jps`.
- `FileAlreadyExistsException: Output directory ... already exists` -> `hdfs dfs -rm -r <output path>`.
- `Input path does not exist` -> `hdfs dfs -ls /user/$USER/`; redo the `-put` (the folder is `/user/$USER/input`, not your Linux path).
- Job sits at `ACCEPTED` or `Container ... is running beyond virtual memory limits` -> `jps` must show NodeManager; free RAM (close Firefox/Eclipse); the vmem line in `yarn-site.xml` is already false; restart YARN.
- `Could not find or load main class org.apache.hadoop.mapreduce.v2.app.MRAppMaster` -> `mapred-site.xml` classpath/env lines missing or mistyped; rewrite it from D5, then `stop-yarn.sh; start-yarn.sh`.
- `Premature end of file` / `Unexpected close tag` -> a broken XML file; rewrite that file from D5.
- Warning `Unable to load native-hadoop library` -> harmless, ignore.
- Errors mentioning `InaccessibleObjectException` or `java.lang.reflect` -> you are on Java 17+; switch to 11 (A4).
- `wget` cannot resolve host -> VM network (A2). Retry with `wget -c`.

---

## E) Exp 10: Apache Spark 3.5.x on Ubuntu (RDD WordCount, Local & HDFS Modes, PySpark)

### Aim
Install and configure Apache Spark on Ubuntu VM, verify environment variables, launch interactive Spark Shell, and execute an RDD-based Word Count processing job reading from both local file storage and Hadoop HDFS.

### E1. Prerequisites & Java Verification
Spark 3.5.x runs on Java 11 (recommended) or Java 17. Verify Java from Section A4:
```bash
java -version    # Must show OpenJDK 11 or 17
```

### E2. Download and Place Apache Spark
Download pre-built Spark distribution for Apache Hadoop 3:
```bash
cd ~/Downloads
# Download Spark (using archive mirror to avoid 404 on older builds):
wget -c https://archive.apache.org/dist/spark/spark-3.5.1/spark-3.5.1-bin-hadoop3.tgz || \
wget -c https://dlcdn.apache.org/spark/spark-3.5.2/spark-3.5.2-bin-hadoop3.tgz

# Extract to /usr/local/spark
tar -xzf spark-3.5.*-bin-hadoop3.tgz
sudo rm -rf /usr/local/spark
sudo mv spark-3.5.*-bin-hadoop3 /usr/local/spark
sudo chown -R "$USER":"$USER" /usr/local/spark
```

### E3. Configure Spark Environment Variables
Append Spark paths to your `~/.bashrc`:
```bash
cat << 'EOF' >> ~/.bashrc

# Apache Spark Configuration
export SPARK_HOME=/usr/local/spark
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin
export PYSPARK_PYTHON=python3
EOF

source ~/.bashrc

# Verify installation
spark-shell --version
```

### E4. Local RDD WordCount in Scala (spark-shell)

#### Step 1: Create a test input file
```bash
mkdir -p ~/spark_lab && cd ~/spark_lab
printf "cloud computing lab spark hadoop\nspark rdd parallel processing cloud\ncloud virtualization hadoop\n" > input.txt
cat input.txt
```

#### Step 2: Launch `spark-shell`
```bash
spark-shell --master "local[*]"
```
*(Wait until the `scala>` prompt appears. Web UI is available at `http://localhost:4040` in VM browser).*

#### Step 3: Run the Word Count RDD Transformations & Actions
Type the following Scala commands directly into the `scala>` REPL:
```scala
// 1. Read input text file into an RDD of Strings
val lines = sc.textFile("input.txt")

// 2. Split lines by whitespace into words (Transformation: flatMap)
val words = lines.flatMap(line => line.split("\\s+"))

// 3. Map each word to a key-value tuple (word, 1) (Transformation: map)
val pairs = words.map(word => (word.toLowerCase.replaceAll("[^a-z0-9]", ""), 1)).filter(_._1.nonEmpty)

// 4. Aggregate frequencies by key (Transformation: reduceByKey)
val counts = pairs.reduceByKey(_ + _)

// 5. Trigger execution and print word count results (Action: collect & foreach)
counts.collect().foreach(println)

// 6. Save word count results to disk (Action: saveAsTextFile)
counts.saveAsTextFile("spark_output")

// Exit spark-shell
:quit
```

#### Step 4: Verify output on Linux disk
```bash
cat spark_output/part-*
```
Expected output:
```text
(lab,1)
(processing,1)
(computing,1)
(virtualization,1)
(hadoop,2)
(cloud,3)
(spark,2)
(parallel,1)
(rdd,1)
```

---

### E5. Running Spark WordCount over Hadoop HDFS
This connects your Spark processing engine to your Hadoop distributed file system (Exp 9).

#### Step 1: Ensure Hadoop HDFS is running
```bash
jps    # Check if NameNode and DataNode are active
# If not running, start HDFS:
start-dfs.sh
```

#### Step 2: Upload file to HDFS
```bash
hdfs dfs -mkdir -p /user/$USER/spark_input
hdfs dfs -put -f ~/spark_lab/input.txt /user/$USER/spark_input/
hdfs dfs -ls /user/$USER/spark_input
```

#### Step 3: Execute Spark WordCount reading from and writing to HDFS
Launch `spark-shell`:
```bash
spark-shell --master "local[*]"
```
Inside `scala>`:
```scala
val user = System.getProperty("user.name")
val hdfsLines = sc.textFile("hdfs://localhost:9000/user/" + user + "/spark_input/input.txt")
val hdfsCounts = hdfsLines.flatMap(_.split("\\s+")).map((_, 1)).reduceByKey(_ + _)
hdfsCounts.collect().foreach(println)
hdfsCounts.saveAsTextFile("hdfs://localhost:9000/user/" + user + "/spark_hdfs_out")
:quit
```

#### Step 4: Verify HDFS output
```bash
hdfs dfs -cat /user/$USER/spark_hdfs_out/part-*
```

---

### E6. Zero-Scala Fallback: PySpark (Python WordCount)
If the lab exam allows Python or you prefer avoiding Scala:
```bash
pyspark --master "local[*]"
```
Inside `>>>`:
```python
rdd = sc.textFile("input.txt")
counts = rdd.flatMap(lambda line: line.split()) \
            .map(lambda word: (word.lower(), 1)) \
            .reduceByKey(lambda a, b: a + b)
for word, count in counts.collect():
    print(f"{word}: {count}")
counts.saveAsTextFile("pyspark_output")
exit()
```

---

### E7. Spark Corner Cases & Failure Troubleshooting Matrix

| Error Message / Symptom | Root Cause | Immediate Working Fix |
|---|---|---|
| `org.apache.hadoop.mapred.FileAlreadyExistsException: Output directory ... already exists` | Spark strictly prohibits overwriting existing output directories. | Remove the output folder before rerunning: <br>`rm -rf spark_output` (local) or <br>`hdfs dfs -rm -r /user/$USER/spark_hdfs_out` (HDFS). |
| `java.lang.reflect.InaccessibleObjectException: Unable to make field private final java.lang.String ... accessible` | Java 17+ strong encapsulation breaks older Spark reflection. | **Fix 1 (Recommended):** Use Java 11: `sudo update-alternatives --set java /usr/lib/jvm/java-11-openjdk-amd64/bin/java`. <br>**Fix 2:** Export JVM module flags in `~/.bashrc`: <br>`export SPARK_SUBMIT_OPTS="--add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.invoke=ALL-UNNAMED --add-opens java.base/java.lang.reflect=ALL-UNNAMED --add-opens java.base/java.io=ALL-UNNAMED --add-opens java.base/java.net=ALL-UNNAMED --add-opens java.base/java.nio=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/sun.nio.ch=ALL-UNNAMED"` |
| `java.lang.OutOfMemoryError: Java heap space` or VM freezes on `spark-shell` start | Spark defaults to 1 GB heap which exhausts low-RAM VMs. | Start with restricted memory allocation: <br>`spark-shell --driver-memory 512M --executor-memory 512M --master "local[2]"` |
| `WARN Utils: Service 'SparkUI' could not bind on port 4040. Attempting port 4041.` | Another Spark process is already running or didn't exit cleanly. | Not fatal; Spark automatically picks 4041. To free 4040: <br>`lsof -ti :4040 \| xargs kill -9 2>/dev/null` |
| `java.net.ConnectException: Call From ... to localhost:9000 failed on connection exception` | Hadoop HDFS NameNode daemon is offline. | Run `start-dfs.sh`, verify `jps` shows `NameNode`, then rerun Spark job. |
| `input.txt does not exist` | Spark looking in working directory where file is missing. | Start `spark-shell` from `~/spark_lab`, or provide absolute path: <br>`sc.textFile("/home/student/spark_lab/input.txt")`. |
| `404 Not Found` when running `wget` for Spark | Apache periodically archives older minor releases to archive.apache.org. | Always use the archive link: <br>`https://archive.apache.org/dist/spark/spark-3.5.1/spark-3.5.1-bin-hadoop3.tgz`. |

---

## F) Exp 11A: CloudSim 5.0 Simulation Setup in Eclipse (Ubuntu Desktop & Windows)

### Aim
Set up and configure the CloudSim 5.0 simulation toolkit inside Eclipse IDE, configure the required Apache Commons Math external libraries on the Java Build Path, and successfully execute baseline cloud computing simulation `CloudSimExample1`.

---

### F1. Download and Extract CloudSim Toolkit
Run these commands in Ubuntu terminal:
```bash
mkdir -p ~/cloudsim_setup && cd ~/cloudsim_setup

# Download CloudSim 5.0 release
wget -c https://github.com/Cloudslab/cloudsim/archive/refs/tags/cloudsim-5.0.tar.gz || \
wget -c https://github.com/Cloudslab/cloudsim/releases/download/cloudsim-5.0/cloudsim-5.0.tar.gz
tar -xzf cloudsim-5.0.tar.gz

# Download Apache Commons Math 3.6.1 JAR (Mandatory dependency)
wget -c https://repo1.maven.org/maven2/org/apache/commons/commons-math3/3.6.1/commons-math3-3.6.1.jar

# Check downloaded files
ls -l ~/cloudsim_setup
```

*(Windows alternative: Download the zip files from GitHub and extract them to `C:\cloudsim-5.0`).*

---

### F2. Step-by-Step Eclipse GUI Navigation Guide

#### Step 1: Launch Eclipse IDE
```bash
eclipse &
```
Select your workspace (e.g. `/home/student/eclipse-workspace`) and click **Launch**.
Close the **Welcome** screen tab if it appears.

#### Step 2: Import CloudSim Project into Eclipse Workspace
1. In the top menu bar, click: **File -> Import...**
2. In the Import dialog box, expand folder **General** and select **Existing Projects into Workspace**. Click **Next >**.
3. In the "Select root directory" field, click **Browse...**.
4. Navigate to and select the extracted folder: `~/cloudsim_setup/cloudsim-5.0` (or `cloudsim-cloudsim-5.0`). Click **Open**.
5. In the Projects box, verify that the project is checked.
6. Click **Finish**.

> **Fallback if Eclipse doesn't recognize as existing project:**
> 1. Click **File -> New -> Java Project**.
> 2. Project name: `CloudSim`.
> 3. Uncheck **Use default location**, click **Browse...**, and select `~/cloudsim_setup/cloudsim-5.0`.
> 4. In JRE group: Select **JavaSE-11** (or JavaSE-1.8).
> 5. Click **Finish**. If asked "Create module-info.java?", click **Don't Create**!

#### Step 3: Configure Apache Commons Math External JAR in Build Path
1. In the **Package Explorer** pane on the left, right-click the `CloudSim` project.
2. Select **Build Path -> Configure Build Path...** (or right-click -> **Properties** -> **Java Build Path**).
3. In the Java Build Path window, click on the **Libraries** tab.
4. *Important (in Eclipse 2020+):* Click to highlight **Classpath** (not Modulepath).
5. On the right-hand side, click the button **Add External JARs...**.
6. Browse to `~/cloudsim_setup/commons-math3-3.6.1.jar`. Select it and click **Open**.
7. If your CloudSim folder contains a `jars/` or `lib/` directory, click **Add External JARs...** again, select all `.jar` files inside `jars/`, and click **Open**.
8. Click **Apply and Close**.

#### Step 4: Verify Project Health
- Look at the `CloudSim` project icon in Package Explorer. There must be **NO RED EXCLAMATION MARK (!) or RED ERROR (X)**.
- Expand `CloudSim` -> verify that **Referenced Libraries** lists `commons-math3-3.6.1.jar`.

#### Step 5: Execute Baseline CloudSim Simulation
1. In Package Explorer, expand:
   `CloudSim` -> `sources` (or `src` / `modules/cloudsim-examples/src/main/java`)
2. Expand the package: `org.cloudbus.cloudsim.examples`.
3. Locate `CloudSimExample1.java`.
4. Right-click `CloudSimExample1.java` -> select **Run As -> 1 Java Application**.

#### Step 6: Verify Console Output
In the **Console** view at the bottom of Eclipse, verify the simulation trace:
```text
Starting CloudSimExample1...
Initialising...
Starting CloudSim version 5.0
Datacenter_0 is starting...
Broker is starting...
Entities started.
0.0: Broker: CloudSimExample1: Cloudlet 0 received by Datacenter_0
280.1: Broker: CloudSimExample1: Cloudlet 0 returned to Broker
Broker is shutting down...
Simulation: No more future events
CloudInformationService: Notify all entities to shut down.
Datacenter_0 is shutting down...
Broker is shutting down...
Simulation completed.
Simulation completed.

========== OUTPUT ==========
Cloudlet ID    STATUS    Data center ID    VM ID    Time    Start Time    Finish Time
    0         SUCCESS          2            0       280         0.1          280.1
CloudSimExample1 finished!
```

---

### F3. Headless Terminal Fallback (If Eclipse Freezes or GUI Crashes)
If Eclipse is unavailable or freezes during the exam, you can compile and execute CloudSim directly in the Ubuntu terminal to obtain full marks:
```bash
cd ~/cloudsim_setup
# Compile using standalone javac with classpath
javac -cp ".:commons-math3-3.6.1.jar:$(find . -name '*.jar' | tr '\n' ':')" -d ./bin $(find . -name "*.java") 2>/dev/null || true

# Run CloudSimExample1
java -cp ".:bin:commons-math3-3.6.1.jar:$(find . -name '*.jar' | tr '\n' ':')" org.cloudbus.cloudsim.examples.CloudSimExample1
```

---

### F4. CloudSim Corner Cases & Failure Troubleshooting Matrix

| Error Message / Symptom | Root Cause | Immediate Working Fix |
|---|---|---|
| `java.lang.NoClassDefFoundError: org/apache/commons/math3/...` | Commons Math library is missing from Eclipse Build Path. | Right-click project -> **Build Path -> Configure Build Path -> Libraries -> Classpath -> Add External JARs...** -> select `commons-math3-3.6.1.jar` -> Apply and Close. |
| `module-info.java: error: ... does not open org.cloudbus.cloudsim to ...` | Eclipse generated a Java 9+ `module-info.java` file which enforces strict module encapsulation. | Locate `module-info.java` inside `src` or `sources`, **right-click and Delete it**. CloudSim requires flat classpath. |
| Red exclamation mark on Project icon: `Unbound classpath container` | Project points to a JDK version that is not installed on the system. | Right-click project -> **Properties -> Java Build Path -> Libraries** -> select JRE System Library -> click **Edit...** -> select **Execution environment: JavaSE-11** (or Installed JRE) -> Finish. |
| `Source option 5 is no longer supported. Use 7 or later.` or compiler mismatch | Eclipse Java compiler compliance level is set to an unsupported legacy version. | Right-click project -> **Properties -> Java Compiler** -> check "Enable project specific settings" -> change **Compiler compliance level** to `1.8` or `11` -> Click Apply and rebuild. |
| `The declared package "org.cloudbus.cloudsim.examples" does not match the expected package ""` | The parent folder was not marked as a Java Source Folder. | In Package Explorer, right-click the `sources` (or `src`) folder -> **Build Path -> Use as Source Folder**. |

---

## G) Exp 11B: CloudSim Custom Policy - First Come First Serve (FCFS) Resource Allocation

### Aim
Implement a First-Come First-Served (FCFS) resource allocation and cloudlet scheduling policy in CloudSim. Demonstrate sequential execution of multiple cloudlets with varying execution lengths and verify that tasks are executed strictly in order of arrival without preemption.

### G1. Understanding FCFS Policy in CloudSim
- **FCFS Scheduling Architecture:** CloudSim implements non-preemptive FCFS cloudlet scheduling using `CloudletSchedulerSpaceShared`.
- In Space-Shared scheduling, when multiple cloudlets arrive at a Virtual Machine (VM), the VM assigns its processing cores to Cloudlet 1. Subsequent cloudlets wait in a Waiting Queue until the active cloudlet finishes execution.
- In contrast, `CloudletSchedulerTimeShared` slices CPU time concurrently (Round Robin), meaning all cloudlets finish simultaneously. For FCFS, we specifically configure `CloudletSchedulerSpaceShared`.

---

### G2. Complete Working Source Code: `ResourceAllocFCFS.java`
Create this class inside the `org.cloudbus.cloudsim.examples` package in your Eclipse project:

```java
package org.cloudbus.cloudsim.examples;

import java.text.DecimalFormat;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.LinkedList;
import java.util.List;

import org.cloudbus.cloudsim.Cloudlet;
import org.cloudbus.cloudsim.CloudletSchedulerSpaceShared;
import org.cloudbus.cloudsim.Datacenter;
import org.cloudbus.cloudsim.DatacenterBroker;
import org.cloudbus.cloudsim.DatacenterCharacteristics;
import org.cloudbus.cloudsim.Host;
import org.cloudbus.cloudsim.Log;
import org.cloudbus.cloudsim.Pe;
import org.cloudbus.cloudsim.Storage;
import org.cloudbus.cloudsim.UtilizationModel;
import org.cloudbus.cloudsim.UtilizationModelFull;
import org.cloudbus.cloudsim.Vm;
import org.cloudbus.cloudsim.VmAllocationPolicySimple;
import org.cloudbus.cloudsim.VmSchedulerTimeShared;
import org.cloudbus.cloudsim.core.CloudSim;
import org.cloudbus.cloudsim.provisioners.BwProvisionerSimple;
import org.cloudbus.cloudsim.provisioners.PeProvisionerSimple;
import org.cloudbus.cloudsim.provisioners.RamProvisionerSimple;

/**
 * Cloud Computing Lab: Experiment 11B
 * First-Come-First-Serve (FCFS) Cloudlet Resource Allocation Policy Demonstration.
 */
public class ResourceAllocFCFS {

    private static List<Cloudlet> cloudletList;
    private static List<Vm> vmlist;

    public static void main(String[] args) {
        Log.printLine("========== Starting CloudSim FCFS Resource Allocation Simulation ==========");

        try {
            // Step 1: Initialize CloudSim package
            int num_user = 1;
            Calendar calendar = Calendar.getInstance();
            boolean trace_flag = false;
            CloudSim.init(num_user, calendar, trace_flag);

            // Step 2: Create Datacenter with 1 Host
            Datacenter datacenter0 = createDatacenter("Datacenter_0");

            // Step 3: Create Broker
            DatacenterBroker broker = createBroker();
            int brokerId = broker.getId();

            // Step 4: Create 1 Virtual Machine (VM) with SpaceShared scheduler (FCFS for Cloudlets)
            vmlist = new ArrayList<Vm>();
            int vmid = 0;
            int mips = 250;           // Processing speed: 250 MIPS
            long size = 10000;        // Image size (MB)
            int ram = 2048;           // VM memory (MB)
            long bw = 1000;           // Bandwidth
            int pesNumber = 1;        // Single core ensures strict FCFS queueing
            String vmm = "Xen";

            // CloudletSchedulerSpaceShared enforces First-Come-First-Served task queueing
            Vm vm1 = new Vm(vmid, brokerId, mips, pesNumber, ram, bw, size, vmm, new CloudletSchedulerSpaceShared());
            vmlist.add(vm1);
            broker.submitVmList(vmlist);

            // Step 5: Create 4 Cloudlets with different lengths arriving in sequential order
            cloudletList = new ArrayList<Cloudlet>();
            UtilizationModel utilizationModel = new UtilizationModelFull();

            // Cloudlet parameters: length (MI), file size, output size
            long[] cloudletLengths = { 5000, 10000, 15000, 20000 };

            for (int id = 0; id < 4; id++) {
                Cloudlet cloudlet = new Cloudlet(
                    id,
                    cloudletLengths[id],
                    pesNumber,
                    300,
                    300,
                    utilizationModel,
                    utilizationModel,
                    utilizationModel
                );
                cloudlet.setUserId(brokerId);
                cloudlet.setVmId(vmid);
                cloudletList.add(cloudlet);
            }

            broker.submitCloudletList(cloudletList);

            // Step 6: Start Simulation
            CloudSim.startSimulation();

            // Step 7: Retrieve simulation results
            List<Cloudlet> newList = broker.getCloudletReceivedList();
            CloudSim.stopSimulation();

            // Print FCFS results table
            printCloudletList(newList);

            Log.printLine("========== CloudSim FCFS Simulation Finished Successfully ==========");
        } catch (Exception e) {
            e.printStackTrace();
            Log.printLine("Simulation error encountered: " + e.getMessage());
        }
    }

    private static Datacenter createDatacenter(String name) {
        List<Host> hostList = new ArrayList<Host>();
        List<Pe> peList = new ArrayList<Pe>();

        int mips = 1000;
        peList.add(new Pe(0, new PeProvisionerSimple(mips)));

        int hostId = 0;
        int ram = 4096;
        long storage = 1000000;
        int bw = 10000;

        hostList.add(
            new Host(
                hostId,
                new RamProvisionerSimple(ram),
                new BwProvisionerSimple(bw),
                storage,
                peList,
                new VmSchedulerTimeShared(peList)
            )
        );

        String arch = "x86";
        String os = "Linux";
        String vmm = "Xen";
        double time_zone = 10.0;
        double cost = 3.0;
        double costPerMem = 0.05;
        double costPerStorage = 0.001;
        double costPerBw = 0.0;
        LinkedList<Storage> storageList = new LinkedList<Storage>();

        DatacenterCharacteristics characteristics = new DatacenterCharacteristics(
            arch, os, vmm, hostList, time_zone, cost, costPerMem, costPerStorage, costPerBw
        );

        Datacenter datacenter = null;
        try {
            datacenter = new Datacenter(name, characteristics, new VmAllocationPolicySimple(hostList), storageList, 0);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return datacenter;
    }

    private static DatacenterBroker createBroker() {
        DatacenterBroker broker = null;
        try {
            broker = new DatacenterBroker("Broker");
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
        return broker;
    }

    private static void printCloudletList(List<Cloudlet> list) {
        int size = list.size();
        Cloudlet cloudlet;

        String indent = "    ";
        Log.printLine();
        Log.printLine("========== FCFS EXECUTION RESULTS ==========");
        Log.printLine("Cloudlet ID" + indent + "STATUS" + indent +
                "Data center ID" + indent + "VM ID" + indent + "Time" + indent + "Start Time" + indent + "Finish Time");

        DecimalFormat dft = new DecimalFormat("###.##");
        for (int i = 0; i < size; i++) {
            cloudlet = list.get(i);
            Log.print(indent + cloudlet.getCloudletId() + indent + indent);

            if (cloudlet.getCloudletStatus() == Cloudlet.SUCCESS) {
                Log.print("SUCCESS");
                Log.printLine(indent + indent + cloudlet.getResourceId() + indent + indent + indent + cloudlet.getVmId() +
                        indent + indent + dft.format(cloudlet.getActualCPUTime()) + indent + indent + dft.format(cloudlet.getExecStartTime()) +
                        indent + indent + dft.format(cloudlet.getFinishTime()));
            }
        }
        Log.printLine("============================================");
    }
}
```

---

### G3. Step-by-Step Eclipse Navigation for Exp 11B
1. In Eclipse **Package Explorer**, expand `CloudSim` -> `sources` (or `src`).
2. Right-click the package `org.cloudbus.cloudsim.examples` -> select **New -> Class**.
3. In the "Name" field, type exactly: `ResourceAllocFCFS`.
4. Click **Finish**.
5. Select all existing text in `ResourceAllocFCFS.java` and paste the complete source code from above.
6. Press `Ctrl + S` to save the file. Ensure no syntax errors appear.
7. Right-click anywhere in the code editor (or right-click `ResourceAllocFCFS.java` in Package Explorer) -> select **Run As -> 1 Java Application**.

---

### G4. Expected Output & Viva Verification
Verify that the output table shows sequential, non-overlapping start and finish times:
```text
========== FCFS EXECUTION RESULTS ==========
Cloudlet ID    STATUS    Data center ID    VM ID    Time    Start Time    Finish Time
    0         SUCCESS          2            0        20         0.1           20.1
    1         SUCCESS          2            0        40        20.1           60.1
    2         SUCCESS          2            0        60        60.1          120.1
    3         SUCCESS          2            0        80       120.1          200.1
============================================
```
**Why this proves FCFS:**
- Cloudlet 0 starts at `0.1` and finishes at `20.1`.
- Cloudlet 1 is held in queue and starts **only after** Cloudlet 0 finishes (Start Time = `20.1`, Finish = `60.1`).
- Cloudlet 2 starts **only after** Cloudlet 1 finishes (Start Time = `60.1`).
- Cloudlet 3 starts **only after** Cloudlet 2 finishes (Start Time = `120.1`).
- This strictly demonstrates First-Come-First-Served resource allocation without preemption.

---

## H) Exp 12: CloudAnalyst Simulation in NetBeans & Standalone GUI

### Aim
Set up and execute cloud infrastructure and geographic user base simulations using the **CloudAnalyst** simulation tool in NetBeans IDE (or standalone GUI). Evaluate the impact of Data Center Broker routing policies (Closest Data Center vs Optimize Response Time) and VM Load Balancing algorithms (Round Robin vs Throttled) on response times and infrastructure costs.

---

### H1. The 10-Second Zero-Error Direct Launch (Recommended for Lab Exam)
If NetBeans has compiler errors or plugin issues, CloudAnalyst can be launched instantly as a standalone Java GUI application:
```bash
# 1. Download CloudAnalyst zip if not already present
cd ~/Downloads
wget -c http://clouds.cis.unimelb.edu.au/cloudsim/CloudAnalyst.zip || \
wget -c https://raw.githubusercontent.com/manikanta-5/CloudAnalyst/master/CloudAnalyst.zip
unzip -o CloudAnalyst.zip

# 2. Launch the CloudAnalyst GUI immediately with Java 11:
cd CloudAnalyst
java -jar CloudAnalyst.jar || java -jar dist/CloudAnalyst.jar
```
*(The graphical World Map simulation window will appear immediately on your desktop!)*

---

### H2. Opening and Running in Apache NetBeans IDE
If the lab manual specifically mandates opening and building within NetBeans:

#### Step 1: Launch NetBeans
```bash
netbeans &
```

#### Step 2: Open CloudAnalyst Project
1. In the top menu bar, click: **File -> Open Project...** (or `Ctrl + Shift + O`).
2. Navigate to your extracted `~/Downloads/CloudAnalyst` directory.
3. Look for the folder with the coffee cup icon labeled `CloudAnalyst`. Select it and click **Open Project**.

#### Step 3: Resolve Project Libraries (if prompted)
1. If NetBeans displays a "Resolve Missing Server / Libraries" warning:
   - Right-click `CloudAnalyst` in the Projects window -> select **Properties**.
   - Click **Libraries** on the left panel.
   - Ensure the Java Platform is set to **JDK 11** (or JDK 1.8).
   - Under Compile-time Libraries, if any JAR shows red, click **Add JAR/Folder...**, navigate to `CloudAnalyst/lib`, select all `.jar` files, and click **Open**.
   - Click **OK**.

#### Step 4: Clean, Build, and Run
1. Right-click the `CloudAnalyst` project node -> click **Clean and Build**.
2. Look at the Output window at the bottom: verify `BUILD SUCCESSFUL`.
3. Right-click `CloudAnalyst` -> click **Run** (or press `F6`).

---

### H3. Exact Navigational Steps in the CloudAnalyst GUI Simulation

#### Step 1: Main Simulation Window
When CloudAnalyst opens, you will see a world map displaying 6 global regions (0: N. America, 1: S. America, 2: Europe, 3: Africa, 4: Asia, 5: Oceania).
- Click the large button in the top-left toolbar labeled: **Configure Simulation**.

#### Step 2: Simulation Duration Configuration (CRITICAL STEP)
In the Simulation Configuration window:
- Locate the **Simulation Duration** field at the top right.
- Change the value from `24.0` (hours) to: `1.0` (or `0.1` hours / 60 minutes).
> **CRITICAL EXAM TIP:** If you leave the duration at 24 hours, the simulation simulates 24 hours of global web traffic, which creates millions of events and may freeze your VM. Setting `1.0` or `0.1` hours completes in 3 seconds while giving the exact same analytical charts!

#### Step 3: Geographic User Bases Configuration
In the **User Bases** table (middle-left panel):
Set up 3 User Bases representing users in different continents:
- **UB1**:
  - Region: `0` (North America)
  - Requests per User per Hr: `60`
  - Peak Hours: `3` to `9`
  - Avg Peak Users: `1000`
  - Avg Off-Peak Users: `100`
- **UB2**:
  - Region: `1` (South America)
  - Requests per User per Hr: `60`
  - Peak Hours: `7` to `13`
  - Avg Peak Users: `1000`
  - Avg Off-Peak Users: `100`
- **UB3**:
  - Region: `2` (Europe)
  - Requests per User per Hr: `60`
  - Peak Hours: `12` to `18`
  - Avg Peak Users: `1000`
  - Avg Off-Peak Users: `100`

#### Step 4: Data Centers Configuration
Click on the **Data Centers** section / tab (middle-right panel):
Configure 2 Data Centers:
- **DC1 (North America Data Center)**:
  - Name: `DC1`
  - Region: `0`
  - Architecture: `x86`, OS: `Linux`, VMM: `Xen`
  - Physical HW Hosts: `4`
  - Host RAM: `204800` MB, Storage: `1000000` MB, Available BW: `1000000`
  - Virtual Machines: Click the VM configuration row:
    - Number of VMs: `4`
    - Image Size: `10000`
    - VM Memory (RAM): `512` MB
    - Bandwidth: `1000`
- **DC2 (Europe Data Center)**:
  - Name: `DC2`
  - Region: `2`
  - Architecture: `x86`, OS: `Linux`, VMM: `Xen`
  - Physical HW Hosts: `4`
  - Host RAM: `204800` MB, Storage: `1000000` MB, Available BW: `1000000`
  - Virtual Machines:
    - Number of VMs: `4`
    - VM Memory (RAM): `512` MB

#### Step 5: Service Broker Policy Selection
In the **Service Broker Policy** dropdown menu (bottom left of config dialog), select:
- **Closest Data Center** (Routes user requests to the DC with lowest network latency).
*(For comparison studies in your viva, you can run a second trial choosing **Optimize Response Time**).*

#### Step 6: Load Balancing Policy Selection
In the **Load Balancing Policy** dropdown menu inside Data Center configuration, select:
- **Round Robin** (Distributes incoming requests equally among VMs in circular order).
*(For comparison, **Throttled** allocates requests only up to a fixed VM threshold).*

#### Step 7: Finalize & Run
1. Click the **Done** button in the bottom-right corner of the configuration dialog.
2. The dialog will close, returning you to the World Map.
3. In the top toolbar, click the button: **Run Simulation**.
4. A progress popup dialog appears ("Simulating..."). Wait for the progress bar to reach 100%.
5. The popup will say: "Simulation Completed". Click **OK**.

---

### H4. Analyzing Simulation Results & Lab Viva Metrics
The **Simulation Results** window opens automatically with statistical metrics. Capture these tables and graphs for your lab record:

#### 1. Overall Response Time Table
- **Average (ms):** e.g. `248.52 ms`
- **Minimum (ms):** e.g. `42.10 ms`
- **Maximum (ms):** e.g. `512.30 ms`

#### 2. Response Time by Region / User Base
- Table displaying Avg Response Time for UB1, UB2, and UB3.
- Notice that UB1 has the lowest response time because it is in Region 0 where DC1 resides (low network latency).

#### 3. Data Center Request Servicing Times
- Displays request processing delay and loading across DC1 and DC2.

#### 4. Cost Analysis Summary
- **Virtual Machine Cost ($):** Computed based on VM operating hours and memory allocation.
- **Data Transfer Cost ($):** Computed based on incoming and outgoing network bandwidth.
- **Total Cost ($):** VM Cost + Data Transfer Cost.

---

### H5. CloudAnalyst Corner Cases & Troubleshooting Matrix

| Error Message / Symptom | Root Cause | Immediate Working Fix |
|---|---|---|
| `UnsupportedClassVersionError: ... has been compiled by a more recent version of the Java Runtime` | Trying to build/run CloudAnalyst with Java 17/21 instead of Java 11 or 8. | Launch directly using Java 11: <br>`/usr/lib/jvm/java-11-openjdk-amd64/bin/java -jar CloudAnalyst.jar` |
| NetBeans Build Error: `package org.jfree.chart does not exist` | Missing JFreeChart charting libraries from project properties. | Right-click Project -> **Properties -> Libraries -> Add JAR/Folder** -> select all jars in `CloudAnalyst/lib` (especially `jfreechart.jar`, `jcommon.jar`). |
| GUI fails to launch: `No X11 DISPLAY variable was set` | Attempting to launch GUI from an SSH session or non-GUI terminal. | Open a terminal inside the Ubuntu Desktop GUI environment (`Ctrl + Alt + T`), or set `export DISPLAY=:0`. |
| Simulation hangs at 50% or "Java heap space" out of memory | Default simulation duration was left at 24 hours with thousands of users. | Click **Configure Simulation** -> Change Duration to `0.1` hours (or 6 minutes) and reduce Avg Peak Users to `500`. |
| "Simulation Paused" button stuck | Simulation was paused by user or reached an unhandled event. | Click "Stop Simulation", re-open "Configure Simulation", click "Done", and click "Run Simulation" again. |
| World Map is blank / gray box | Linux VM 3D graphics acceleration glitch with Java Swing. | Launch Java with 2D software rendering flag: <br>`java -Dsun.java2d.opengl=false -Dsun.java2d.pmoffscreen=false -jar CloudAnalyst.jar` |
---

### 📥 One-Liner to Download this Cheat Sheet inside your Ubuntu VM:
```bash
wget -c "https://raw.githubusercontent.com/Abpokelove/cloudlab/main/sheet.md" -O lab_sheet.md
```