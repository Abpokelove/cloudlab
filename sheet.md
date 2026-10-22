# Cloud Computing Lab Cheat Sheet (v3, Ubuntu VM edition)

Covers: Exp 2A, 2B, 2C, 3, 4, 5, 6, 8, 9, 10, 11A, 11B, 12 (Exp 1 and 7 omitted).

**Rules for every experiment**
- One experiment = one folder = its own `package.json`. Never mix projects.
- Replace every `YOUR_...` placeholder. Never publish keys, tokens, passwords or `.env` files.
- Code is written **generically**: each experiment has a small `CONFIG` block at the top. In the exam, change only those lines (resource name, fields, API URL) to match the question.
- Commands are labelled **Ubuntu** (also WSL) or **Windows (PowerShell)**. Never paste shell commands into a `scala>` or JS console.
- Cloud services (Firebase, Dropbox, Atlas, Auth0, NASA) need internet and an account. Local fallbacks are given where possible.
- **Using an Ubuntu VM?** If any `sudo apt ...` command says "Could not get lock ... held by process", jump to **Part 2, Section A3** (one-command fix). Part 2 (Experiments 8 to 12) also has "already installed? check first" blocks and a failure-fix list for every experiment.
- Section numbers 1) to 7) in Part 1 are just headings; the experiment numbers are in the titles.

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
      weather: 'latitude=13.08&longitude=80.27',
      openweather: 'q=Chennai',
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
    .validate(body ?? {}, { abortEarly: false, stripUnknown: true });

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


---
---

# PART 2: Experiments 8 to 12 on an Ubuntu VM (beginner-proof edition)

**Correct experiment numbering used from here on** (same as your lab PDFs):

| Exp | Topic | Section below |
|---|---|---|
| 8 | Hadoop 3.3.6 (HDFS + YARN + WordCount) | Section B |
| 9 | OpenMP (9A) and MPI (9B) | Section C |
| 10 | Apache Spark (RDD WordCount, local + HDFS) | Section D |
| 11A | CloudSim installation | Section E |
| 11B | CloudSim FCFS resource allocation | Section F |
| 12 | CloudAnalyst | Section G |

(An earlier draft wrongly labelled OpenMP/MPI as "8A/8B" and Hadoop as "9". Use the table above.)

**How to read this part**
- Every command block is meant to be pasted into the **Ubuntu Terminal** (`Ctrl + Alt + T`). Paste with `Ctrl + Shift + V` (plain `Ctrl + V` does not work in the terminal).
- Lines starting with `#` are comments; pasting them is harmless.
- When you type your password after `sudo`, **nothing appears on screen**. That is normal. Type it and press Enter.
- Do the steps **in order**. After each "Check" block, compare your output with the "Expected" text before moving on.
- Never run a command you do not understand twice: formatting HDFS twice or running a "setup" block twice causes the most common failures. Blocks that are unsafe to repeat are marked **RUN ONCE**.

---

## A) Ubuntu VM first-aid (do this BEFORE any experiment)

### A0. Make life easy in VirtualBox / VMware
- **Copy-paste between your PC and the VM (VirtualBox):** VM window menu **Devices -> Shared Clipboard -> Bidirectional**. If it does not work: **Devices -> Insert Guest Additions CD image**, then in the VM run `sudo apt install -y build-essential dkms linux-headers-$(uname -r)` and install from the CD. Restart the VM.
- **Give the VM enough power** (shut the VM down first): VirtualBox **Settings -> System -> Motherboard -> Base Memory = 4096 MB or more**; **Processor tab -> CPUs = 2 or more** (4 if your PC has 8+ cores). Hadoop/Spark need ~4 GB; MPI/OpenMP speedup needs 2+ CPUs.
- **Network must be NAT or Bridged:** **Settings -> Network -> Adapter 1 -> Attached to: NAT**, "Cable Connected" ticked.

### A1. Terminal survival basics

| Need | Do this |
|---|---|
| Open terminal | `Ctrl + Alt + T` (or Activities -> type "Terminal") |
| Paste / copy in terminal | `Ctrl + Shift + V` / `Ctrl + Shift + C` |
| Stop a running program | `Ctrl + C` |
| Where am I? What is here? | `pwd` and `ls -l` |
| Go to home / into a folder / up | `cd ~` / `cd foldername` / `cd ..` |
| Edit a file with nano | `nano file.txt` -> type -> `Ctrl + O` then `Enter` (save) -> `Ctrl + X` (exit) |
| Search command history | `Ctrl + R` then type part of the command |
| Clear the screen | `clear` |

### A2. First command of every lab session: check resources and internet

```bash
echo "CPU cores : $(nproc)"
free -h
df -h /
ping -c 2 archive.ubuntu.com
date
```

Expected: cores 2 or more, "Mem" total about 3.5 GB or more, "/" has **at least 10 GB Avail**, ping shows `0% packet loss`, and `date` shows the real current date/time.
If ping fails go to **A6 (network)**; if `date` is wrong go to **A6 (clock)**.

### A3. THE APT LOCK PROBLEM ("Could not get lock ... held by process ...")

**What you see**
```text
E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 1234 (unattended-upgr)
E: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?
Waiting for cache lock: Could not get lock /var/lib/dpkg/lock-frontend ...
```

**Why:** right after the VM boots, Ubuntu itself runs updates in the background (`unattended-upgrades`, `apt-daily`, `packagekit`, the "Software Updater" window). Only one program may use apt at a time, so your command is refused.

**Fix in the correct order. Do not skip to the dangerous steps.**

**Step 1: Wait (safest).** Close the "Software Updater" / "Software & Updates" window if it is open. Run:
```bash
# waits until the background update finishes (up to ~5 minutes), then tells you
for i in $(seq 1 60); do
  if sudo fuser /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/lib/apt/lists/lock /var/cache/apt/archives/lock >/dev/null 2>&1; then
    echo "apt is busy... waiting ($i/60)"; sleep 5
  else
    echo "apt is FREE. You can run your install command now."; break
  fi
done
```
If it ends with "apt is FREE", just run your `sudo apt install ...` again.

**Step 2: Make apt wait by itself (works for the rest of the day).** Run once:
```bash
echo 'DPkg::Lock::Timeout "300";' | sudo tee /etc/apt/apt.conf.d/80locktimeout
```
From now on `sudo apt install ...` waits up to 5 minutes instead of failing immediately.

**Step 3: If still stuck after 5 minutes: the automatic repair script** (creates a script once, then runs it). This kills only apt-related processes, removes stale locks only when nobody owns them, and repairs interrupted installs.
```bash
cat > ~/fixapt.sh <<'EOF'
#!/bin/bash
LOCKS="/var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/lib/apt/lists/lock /var/cache/apt/archives/lock"
busy() {
  if command -v fuser >/dev/null 2>&1; then sudo fuser $LOCKS >/dev/null 2>&1
  else sudo lsof $LOCKS >/dev/null 2>&1; fi
}
echo "[1/5] Processes using apt/dpkg right now:"
ps -eo pid,etime,cmd | grep -E 'apt|dpkg|unattended|packagekit|update-manager' | grep -v grep || echo "   none"

echo "[2/5] Stopping background updaters..."
sudo systemctl stop unattended-upgrades apt-daily.service apt-daily-upgrade.service packagekit 2>/dev/null
sleep 3
if busy; then
  echo "   still locked -> force-killing apt-related processes"
  sudo pkill -9 -f unattended-upgrade 2>/dev/null
  for p in apt apt-get dpkg aptd packagekitd update-manager; do sudo pkill -9 -x "$p" 2>/dev/null; done
  sleep 2
fi

echo "[3/5] Removing stale lock files (only if nobody holds them)..."
if busy; then
  echo "   a process STILL holds a lock. Run:  sudo fuser -v $LOCKS   then   sudo kill -9 PID"
  exit 1
fi
sudo rm -f $LOCKS

echo "[4/5] Repairing interrupted installs..."
sudo dpkg --configure -a
sudo apt-get install -f -y

echo "[5/5] Refreshing package lists..."
sudo apt-get update
echo "DONE. Now run your install command again."
EOF
chmod +x ~/fixapt.sh
bash ~/fixapt.sh
```
Keep this script. Any time apt complains, run `bash ~/fixapt.sh` again.

**Step 4 (optional but recommended): stop the VM from updating in the background during the lab.** Run once after the VM boots:
```bash
sudo systemctl stop unattended-upgrades packagekit apt-daily.timer apt-daily-upgrade.timer 2>/dev/null
sudo systemctl disable unattended-upgrades apt-daily.timer apt-daily-upgrade.timer 2>/dev/null
sudo sed -i 's/"1"/"0"/g' /etc/apt/apt.conf.d/20auto-upgrades 2>/dev/null
echo 'APT::Acquire::Retries "3";' | sudo tee /etc/apt/apt.conf.d/80retries
```
(To undo later: `sudo systemctl enable unattended-upgrades apt-daily.timer apt-daily-upgrade.timer`.)

**Manual diagnosis (only if the script says a process still holds the lock):**
```bash
sudo fuser -v /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/lib/apt/lists/lock
sudo kill -9 PID_NUMBER_SHOWN_ABOVE
sudo dpkg --configure -a
```

### A4. Other apt / package errors and their one-line fixes

| Error text | Meaning | Fix |
|---|---|---|
| `E: Could not open lock file ... (13: Permission denied)` | You forgot `sudo` | Re-run with `sudo` in front |
| `dpkg was interrupted, you must manually run 'sudo dpkg --configure -a'` | A previous install was cut off | `sudo dpkg --configure -a && sudo apt-get install -f -y` |
| `E: Unable to locate package xyz` | Package lists empty or "universe" repo off | `sudo apt update`; if still missing: `sudo apt install -y software-properties-common && sudo add-apt-repository -y universe && sudo apt update` |
| `Temporary failure resolving 'archive.ubuntu.com'` | No DNS / network | See A6 |
| `Hash Sum mismatch` or `File has unexpected size` | Corrupt cached lists | `sudo rm -rf /var/lib/apt/lists/* && sudo apt update` |
| `NO_PUBKEY ...` / `GPG error` | Repository key missing | Check the system date (A6); remove the bad repo from `/etc/apt/sources.list.d/` |
| `E: You have held broken packages` / `Unmet dependencies` | Half-installed packages | `sudo apt --fix-broken install -y && sudo apt update && sudo apt full-upgrade -y` |
| `404 Not Found [IP: ...]` while installing | Old package list | `sudo apt update` then retry |
| `No space left on device` | Virtual disk full | `df -h /`, then `sudo apt clean; sudo journalctl --vacuum-size=50M; rm -rf ~/.cache/*` |
| `username is not in the sudoers file` | Your user has no sudo rights | `su -` (root password), then `usermod -aG sudo YOUR_USERNAME`, log out and in |
| `snap ... has changes in progress` | Another snap job running | `snap changes`, wait, or `sudo snap abort ID` |
| `bash: syntax error` / `\r: command not found` after copying a file from Windows | Windows line endings | `sed -i 's/\r$//' filename` |
| `Permission denied` writing in `/usr/local/...` | Folder owned by root | `sudo chown -R $USER:$USER /usr/local/FOLDER` |

### A5. Low-RAM VM? Add swap (prevents "Cannot allocate memory", frozen Hadoop/Spark/Eclipse)
```bash
free -h | grep -i swap               # if Swap total is 0B you have none
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
free -h | grep -i swap               # should now show 2.0Gi
```
(Run once. It disappears after reboot unless you add `/swapfile none swap sw 0 0` to `/etc/fstab`; not needed for a single lab.)

### A6. Network and clock problems
```bash
ping -c 2 8.8.8.8            # works  -> internet OK, DNS is the problem
ping -c 2 archive.ubuntu.com # fails  -> DNS problem
```
DNS fix:
```bash
sudo systemctl restart systemd-resolved NetworkManager
printf 'nameserver 8.8.8.8\nnameserver 1.1.1.1\n' | sudo tee /etc/resolv.conf
ping -c 2 archive.ubuntu.com
```
If even `ping 8.8.8.8` fails: shut down the VM, VirtualBox **Settings -> Network -> Adapter 1 -> NAT**, tick **Cable Connected**, start again.

Wrong date/time (causes `certificate is not yet valid` and download errors, common after pausing the VM):
```bash
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd
sleep 5; date
```

### A7. Install the base tools (safe to run again)
```bash
sudo apt update
sudo apt install -y build-essential wget curl unzip tar git nano lsof psmisc net-tools openjdk-11-jdk openssh-server maven python3 python3-pip
```

### A8. Use Java 11 for all of Part 2
```bash
ls /usr/lib/jvm                      # shows installed JDK folders
dpkg --print-architecture           # amd64 (normal PC) or arm64 (Apple M1/M2 VM)
sudo update-alternatives --set java  /usr/lib/jvm/java-11-openjdk-$(dpkg --print-architecture)/bin/java
sudo update-alternatives --set javac /usr/lib/jvm/java-11-openjdk-$(dpkg --print-architecture)/bin/javac
java -version
javac -version
```
Expected: both say `11.0.x`. If `update-alternatives` says "no alternatives for java-11", JDK 11 is not installed: `sudo apt install -y openjdk-11-jdk`.
Interactive alternative: `sudo update-alternatives --config java` then type the number of the Java 11 line and press Enter.

### A9. The helper that stops duplicate lines in `~/.bashrc`
`~/.bashrc` holds your environment variables. Pasting the same `export` lines twice is harmless but messy; this helper adds a line only if it is not already there. Paste it once **in each new terminal** before using it:
```bash
addline() { grep -qxF "$1" ~/.bashrc || echo "$1" >> ~/.bashrc; }
```
Reload changes in the current terminal with `source ~/.bashrc`. Opening a new terminal also loads them.

### A10. One command that reports everything installed (run any time, also the night before the lab)
```bash
cat > ~/labcheck.sh <<'EOF'
#!/bin/bash
check() { local c="$1"; shift
  if command -v "$c" >/dev/null 2>&1; then echo "[OK]      $c : $("$@" 2>&1 | head -n 1)"
  else echo "[MISSING] $c"; fi; }
echo "=========== TOOLS ==========="
check java java -version
check javac javac -version
check git git --version
check mvn mvn -v
check python3 python3 --version
check gcc gcc --version
check mpicc mpicc --version
check mpirun mpirun --version
check hadoop hadoop version
check spark-submit bash -c "spark-submit --version 2>&1 | grep -m1 -i 'version'"
echo "=========== GUI IDEs ==========="
snap list 2>/dev/null | grep -Ei 'eclipse|netbeans' || echo "[MISSING] eclipse / netbeans (snap)"
echo "=========== ENV VARIABLES ==========="
echo "JAVA_HOME   = ${JAVA_HOME:-<not set>}"
echo "HADOOP_HOME = ${HADOOP_HOME:-<not set>}"
echo "SPARK_HOME  = ${SPARK_HOME:-<not set>}"
echo "=========== RUNNING JAVA DAEMONS (jps) ==========="
jps 2>/dev/null || echo "jps not found (JDK missing)"
echo "=========== SSH / DISK / RAM ==========="
systemctl is-active ssh
df -h / | tail -n 1
free -h | sed -n 2p
EOF
chmod +x ~/labcheck.sh
bash ~/labcheck.sh
```
`[MISSING]` lines tell you exactly what to install. `HADOOP_HOME = <not set>` right after installing means you forgot `source ~/.bashrc`.

---

## B) Experiment 8: Hadoop 3.3.6 (single-node: HDFS + YARN + WordCount)

### Aim
Install Hadoop 3.x in pseudo-distributed mode on one machine, start HDFS and YARN, store files in HDFS and run a MapReduce WordCount job.

### B0. "Is Hadoop already installed?" (CHECK FIRST, do not reinstall blindly)
```bash
echo "JAVA_HOME   = $JAVA_HOME"
echo "HADOOP_HOME = $HADOOP_HOME"
command -v hadoop || echo "hadoop command NOT found"
hadoop version 2>&1 | head -n 1
ls -d /usr/local/hadoop* /opt/hadoop* ~/hadoop* 2>/dev/null
ls ~/hadoopdata 2>/dev/null
grep -n "HADOOP\|JAVA_HOME" ~/.bashrc
systemctl is-active ssh
jps
ss -tulpn 2>/dev/null | grep -E ':9000|:9870|:8088'
```

| What you see | What it means | What to do |
|---|---|---|
| `Hadoop 3.3.6` printed **and** `jps` lists NameNode, DataNode, SecondaryNameNode, ResourceManager, NodeManager | Everything is already running | Skip to **B7** (HDFS commands) |
| `Hadoop 3.x` printed but `jps` shows only `Jps` | Installed, just stopped (normal after reboot) | `start-dfs.sh` then `start-yarn.sh` then `jps` (**never format again**) |
| `hadoop command NOT found` but `/usr/local/hadoop` exists | Install is fine, environment lines missing | Do **B4** only, then `source ~/.bashrc` |
| `hadoop command NOT found` and no folder | Not installed | Start at **B1** |
| A different version (e.g. 3.3.4 / 3.4.0) works | Fine; commands are identical | Use the wildcard jar name shown in B7; skip install |
| `~/hadoopdata` or `/usr/local/hadoop/hadoop_data` already exists | Already formatted once | Do **not** run `hdfs namenode -format` again unless a failure fix tells you to |
| `jps` shows some but not all 5 daemons | Partial start | Go to **B10 (Fail fixes)** |
| `JAVA_HOME` empty but `hadoop version` works | Normal; Hadoop reads JAVA_HOME from `hadoop-env.sh` | Nothing to do |

### B1. Packages (Java 11 + SSH)
If apt says "Could not get lock", go to **A3** first.
```bash
sudo apt update
sudo apt install -y openjdk-11-jdk openssh-server openssh-client wget tar
sudo systemctl enable --now ssh
systemctl is-active ssh
java -version
```
Expected: `active` and `openjdk version "11.0.x"`. Do **not** install `pdsh`; it breaks `start-dfs.sh`.
If `java -version` shows 17 or 21, do **A8**.

### B2. Passwordless SSH to localhost (Hadoop starts its daemons through SSH)
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -P "" -f ~/.ssh/id_ed25519
grep -qF "$(cat ~/.ssh/id_ed25519.pub)" ~/.ssh/authorized_keys 2>/dev/null || cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod go-w ~
ssh-keyscan -H localhost 127.0.0.1 0.0.0.0 >> ~/.ssh/known_hosts 2>/dev/null
ssh -o BatchMode=yes localhost 'echo SSH-OK'
```
Expected last line: `SSH-OK` with **no password prompt**. If it asks for a password or fails, see **B10 (SSH problems)**.

### B3. Download and place Hadoop (about 700 MB: needs 3 GB free disk and a few minutes)
```bash
cd ~
wget -c https://archive.apache.org/dist/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
ls -lh hadoop-3.3.6.tar.gz              # should be roughly 690-730 MB
tar -tzf hadoop-3.3.6.tar.gz > /dev/null && echo "ARCHIVE OK"
```
Only if you see `ARCHIVE OK`:
```bash
tar -xzf hadoop-3.3.6.tar.gz
sudo rm -rf /usr/local/hadoop              # removes an older failed install, if any
sudo mv hadoop-3.3.6 /usr/local/hadoop
sudo chown -R $USER:$USER /usr/local/hadoop
ls /usr/local/hadoop
```
Expected: folders `bin etc lib libexec sbin share ...`.
`wget` stopped halfway? Run the same `wget -c` line again; it resumes. `unexpected end of file` means a broken download: `rm hadoop-3.3.6.tar.gz` and download again.

### B4. Environment variables (safe to repeat; uses the helper from A9)
```bash
addline() { grep -qxF "$1" ~/.bashrc || echo "$1" >> ~/.bashrc; }
addline 'export HADOOP_HOME=/usr/local/hadoop'
addline 'export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop'
addline 'export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin'
addline 'export PDSH_RCMD_TYPE=ssh'
source ~/.bashrc

JH=/usr/lib/jvm/java-11-openjdk-$(dpkg --print-architecture)
ls -d $JH || echo "JDK 11 folder not found: run  ls /usr/lib/jvm  and use that folder name below"
grep -q '^export JAVA_HOME=' $HADOOP_HOME/etc/hadoop/hadoop-env.sh \
  && sed -i "s|^export JAVA_HOME=.*|export JAVA_HOME=$JH|" $HADOOP_HOME/etc/hadoop/hadoop-env.sh \
  || echo "export JAVA_HOME=$JH" >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh
echo 'export HADOOP_SSH_OPTS="-o StrictHostKeyChecking=no"' >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh

hadoop version | head -n 1
grep -n '^export JAVA_HOME' $HADOOP_HOME/etc/hadoop/hadoop-env.sh
```
Expected: `Hadoop 3.3.6` and a `JAVA_HOME=/usr/lib/jvm/java-11-openjdk-...` line.
(JAVA_HOME is written **inside `hadoop-env.sh`** on purpose: the daemons start through SSH and do not read `.bashrc`.)
If you are logged in as **root** (prompt shows `root@`): also run
```bash
cat >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh <<'EOF'
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root
EOF
```

### B5. Configuration files (RUN the whole block; it overwrites the four files safely)
```bash
mkdir -p ~/hadoopdata/namenode ~/hadoopdata/datanode ~/hadoopdata/tmp
cd $HADOOP_HOME/etc/hadoop

cat > core-site.xml <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property><name>fs.defaultFS</name><value>hdfs://localhost:9000</value></property>
  <property><name>hadoop.tmp.dir</name><value>$HOME/hadoopdata/tmp</value></property>
</configuration>
EOF

cat > hdfs-site.xml <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property><name>dfs.replication</name><value>1</value></property>
  <property><name>dfs.namenode.name.dir</name><value>file:$HOME/hadoopdata/namenode</value></property>
  <property><name>dfs.datanode.data.dir</name><value>file:$HOME/hadoopdata/datanode</value></property>
</configuration>
EOF

cat > mapred-site.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property><name>mapreduce.framework.name</name><value>yarn</value></property>
  <property><name>mapreduce.application.classpath</name><value>/usr/local/hadoop/share/hadoop/mapreduce/*:/usr/local/hadoop/share/hadoop/mapreduce/lib/*</value></property>
  <property><name>yarn.app.mapreduce.am.env</name><value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value></property>
  <property><name>mapreduce.map.env</name><value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value></property>
  <property><name>mapreduce.reduce.env</name><value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value></property>
</configuration>
EOF

cat > yarn-site.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property><name>yarn.nodemanager.aux-services</name><value>mapreduce_shuffle</value></property>
  <property><name>yarn.nodemanager.env-whitelist</name><value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value></property>
  <property><name>yarn.nodemanager.vmem-check-enabled</name><value>false</value></property>
</configuration>
EOF
cd ~
```
Check the files are readable XML (optional):
```bash
sudo apt install -y libxml2-utils
for f in core hdfs mapred yarn; do xmllint --noout $HADOOP_HOME/etc/hadoop/$f-site.xml && echo "$f-site.xml OK"; done
```
Four lines ending in `OK` expected. If you installed Hadoop in another folder (not `/usr/local/hadoop`), change the two `mapred-site.xml` paths.
The lab sheet may tell you to `cp mapred-site.xml.template`. In Hadoop 3.x that template does **not** exist; `mapred-site.xml` is already there, and the block above replaces it.

### B6. Format (FIRST TIME ONLY) and start
```bash
hdfs namenode -format
```
Look for `has been successfully formatted` near the end. **RUN ONCE.** Formatting again later erases everything in HDFS and causes the "Incompatible clusterIDs" error.
```bash
start-dfs.sh
start-yarn.sh
jps
```
Expected `jps` (numbers differ):
```text
2301 NameNode
2455 DataNode
2677 SecondaryNameNode
2890 ResourceManager
3011 NodeManager
3355 Jps
```
All 5 daemons must be present. Extra checks:
```bash
hdfs dfsadmin -report | head -n 20          # "Live datanodes (1)"
```
Open in the VM's Firefox: `http://localhost:9870` (HDFS) and `http://localhost:8088` (YARN). Take screenshots for the record.

**After every VM reboot** you only need: `start-dfs.sh && start-yarn.sh && jps`.

### B7. HDFS commands and the built-in WordCount
```bash
mkdir -p ~/wc_input
printf 'hello cloud hello hadoop\nbig data in the cloud\n' > ~/wc_input/a.txt

hdfs dfs -mkdir -p /user/$USER
hdfs dfs -put -f ~/wc_input /user/$USER/input
hdfs dfs -ls /user/$USER/input

EXJAR=$(ls $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar)
echo "$EXJAR"
hadoop jar $EXJAR wordcount /user/$USER/input /user/$USER/output
hdfs dfs -cat /user/$USER/output/part-r-00000
```
Expected output:
```text
big     1
cloud   2
data    1
hadoop  1
hello   2
in      1
the     1
```
The job prints `map 100% reduce 100%` and `Job ... completed successfully`.
Running again needs a **new/empty output folder**: `hdfs dfs -rm -r /user/$USER/output`.

Useful HDFS commands for viva/record:
```bash
hdfs dfs -ls /                       # list root
hdfs dfs -cat /user/$USER/input/a.txt
hdfs dfs -get /user/$USER/output/part-r-00000 ~/result.txt     # copy to Linux
hdfs dfs -du -h /user/$USER
hdfs dfs -rm -r /user/$USER/output
hdfs dfs -rm -f /user/$USER/input/a.txt
hdfs dfsadmin -report
hdfs fsck / -files -blocks | tail -n 15
```

### B8. Your own WordCount.java (when the question asks you to write the code)
```bash
mkdir -p ~/wc_java && cd ~/wc_java
nano WordCount.java
```
Paste:
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
    if (args.length < 2) { System.err.println("Usage: WordCount <input> <output>"); System.exit(2); }
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
Save (`Ctrl + O`, Enter, `Ctrl + X`), then compile and run:
```bash
mkdir -p classes
javac -classpath "$(hadoop classpath)" -d classes WordCount.java
jar -cvf wc.jar -C classes/ .
hdfs dfs -rm -r -f /user/$USER/output2
hadoop jar wc.jar WordCount /user/$USER/input /user/$USER/output2
hdfs dfs -cat /user/$USER/output2/part-r-00000
cd ~
```
`javac: invalid target release` or `class file version` errors mean Java is not 11: do **A8**.

### B9. Stop, restart, and a no-daemon fallback
```bash
stop-yarn.sh
stop-dfs.sh
jps                      # only Jps should remain
```
If the cluster will not start and time is short, run WordCount on the local disk without HDFS/YARN (proves the MapReduce logic; say so in your record):
```bash
EXJAR=$(ls $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar)
rm -rf ~/wc_output_local
hadoop jar $EXJAR wordcount -D mapreduce.framework.name=local -D fs.defaultFS=file:/// ~/wc_input ~/wc_output_local
cat ~/wc_output_local/part-r-00000
```

### B10. Hadoop failure fixes (find your message, run the fix)

**1. `ssh: connect to host localhost port 22: Connection refused`**
```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
systemctl status ssh --no-pager | head -n 5
```

**2. `Permission denied (publickey)` or SSH asks for a password**
```bash
chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys; chmod go-w ~
ls -l ~/.ssh
ssh -v localhost 2>&1 | tail -n 15
```
Then repeat **B2**.

**3. `pdsh@...: localhost: rcmd: socket: Permission denied`**
```bash
export PDSH_RCMD_TYPE=ssh
sudo apt remove -y pdsh
start-dfs.sh
```

**4. `Error: JAVA_HOME is not set` / `JAVA_HOME ... does not exist`**
```bash
ls /usr/lib/jvm
nano $HADOOP_HOME/etc/hadoop/hadoop-env.sh      # fix the line: export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```
(`Ctrl + W` searches inside nano; type `JAVA_HOME`.)

**5. `hadoop: command not found` / `start-dfs.sh: command not found`**
```bash
source ~/.bashrc
echo $HADOOP_HOME
ls $HADOOP_HOME/bin | head
```
Empty `HADOOP_HOME` means redo **B4**. Also check you open the terminal as the same user.

**6. `jps: command not found`** (only the JRE is installed)
```bash
sudo apt install -y openjdk-11-jdk
```

**7. NameNode missing in `jps`**
```bash
tail -n 40 $HADOOP_HOME/logs/hadoop-*-namenode-*.log
```
- Log says `NameNode is not formatted` -> `hdfs namenode -format` (only then), `start-dfs.sh`.
- Log says `Address already in use` -> `sudo lsof -i :9000` then `sudo kill -9 PID`, start again.
- Log says `Incorrect configuration: namenode address ... is not configured` -> `core-site.xml` is wrong/empty: redo **B5**.

**8. DataNode missing; log shows `Incompatible clusterIDs`** (happens after formatting twice). Clean restart (this deletes HDFS data; acceptable in a lab):
```bash
stop-yarn.sh; stop-dfs.sh
rm -rf ~/hadoopdata/namenode/* ~/hadoopdata/datanode/* ~/hadoopdata/tmp/*
hdfs namenode -format
start-dfs.sh; start-yarn.sh; jps
```
DataNode log `Permission denied` on its folder: `chmod -R 755 ~/hadoopdata && chown -R $USER:$USER ~/hadoopdata`.

**9. Old daemons keep running / ports busy** (`Address already in use`, `Stale pid`)
```bash
stop-yarn.sh; stop-dfs.sh
jps | grep -E 'NameNode|DataNode|SecondaryNameNode|ResourceManager|NodeManager' | awk '{print $1}' | xargs -r kill -9
sudo lsof -i :9000 -i :9870 -i :8088
start-dfs.sh; start-yarn.sh; jps
```

**10. `Connection refused ... localhost:9000`** while running `hdfs dfs ...` -> NameNode is down: `start-dfs.sh`, `jps`, read item 7.

**11. `Name node is in safe mode`** -> wait 30 seconds after start, or `hdfs dfsadmin -safemode leave`.

**12. `FileAlreadyExistsException: Output directory ... already exists`** -> `hdfs dfs -rm -r /user/$USER/output` (use your folder name).

**13. `Input path does not exist: hdfs://localhost:9000/...`** -> Linux paths are not HDFS paths. List with `hdfs dfs -ls /user/$USER/` and redo the `-put` of **B7**.

**14. Job stays at `ACCEPTED` / never starts / `Container is running beyond virtual memory limits`**
```bash
jps | grep -E 'ResourceManager|NodeManager'      # both must be listed
free -h                                          # close Firefox/Eclipse; add swap (A5)
yarn node -list                                  # should show 1 RUNNING node
stop-yarn.sh; start-yarn.sh
```

**15. `Could not find or load main class org.apache.hadoop.mapreduce.v2.app.MRAppMaster`** -> the classpath/env lines in `mapred-site.xml` are missing or mistyped. Redo **B5**, then `stop-yarn.sh; start-yarn.sh` and rerun the job.

**16. `Premature end of file` / `Unexpected close tag` / `XML document structures must start and end`** -> a config file is damaged. Redo **B5** and run the `xmllint` check.

**17. `InaccessibleObjectException` / `java.lang.reflect` errors** -> Hadoop 3.3 is running on Java 17+. Do **A8** (Java 11), then `stop-yarn.sh; stop-dfs.sh; start-dfs.sh; start-yarn.sh`.

**18. `WARN util.NativeCodeLoader: Unable to load native-hadoop library`** -> harmless; ignore.

**19. `ssh: Could not resolve hostname ...` / hostname problems**
```bash
hostname
echo "127.0.0.1 localhost $(hostname)" | sudo tee -a /etc/hosts
```

**20. `Cannot allocate memory` / VM freezes** -> add swap (**A5**), close other apps, run `start-dfs.sh` then `start-yarn.sh`, not everything at once.

**21. `tar: Error is not recoverable` / `gzip: unexpected end of file` / `wget` timeout** -> broken download. `rm ~/hadoop-3.3.6.tar.gz` and repeat **B3**. If the network is slow, download the same URL in Firefox and move the file to `~`.

**22. Everything is broken, start fresh** (clean reinstall, keeps your input files in `~/wc_input`)
```bash
stop-yarn.sh 2>/dev/null; stop-dfs.sh 2>/dev/null
jps | awk '$2!="Jps"{print $1}' | xargs -r kill -9
sudo rm -rf /usr/local/hadoop ~/hadoopdata
sed -i '/HADOOP_HOME\|HADOOP_CONF_DIR\|PDSH_RCMD_TYPE/d' ~/.bashrc
```
Then open a **new terminal** and repeat from **B3**.

### Viva questions (Hadoop)
- HDFS vs normal file system? Blocks (128 MB default), replication, NameNode (metadata) vs DataNode (data).
- Roles of YARN ResourceManager and NodeManager?
- Map, shuffle/sort, reduce phases of WordCount? Why a combiner?
- Why replication is 1 here? (single node).

---

## C) Experiment 9: OpenMP (9A) and MPI (9B)

### Aim
9A: shared-memory parallel programming with OpenMP (threads in one process).
9B: message-passing parallel programming with MPI (separate processes that send messages).

Both are plain C programs. No cloud account needed; everything is offline after the install.

### C0. "Already installed?" check
```bash
nproc
gcc --version | head -n 1
echo | gcc -fopenmp -dM -E - | grep -i _OPENMP
command -v mpicc mpirun
mpirun --version 2>&1 | head -n 1
dpkg -l | grep -E 'openmpi|mpich' | awk '{print $2, $3}'
hostname; getent hosts "$(hostname)"
```

| What you see | Meaning | Do |
|---|---|---|
| `#define _OPENMP 201511` (or similar) | GCC supports OpenMP | 9A needs no install |
| `mpicc` and `mpirun` paths shown, `Open MPI` version | MPI ready | 9B needs no install |
| `mpicc` found but `mpirun --version` says MPICH, or both `openmpi` and `mpich` listed | Two MPI versions mixed | Fix **C5 item 6** |
| Empty `getent` line for your hostname | `mpirun` may hang | Fix **C5 item 8** |
| Nothing found | Not installed | **C1** |

### C1. Install
If apt reports a lock, use **A3** first.
```bash
sudo apt update
sudo apt install -y build-essential openmpi-bin libopenmpi-dev
gcc --version | head -n 1
mpicc --version | head -n 1
mpirun --version | head -n 1
```
Expected: GCC version, then `gcc ...`, then `mpirun (Open MPI) 4.x`. Open a **new terminal** if `mpicc` is "not found" right after install.

### C2. Work folder
```bash
mkdir -p ~/lab9 && cd ~/lab9
pwd          # /home/YOURNAME/lab9
```
All files below are created with `cat > file <<'EOF' ... EOF`, so no editor is needed. Paste each whole block including the first and last line.

---

### 9A. OpenMP

#### 9A-1. Hello (parallel region)
```bash
cat > omp_hello.c <<'EOF'
#include <stdio.h>
#include <omp.h>

int main(void) {
    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        printf("Hello from thread %d of %d\n", tid, omp_get_num_threads());
    }
    return 0;
}
EOF
gcc -fopenmp omp_hello.c -o omp_hello
./omp_hello
OMP_NUM_THREADS=2 ./omp_hello      # choose how many threads from the command line
```
Expected: one line per thread (as many as CPU cores), in **random order** (normal). If every line says "thread 0 of 1", you forgot `-fopenmp`.

#### 9A-2. Data sharing: private, firstprivate, lastprivate, critical, atomic, reduction
```bash
cat > omp_sharing.c <<'EOF'
#include <stdio.h>
#include <omp.h>

int main(void) {
    int x = 10;          /* to demonstrate private / firstprivate */
    int last = -1;       /* to demonstrate lastprivate */
    int unsafe = 0, crit = 0, atom = 0, sum = 0;
    omp_set_num_threads(4);

    /* private: each thread gets its OWN x, uninitialised; original is untouched */
    #pragma omp parallel private(x)
    {
        x = omp_get_thread_num() * 100;
        printf("private      : thread %d sets its own x = %d\n", omp_get_thread_num(), x);
    }
    printf("after private      : original x = %d (unchanged)\n\n", x);

    /* firstprivate: each thread's own x STARTS with the original value (10) */
    #pragma omp parallel firstprivate(x)
    {
        x += omp_get_thread_num();
        printf("firstprivate : thread %d started from 10, now x = %d\n", omp_get_thread_num(), x);
    }
    printf("after firstprivate : original x = %d (unchanged)\n\n", x);

    /* lastprivate: value from the logically LAST iteration is copied out */
    #pragma omp parallel for lastprivate(last)
    for (int i = 0; i < 8; i++) last = i;
    printf("lastprivate  : last = %d (expected 7)\n\n", last);

    /* race condition, then the fixes */
    #pragma omp parallel for
    for (int i = 0; i < 100000; i++) unsafe++;              /* WRONG: data race */

    #pragma omp parallel for
    for (int i = 0; i < 100000; i++) {
        #pragma omp critical
        crit++;                                             /* correct, one thread at a time */
    }

    #pragma omp parallel for
    for (int i = 0; i < 100000; i++) {
        #pragma omp atomic
        atom++;                                             /* correct, faster than critical */
    }

    #pragma omp parallel for reduction(+:sum)
    for (int i = 1; i <= 100; i++) sum += i;                /* correct, fastest */

    printf("unsafe counter    = %d (expected 100000, usually LESS)\n", unsafe);
    printf("critical counter  = %d (expected 100000)\n", crit);
    printf("atomic counter    = %d (expected 100000)\n", atom);
    printf("reduction sum 1..100 = %d (expected 5050)\n", sum);
    return 0;
}
EOF
gcc -fopenmp omp_sharing.c -o omp_sharing
./omp_sharing
```
Expected: private values differ per thread, original `x` stays 10, `last = 7`, unsafe counter usually below 100000, the other three exact.

#### 9A-3. Speed-up measurement (serial vs parallel)
```bash
cat > omp_speed.c <<'EOF'
#include <stdio.h>
#include <math.h>
#include <omp.h>

/* ===== CONFIG ===== */
#define N 80000000L        /* raise if speed-up is not visible */
/* ================== */

int main(void) {
    printf("Cores: %d | Max threads: %d\n", omp_get_num_procs(), omp_get_max_threads());

    double t0 = omp_get_wtime(), s1 = 0.0;
    for (long i = 1; i <= N; i++) s1 += sqrt((double)i);
    double t1 = omp_get_wtime(), s2 = 0.0;

    #pragma omp parallel for reduction(+:s2)
    for (long i = 1; i <= N; i++) s2 += sqrt((double)i);
    double t2 = omp_get_wtime();

    printf("Serial   : %.3f s  (sum = %.1f)\n", t1 - t0, s1);
    printf("Parallel : %.3f s  (sum = %.1f)\n", t2 - t1, s2);
    printf("Speed-up : %.2fx\n", (t1 - t0) / (t2 - t1));

    #pragma omp parallel sections
    {
        #pragma omp section
        printf("Section A ran on thread %d\n", omp_get_thread_num());
        #pragma omp section
        printf("Section B ran on thread %d\n", omp_get_thread_num());
    }
    return 0;
}
EOF
gcc -fopenmp -O2 omp_speed.c -o omp_speed -lm
for t in 1 2 4; do echo "=== $t thread(s) ==="; OMP_NUM_THREADS=$t ./omp_speed | grep -E 'Serial|Parallel|Speed'; done
```
Fill the table in your record: threads vs parallel time vs speed-up. With 1 CPU core in the VM the speed-up stays near 1x (see C5 item 5).

---

### 9B. MPI

#### 9B-1. Hello World
```bash
cat > mpi_hello.c <<'EOF'
#include <mpi.h>
#include <stdio.h>

int main(int argc, char **argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    printf("Hello from process %d of %d\n", rank, size);
    MPI_Finalize();
    return 0;
}
EOF
mpicc mpi_hello.c -o mpi_hello
mpirun -np 4 ./mpi_hello
```
Expected (any order): `Hello from process 0 of 4` ... `3 of 4`.
**If you see "There are not enough slots available"** your VM has fewer than 4 cores. Use:
```bash
mpirun --oversubscribe -np 4 ./mpi_hello
```

#### 9B-2. Send/Recv, Bcast, Scatter, Reduce, and parallel pi
```bash
cat > mpi_demo.c <<'EOF'
#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

/* ===== CONFIG ===== */
#define PER_PROC 4            /* elements each process receives in Scatter */
#define STEPS    100000000L   /* intervals for the pi integration */
/* ================== */

int main(int argc, char **argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    printf("Hello from rank %d of %d\n", rank, size);
    MPI_Barrier(MPI_COMM_WORLD);

    /* 1. Point-to-point: rank 0 -> rank 1 */
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

    /* 2. Broadcast: rank 0 -> everyone */
    int value = (rank == 0) ? 42 : 0;
    MPI_Bcast(&value, 1, MPI_INT, 0, MPI_COMM_WORLD);
    printf("Rank %d now has broadcast value %d\n", rank, value);

    /* 3. Scatter pieces of an array, sum locally, Reduce to rank 0 */
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
        printf("Sum of 1..%d = %d (expected %d)\n", n, total, n * (n + 1) / 2);
        free(all);
    }

    /* 4. Pi by numerical integration, work split across ranks */
    double t0 = MPI_Wtime(), h = 1.0 / (double)STEPS, local_sum = 0.0, pi = 0.0;
    for (long i = rank; i < STEPS; i += size) {
        double x = h * ((double)i + 0.5);
        local_sum += 4.0 / (1.0 + x * x);
    }
    local_sum *= h;
    MPI_Reduce(&local_sum, &pi, 1, MPI_DOUBLE, MPI_SUM, 0, MPI_COMM_WORLD);
    if (rank == 0) printf("Pi = %.10f  time = %.3f s using %d processes\n", pi, MPI_Wtime() - t0, size);

    MPI_Finalize();
    return 0;
}
EOF
mpicc mpi_demo.c -o mpi_demo -lm
mpirun --oversubscribe -np 4 ./mpi_demo
for p in 1 2 4; do echo "== $p process(es)"; mpirun --oversubscribe -np $p ./mpi_demo | grep Pi; done
```
Expected: `Sum of 1..16 = 136 (expected 136)` and `Pi = 3.1415926536`. Run time falls as `p` grows (up to your core count).

---

### C5. OpenMP and MPI failure fixes

**1. `fatal error: omp.h: No such file or directory`** -> `sudo apt install -y build-essential`; check `gcc --version`.

**2. `undefined reference to 'GOMP_parallel'` / `omp_get_thread_num`** -> compile **with** `-fopenmp`: `gcc -fopenmp file.c -o file`.

**3. `undefined reference to 'sqrt'`** -> put `-lm` **after** the source file: `gcc -fopenmp -O2 file.c -o file -lm`.

**4. Every line says "thread 0 of 1" or only one line prints** -> `-fopenmp` missing, or `OMP_NUM_THREADS=1` is exported: `echo $OMP_NUM_THREADS; unset OMP_NUM_THREADS`.

**5. Speed-up is about 1x or less** -> `nproc` shows 1 (give the VM 2+ CPUs: **A0**), or `N` is too small (raise it), or you forgot `-O2`.

**6. `mpicc: command not found` / `mpirun: command not found`** -> `sudo apt install -y openmpi-bin libopenmpi-dev`, open a **new terminal**.
Mixed MPICH/Open MPI (`mpicc` and `mpirun` disagree, odd errors like `ompi_mpi_init` / `PMI` failed):
```bash
sudo update-alternatives --config mpirun
sudo update-alternatives --config mpicc
mpirun --version; mpicc --version     # both must say Open MPI
```
If still confused: `sudo apt remove -y mpich libmpich-dev && sudo apt install -y openmpi-bin libopenmpi-dev`.

**7. `There are not enough slots available in the system to satisfy the N slots`** -> use `mpirun --oversubscribe -np N ./prog`, or lower `-np` to `$(nproc)`. For every run in the shell: `export OMPI_MCA_rmaps_base_oversubscribe=1`.

**8. `mpirun` hangs forever / prints nothing / "unable to resolve hostname"** -> your hostname is not in `/etc/hosts`:
```bash
hostname
echo "127.0.0.1 $(hostname)" | sudo tee -a /etc/hosts
mpirun --oversubscribe -np 2 ./mpi_hello
```
Press `Ctrl + C` to stop a hang.

**9. `mpirun has detected an attempt to run as root`** -> you are in a root shell (`sudo -i` / `su`). Use your normal user. Last resort: `mpirun --allow-run-as-root --oversubscribe -np 4 ./mpi_hello`.

**10. `error while loading shared libraries: libmpi.so.40`** -> `sudo ldconfig`, or `sudo apt install --reinstall -y libopenmpi-dev openmpi-bin`.

**11. Warnings: `Read -1, expected ...`, `vader`, `CMA`, `Open MPI failed to TCP connect`, `btl_tcp ... no route`**
```bash
export OMPI_MCA_btl_vader_single_copy_mechanism=none
mpirun --mca btl_tcp_if_include lo --oversubscribe -np 4 ./mpi_demo
```
They are warnings, not errors, if the program output still appears.

**12. Program hangs after printing a few lines** -> a `MPI_Send` has no matching `MPI_Recv` (or you ran `-np 1` for code that needs rank 1). Run with `-np 2` or more. `Ctrl + C` to stop.

**13. `Segmentation fault`** in Scatter -> receive buffer size must equal `PER_PROC`; `all` must be allocated only on rank 0 (as in the code).

**14. Output lines in strange order, duplicated "Hello"** -> normal; processes/threads print independently.

**15. `./prog: Permission denied` / `command not found`** -> `chmod +x prog` and run as `./prog` (with `./`).

**16. Compile errors after copying from a Windows editor** -> `sed -i 's/\r$//' file.c`.

**17. ARM VM (Apple silicon) errors about architecture** -> use the arm64 packages exactly as installed by apt (nothing special); do not download x86 binaries.

### Viva questions (OpenMP/MPI)
- Threads vs processes; shared vs distributed memory.
- `private` vs `firstprivate` vs `lastprivate`; why `critical`/`atomic`/`reduction` fix a race.
- `MPI_Send/Recv` vs collective (`Bcast`, `Scatter`, `Reduce`); what are rank and communicator?
- Why speed-up is below the ideal (overhead, Amdahl's law)?

---

## D) Experiment 10: Apache Spark 3.5.x (RDD WordCount, local and on HDFS)

### Aim
Install Spark, run the Spark shell, and perform RDD word count on a local text file, then on a file stored in HDFS.

### D0. "Already installed?" check
```bash
echo "SPARK_HOME = $SPARK_HOME"
command -v spark-shell spark-submit pyspark
spark-submit --version 2>&1 | grep -m1 -i "version"
ls -d /usr/local/spark* /opt/spark* ~/spark* 2>/dev/null
grep -n "SPARK" ~/.bashrc
java -version 2>&1 | head -n 1
jps
```

| What you see | Meaning | Do |
|---|---|---|
| `version 3.x` printed | Spark already installed | Skip to **D5** (commands are identical) |
| Folder `/usr/local/spark` exists but `command not found` | Environment missing | Do **D3** only |
| Folder `/usr/local/spark/spark-3.5.1-bin-hadoop3` (nested) | Wrong extraction | `sudo mv /usr/local/spark/spark-*/* /usr/local/spark/` |
| Nothing | Not installed | **D1** |
| `java` is 17/21 | Spark 3.5 works, but Hadoop does not | Prefer Java 11: **A8** |

### D1. Java (Spark 3.5 supports Java 8/11/17; use 11)
```bash
java -version 2>&1 | head -n 1       # 11.0.x expected; if not: do A8
```

### D2. Download and extract (about 380 MB)
```bash
cd ~
wget -c https://archive.apache.org/dist/spark/spark-3.5.1/spark-3.5.1-bin-hadoop3.tgz
ls -lh spark-3.5.1-bin-hadoop3.tgz
tar -tzf spark-3.5.1-bin-hadoop3.tgz > /dev/null && echo "ARCHIVE OK"
```
Only if `ARCHIVE OK`:
```bash
tar -xzf spark-3.5.1-bin-hadoop3.tgz
sudo rm -rf /usr/local/spark
sudo mv spark-3.5.1-bin-hadoop3 /usr/local/spark
sudo chown -R $USER:$USER /usr/local/spark
ls /usr/local/spark
```
Expected: `bin conf jars python sbin ...`.
(Always use `archive.apache.org`; `dlcdn.apache.org` removes older versions and gives 404.)

### D3. Environment variables
```bash
addline() { grep -qxF "$1" ~/.bashrc || echo "$1" >> ~/.bashrc; }
addline 'export SPARK_HOME=/usr/local/spark'
addline 'export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin'
addline 'export PYSPARK_PYTHON=python3'
addline 'export SPARK_LOCAL_IP=127.0.0.1'
source ~/.bashrc
spark-submit --version 2>&1 | grep -i -m1 "version"
```
Expected: `version 3.5.1`.

### D4. Create the test input
```bash
mkdir -p ~/spark_lab && cd ~/spark_lab
printf 'cloud computing lab spark hadoop\nspark rdd parallel processing cloud\ncloud virtualization hadoop\n' > input.txt
cat input.txt
pwd
```

### D5. WordCount in the Scala shell (the lab method)
```bash
cd ~/spark_lab
rm -rf spark_output                    # Spark refuses to overwrite an existing output folder
spark-shell --master "local[*]"
```
Wait for the `scala>` prompt (10-40 seconds in a VM). **Now type/paste only Scala here**, one line at a time:
```scala
val lines = sc.textFile("input.txt")
val words = lines.flatMap(line => line.split("\\s+"))
val pairs = words.map(w => (w.toLowerCase.replaceAll("[^a-z0-9]", ""), 1)).filter(_._1.nonEmpty)
val counts = pairs.reduceByKey(_ + _)
counts.collect().foreach(println)
counts.saveAsTextFile("spark_output")
:quit
```
Expected print (order may differ):
```text
(cloud,3)
(spark,2)
(hadoop,2)
(computing,1)
(lab,1)
(rdd,1)
(parallel,1)
(processing,1)
(virtualization,1)
```
Pasting many lines at once and seeing `<console>: error: illegal start`? Type `:paste`, press Enter, paste the block, press `Ctrl + D`.
Check the saved result in a **normal terminal**:
```bash
ls ~/spark_lab/spark_output
cat ~/spark_lab/spark_output/part-*
```
The Spark web page `http://localhost:4040` works only while `spark-shell` is open (take the screenshot before `:quit`).

### D6. The same job as a script (PySpark with spark-submit; good fallback)
```bash
cd ~/spark_lab
cat > wordcount.py <<'EOF'
import re, sys
from pyspark import SparkConf, SparkContext

src = sys.argv[1] if len(sys.argv) > 1 else "input.txt"
dst = sys.argv[2] if len(sys.argv) > 2 else "py_output"

sc = SparkContext(conf=SparkConf().setAppName("WordCount").setMaster("local[*]"))
counts = (sc.textFile(src)
            .flatMap(lambda line: re.findall(r"[a-z0-9]+", line.lower()))
            .map(lambda w: (w, 1))
            .reduceByKey(lambda a, b: a + b)
            .sortBy(lambda kv: -kv[1]))
for word, n in counts.collect():
    print(word, n)
counts.saveAsTextFile(dst)
sc.stop()
EOF
rm -rf py_output
spark-submit wordcount.py input.txt py_output 2>/dev/null
cat py_output/part-*
```

### D7. Spark on HDFS (needs Experiment 8 running)
```bash
jps | grep -E 'NameNode|DataNode'          # both must be listed; if not:  start-dfs.sh
hdfs dfs -mkdir -p /user/$USER/spark_input
hdfs dfs -put -f ~/spark_lab/input.txt /user/$USER/spark_input/
hdfs dfs -rm -r -f /user/$USER/spark_hdfs_out
hdfs dfs -ls /user/$USER/spark_input
spark-shell --master "local[*]"
```
At `scala>`:
```scala
val user = System.getProperty("user.name")
val inp  = sc.textFile("hdfs://localhost:9000/user/" + user + "/spark_input/input.txt")
val cnt  = inp.flatMap(_.split("\\s+")).map(w => (w.toLowerCase, 1)).reduceByKey(_ + _)
cnt.collect().foreach(println)
cnt.saveAsTextFile("hdfs://localhost:9000/user/" + user + "/spark_hdfs_out")
:quit
```
Verify:
```bash
hdfs dfs -ls /user/$USER/spark_hdfs_out
hdfs dfs -cat /user/$USER/spark_hdfs_out/part-*
```
(The port `9000` must equal `fs.defaultFS` in your Hadoop `core-site.xml`.)

### D8. Spark failure fixes

**1. `spark-shell: command not found`** -> `source ~/.bashrc`; `echo $SPARK_HOME`; `ls $SPARK_HOME/bin | head`. Empty: redo **D3**. Or run directly: `/usr/local/spark/bin/spark-shell`.

**2. `Failed to find Spark jars directory` / `SPARK_HOME` wrong** -> nested folder. `ls /usr/local/spark`; if it shows only one `spark-3.5.1-bin-hadoop3` folder: `sudo mv /usr/local/spark/spark-*/* /usr/local/spark/`.

**3. `Output directory ... already exists` (`FileAlreadyExistsException`)** -> `rm -rf spark_output` (local) or `hdfs dfs -rm -r /user/$USER/spark_hdfs_out` (HDFS), rerun.

**4. `Input path does not exist: file:/.../input.txt`** -> you started Spark in another folder. `cd ~/spark_lab` first, or use the full path `sc.textFile("/home/YOURNAME/spark_lab/input.txt")`.

**5. `java.net.ConnectException: Call From ... to localhost:9000 failed`** -> HDFS is off: `start-dfs.sh; jps`.

**6. `Your hostname resolves to a loopback address` warning** -> harmless; to silence: `export SPARK_LOCAL_IP=127.0.0.1` (already in D3).

**7. `Service 'SparkUI' could not bind on port 4040. Attempting port 4041`** -> harmless (another Spark shell is open). Close old shells: `pkill -f spark-shell`.

**8. `java.lang.OutOfMemoryError: Java heap space` / VM freezes / killed** ->
```bash
spark-shell --master "local[2]" --driver-memory 512m
```
and add swap (**A5**); close Firefox/Eclipse.

**9. `InaccessibleObjectException` on Java 17+** -> use Java 11 (**A8**).

**10. `error: ';' expected but ... found` / `illegal start of simple expression`** -> you pasted shell commands or a broken line into `scala>`. Only Scala belongs at `scala>`. Use `:paste` for blocks.

**11. Shell hangs at start for minutes** -> normal first start in a slow VM; wait up to 2 minutes. If it never ends, `Ctrl + C` and retry with fewer cores: `--master "local[1]"`.

**12. `tar: Error is not recoverable` / `unexpected end of file`** -> broken download; `rm spark-3.5.1-bin-hadoop3.tgz` and redo **D2**.

**13. `WARN NativeCodeLoader: Unable to load native-hadoop library`** -> harmless.

**14. `python3: command not found` for pyspark** -> `sudo apt install -y python3`.

**15. `Permission denied` under `/usr/local/spark`** -> `sudo chown -R $USER:$USER /usr/local/spark`.

### Viva questions (Spark)
- RDD, transformation (lazy: `flatMap`, `map`, `reduceByKey`) vs action (`collect`, `saveAsTextFile`)?
- Why is Spark faster than MapReduce? (in-memory, DAG)
- What does `local[*]` mean? What is `sc`?

---

## E) Experiment 11A: CloudSim 5.0 installation and first run (Eclipse on Ubuntu)

### Aim
Install CloudSim 5.0 in Eclipse, add the Apache Commons Math library, and run the sample simulation `CloudSimExample1` to prove the setup works.

### E0. "Already installed?" check
```bash
java -version 2>&1 | head -n 1
javac -version
mvn -v 2>&1 | head -n 1
git --version
snap list 2>/dev/null | grep -i eclipse || echo "Eclipse (snap) NOT installed"
command -v eclipse
ls ~/cloudsim_setup 2>/dev/null
ls ~/eclipse-workspace 2>/dev/null
find ~ -maxdepth 4 -name "CloudSimExample1.java" 2>/dev/null
```

| What you see | Do |
|---|---|
| Eclipse listed by `snap list` | Skip E1 |
| `CloudSimExample1.java` path printed | CloudSim source already downloaded; skip E2 download and use that folder |
| `~/eclipse-workspace` has a `CloudSim` folder | Open Eclipse, the project may already exist (check Package Explorer) |
| Java is not 11 | **A8** |

### E1. Install Eclipse (once; about 300 MB)
If apt/snap says "lock" or "changes in progress", use **A3/A4**.
```bash
sudo snap install eclipse --classic
snap list | grep eclipse
```
Slow or failing snap? Use the tarball instead: download "Eclipse IDE for Java Developers" (Linux x86_64 `.tar.gz`) in Firefox from eclipse.org/downloads, then
```bash
tar -xzf ~/Downloads/eclipse-java-*.tar.gz -C ~
~/eclipse/eclipse &
```

### E2. Download CloudSim 5.0 and the Commons Math library
```bash
mkdir -p ~/cloudsim_setup && cd ~/cloudsim_setup
git clone https://github.com/Cloudslab/cloudsim.git
cd cloudsim
git tag | sort -V | tail -n 15
TAG=$(git tag | grep -i '5\.0' | head -n 1)
echo "Chosen tag: [$TAG]"
git checkout "tags/$TAG"
cd ..
wget -c https://repo1.maven.org/maven2/org/apache/commons/commons-math3/3.6.1/commons-math3-3.6.1.jar
ls -lh commons-math3-3.6.1.jar
find ~/cloudsim_setup -name CloudSimExample1.java
```
Expected: a tag containing `5.0`, a JAR of about 2 MB, and a path ending in `.../modules/cloudsim-examples/src/main/java/org/cloudbus/cloudsim/examples/CloudSimExample1.java`.
- Message `You are in 'detached HEAD' state` after `git checkout` is **normal**.
- `Chosen tag: []` (empty) -> in Firefox open `github.com/Cloudslab/cloudsim/releases`, download **Source code (zip)** of release 5.0, then `unzip ~/Downloads/cloudsim-*.zip -d ~/cloudsim_setup` and `mv ~/cloudsim_setup/cloudsim-* ~/cloudsim_setup/cloudsim`.
- `git clone` fails (no internet in lab): use the CloudSim zip/folder given by your lab instructor and put it at `~/cloudsim_setup/cloudsim`.
- **Do not use the newest `master`/6.x/7.x version for these exercises:** newer versions rename methods (for example `setUserId` -> `setBrokerId`) and the 11B code will not compile.

### E3. METHOD 1 (recommended): Eclipse with "Link Source" (no Maven needed; works offline)
Replace `YOURNAME` below with your Ubuntu username (`whoami` shows it).

1. **Start Eclipse:** open Activities (top-left), type `Eclipse`, click it (or run `eclipse &`).
2. **Workspace Launcher** dialog: keep `/home/YOURNAME/eclipse-workspace`, click **Launch**. Close the **Welcome** tab (the X on the tab).
3. Menu **File -> New -> Java Project**.
   - **Project name:** `CloudSim`
   - **JRE:** choose *Use an execution environment JRE:* **JavaSE-11**.
     If JavaSE-11 is not in the list: click **Configure JREs...** -> **Add...** -> **Standard VM** -> **Next** -> **Directory...** -> select `/usr/lib/jvm/java-11-openjdk-amd64` (arm64 on ARM) -> **Finish** -> tick the new entry -> **Apply and Close**, then choose JavaSE-11 again.
   - Click **Finish**. If a dialog asks **Create module-info.java**, click **Don't Create**.
4. **Link the CloudSim source:** in **Package Explorer** (left), right-click project `CloudSim` -> **Build Path -> Link Source...**
   - *Linked folder location:* **Browse...** -> `/home/YOURNAME/cloudsim_setup/cloudsim/modules/cloudsim/src/main/java` -> **OK**
   - *Folder name:* `cloudsim-src` -> **Finish**.
5. **Link the examples:** repeat step 4 with `/home/YOURNAME/cloudsim_setup/cloudsim/modules/cloudsim-examples/src/main/java`, *Folder name:* `cloudsim-examples-src`.
   (If your folder layout differs, use the directory that **contains** `org/cloudbus/cloudsim/...`; `find` output from E2 tells you.)
6. **Add the Commons Math JAR:** right-click `CloudSim` -> **Build Path -> Add External Archives...** -> type `Ctrl + L`, paste `/home/YOURNAME/cloudsim_setup/commons-math3-3.6.1.jar`, press Enter.
   Alternative: right-click `CloudSim` -> **Build Path -> Configure Build Path... -> Libraries** tab -> click **Classpath** -> **Add External JARs...** -> choose the JAR -> **Apply and Close**.
   A **Referenced Libraries** node with `commons-math3-3.6.1.jar` must now appear under the project.
7. **Check for errors:** menu **Window -> Show View -> Problems**. It must show **0 errors**. Warnings are fine. Menu **Project -> Clean... -> Clean all projects -> Clean**, wait for the progress bar (bottom-right) to finish.
8. **Run the sample:** expand `CloudSim -> cloudsim-examples-src -> org.cloudbus.cloudsim.examples`, right-click **CloudSimExample1.java** -> **Run As -> 1 Java Application**. The **Console** tab at the bottom shows the output.

### E4. METHOD 2: Eclipse "Existing Maven Projects" (needs internet the first time)
1. **File -> Import...** -> expand **Maven** -> **Existing Maven Projects** -> **Next**.
2. **Root Directory: Browse...** -> `/home/YOURNAME/cloudsim_setup/cloudsim` -> tick all listed projects -> **Finish**. Wait until the bottom-right progress bar ends (first import downloads libraries and takes minutes).
3. Errors? Right-click the project -> **Maven -> Update Project...** -> tick **Force Update** -> **OK**.
4. Run: `cloudsim-examples/src/main/java` -> `org.cloudbus.cloudsim.examples` -> `CloudSimExample1.java` -> **Run As -> Java Application**.
(Your lab PDF says **File -> Import -> General -> Existing Projects into Workspace**. That only works if the folder contains a `.project` file, for example in the ZIP provided by the lab. If Eclipse says *"No projects are found to import"*, use Method 1 or 2.)

### E5. METHOD 3: Pure terminal (use if Eclipse freezes)
```bash
cd ~/cloudsim_setup/cloudsim
mvn -q clean install -DskipTests          # needs internet once; takes several minutes
CS=~/cloudsim_setup/cloudsim
MATH=$(find ~/.m2 ~/cloudsim_setup -name 'commons-math3-*.jar' 2>/dev/null | head -n 1)
CP="$CS/modules/cloudsim/target/classes:$CS/modules/cloudsim-examples/target/classes:$MATH"
java -cp "$CP" org.cloudbus.cloudsim.examples.CloudSimExample1
```
No Maven/internet? Compile straight from source:
```bash
MATH=~/cloudsim_setup/commons-math3-3.6.1.jar
mkdir -p ~/cloudsim_out
javac -nowarn -cp "$MATH" -d ~/cloudsim_out $(find ~/cloudsim_setup/cloudsim/modules/cloudsim/src/main/java ~/cloudsim_setup/cloudsim/modules/cloudsim-examples/src/main/java -name '*.java')
java -cp "$HOME/cloudsim_out:$MATH" org.cloudbus.cloudsim.examples.CloudSimExample1
```

### E6. Expected console output (CloudSimExample1)
```text
Starting CloudSimExample1...
Initialising...
Starting CloudSim version 5.0
Datacenter_0 is starting...
Broker is starting...
Entities started.
0.0: Broker: Cloud Resource List received with 1 resource(s)
0.0: Broker: Trying to Create VM #0 in Datacenter_0
0.1: Broker: VM #0 has been created in Datacenter #2, Host #0
0.1: Broker: Sending cloudlet 0 to VM #0
400.1: Broker: Cloudlet 0 received
400.1: Broker: All Cloudlets executed. Finishing...
...
========== OUTPUT ==========
Cloudlet ID    STATUS    Data center ID    VM ID    Time    Start Time    Finish Time
    0        SUCCESS        2                0        400        0.1            400.1
CloudSimExample1 finished!
```
The exact numbers/lines can differ slightly between versions. **Success = a Cloudlet row with `SUCCESS` and the final "finished!" line, with no red stack trace.** Take a screenshot of the Console.

### E7. CloudSim / Eclipse failure fixes

| Symptom | Cause | Fix |
|---|---|---|
| Red errors `The import org.apache.commons.math3 cannot be resolved` or `NoClassDefFoundError: org/apache/commons/math3/...` | Commons Math JAR not on the build path | **E3 step 6**; then **Project -> Clean** |
| `Project 'CloudSim' has no explicit encoding set` or `The type ... is not accessible` / `module` errors | Eclipse made a `module-info.java` | Delete `module-info.java` (right-click -> Delete). Recreate the project and choose **Don't Create** |
| `Unbound classpath container: 'JRE System Library [JavaSE-11]'` | JDK 11 not registered in Eclipse | **Window -> Preferences -> Java -> Installed JREs -> Add... -> Standard VM -> Directory** `/usr/lib/jvm/java-11-openjdk-amd64` -> tick it -> **Apply and Close**; then right-click project -> **Build Path -> Configure Build Path -> Libraries -> JRE System Library -> Edit... -> Workspace default JRE** |
| `Java compiler level does not match the version of the installed Java project facets` / `Source option 5 is no longer supported` | Compliance level too old | Right-click project -> **Properties -> Java Compiler -> Enable project specific settings -> Compiler compliance level = 11** -> **Apply and Close** |
| `The declared package "org.cloudbus.cloudsim.examples" does not match the expected package ""` | Wrong folder linked / not marked as a source folder | Re-link the folder that **contains** `org/` (E3 steps 4-5), or right-click the folder -> **Build Path -> Use as Source Folder** |
| **Run As** shows no "Java Application" | The class has no `main`, or the file is not selected | Click once on `CloudSimExample1.java` in Package Explorer, then **Run -> Run As -> Java Application** |
| `Error: Could not find or load main class ...` | Project not built / errors present | Fix Problems tab to 0 errors, **Project -> Clean**, run again |
| Eclipse window does not open / "JVM terminated. Exit code=13" | Wrong Java default | `sudo update-alternatives --config java` (choose 11), start Eclipse again from the terminal to see messages |
| `No projects are found to import` | Folder has no `.project` | Use E3 (Link Source) or E4 (Maven) |
| `mvn: command not found` | Maven missing | `sudo apt install -y maven` |
| Maven `release version 17 not supported` / `invalid target release` | Wrong JDK | **A8** (Java 11), `mvn -v` must show Java 11 |
| Maven `Could not transfer artifact` / `Connection reset` | No internet in VM | Use E5's second block (no Maven) or E3 |
| `git clone` stuck / `Could not resolve host: github.com` | Network | **A6** |
| Eclipse very slow | Low RAM | Give VM 4 GB (A0), add swap (A5), close Firefox |
| `Workspace in use or cannot be created` | A previous Eclipse is still running | `pkill -f eclipse` then start again; or choose a new workspace folder |

---

## F) Experiment 11B: CloudSim FCFS (First Come First Served) resource allocation

### Aim
Implement FCFS allocation in CloudSim and show that cloudlets run one after another in arrival order (a later cloudlet waits until the earlier one ends).

### F1. Idea in four lines
- CloudSim's `CloudletSchedulerSpaceShared` gives the VM's CPU to one cloudlet at a time; others wait in a queue = **FCFS**.
- `CloudletSchedulerTimeShared` divides the CPU among all cloudlets at once = round-robin-like (not FCFS).
- Cloudlets are submitted in order 0, 1, 2, 3 with different lengths (MI = million instructions).
- Time = length / MIPS. At 250 MIPS: 5000 MI -> 20 s, 10000 -> 40 s, 15000 -> 60 s, 20000 -> 80 s.

### F2. The source code (`ResourceAllocFCFS.java`)
Generic: change only the CONFIG block (number of VMs, speed, cloudlet lengths, FCFS on/off).
```java
package org.cloudbus.cloudsim.examples;

import java.text.DecimalFormat;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.LinkedList;
import java.util.List;

import org.cloudbus.cloudsim.Cloudlet;
import org.cloudbus.cloudsim.CloudletScheduler;
import org.cloudbus.cloudsim.CloudletSchedulerSpaceShared;
import org.cloudbus.cloudsim.CloudletSchedulerTimeShared;
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

public class ResourceAllocFCFS {

    // ================= CONFIG: change only these lines =================
    static final boolean FCFS = true;                          // true = FCFS (space-shared), false = time-shared (for comparison)
    static final int VM_COUNT = 1;                             // number of VMs
    static final int VM_MIPS = 250;                            // speed of each VM
    static final long[] CLOUDLET_LENGTHS = {5000, 10000, 15000, 20000};   // one cloudlet per entry, in arrival order
    // ===================================================================

    public static void main(String[] args) {
        Log.printLine("===== CloudSim FCFS resource allocation (" + (FCFS ? "FCFS / space-shared" : "time-shared") + ") =====");
        try {
            CloudSim.init(1, Calendar.getInstance(), false);

            createDatacenter("Datacenter_0");
            DatacenterBroker broker = new DatacenterBroker("Broker");
            int brokerId = broker.getId();

            // ---- VMs ----
            List<Vm> vmList = new ArrayList<Vm>();
            for (int i = 0; i < VM_COUNT; i++) {
                CloudletScheduler scheduler = FCFS ? new CloudletSchedulerSpaceShared()
                                                   : new CloudletSchedulerTimeShared();
                vmList.add(new Vm(i, brokerId, VM_MIPS, 1, 512, 1000, 10000, "Xen", scheduler));
            }
            broker.submitVmList(vmList);

            // ---- Cloudlets (submitted in arrival order) ----
            List<Cloudlet> cloudletList = new ArrayList<Cloudlet>();
            UtilizationModel full = new UtilizationModelFull();
            for (int i = 0; i < CLOUDLET_LENGTHS.length; i++) {
                Cloudlet c = new Cloudlet(i, CLOUDLET_LENGTHS[i], 1, 300, 300, full, full, full);
                c.setUserId(brokerId);
                c.setVmId(i % VM_COUNT);                       // cloudlet i goes to VM (i mod VM_COUNT)
                cloudletList.add(c);
            }
            broker.submitCloudletList(cloudletList);

            CloudSim.startSimulation();
            List<Cloudlet> result = broker.getCloudletReceivedList();
            CloudSim.stopSimulation();

            print(result);
            Log.printLine("===== Simulation finished =====");
        } catch (Exception e) {
            e.printStackTrace();
            Log.printLine("Simulation failed: " + e.getMessage());
        }
    }

    private static Datacenter createDatacenter(String name) throws Exception {
        List<Pe> peList = new ArrayList<Pe>();
        for (int p = 0; p < VM_COUNT; p++) peList.add(new Pe(p, new PeProvisionerSimple(1000)));

        List<Host> hostList = new ArrayList<Host>();
        hostList.add(new Host(0, new RamProvisionerSimple(16384), new BwProvisionerSimple(100000),
                1000000, peList, new VmSchedulerTimeShared(peList)));

        DatacenterCharacteristics ch = new DatacenterCharacteristics("x86", "Linux", "Xen", hostList,
                10.0, 3.0, 0.05, 0.001, 0.0);
        return new Datacenter(name, ch, new VmAllocationPolicySimple(hostList), new LinkedList<Storage>(), 0);
    }

    private static void print(List<Cloudlet> list) {
        String t = "    ";
        DecimalFormat f = new DecimalFormat("###.##");
        Log.printLine();
        Log.printLine("========== OUTPUT ==========");
        Log.printLine("Cloudlet" + t + "STATUS" + t + "DC" + t + "VM" + t + "Time" + t + "Start" + t + "Finish" + t + "Waited");
        for (Cloudlet c : list) {
            if (c.getCloudletStatus() == Cloudlet.SUCCESS) {
                double waited = c.getExecStartTime() - c.getSubmissionTime();
                Log.printLine(c.getCloudletId() + t + "SUCCESS" + t + c.getResourceId() + t + c.getVmId() + t
                        + f.format(c.getActualCPUTime()) + t + f.format(c.getExecStartTime()) + t
                        + f.format(c.getFinishTime()) + t + f.format(waited));
            }
        }
    }
}
```

### F3. Create and run it in Eclipse (Method 1 or 2 project)
1. In **Package Explorer** expand `cloudsim-examples-src` (Method 2: `cloudsim-examples/src/main/java`).
2. Right-click the package **org.cloudbus.cloudsim.examples** -> **New -> Class**.
3. **Name:** `ResourceAllocFCFS`. Untick *public static void main* (the code already has it). **Finish**.
4. Click inside the editor, `Ctrl + A` (select all), paste the code from F2 (it replaces everything), `Ctrl + S` (save). The editor must show **no red marks**.
5. Right-click inside the editor (or the file in Package Explorer) -> **Run As -> 1 Java Application**. Read the **Console** tab.
(If your lab names the class `resourcealloc`, rename both the file and the `public class` line; the name must match.)

**Terminal alternative (no Eclipse):**
```bash
nano ~/cloudsim_setup/ResourceAllocFCFS.java        # paste the code, Ctrl+O, Enter, Ctrl+X
CS=~/cloudsim_setup/cloudsim
MATH=$(find ~/.m2 ~/cloudsim_setup -name 'commons-math3-*.jar' 2>/dev/null | head -n 1)
CP="$CS/modules/cloudsim/target/classes:$MATH"
[ -d $CS/modules/cloudsim/target/classes ] || CP="$HOME/cloudsim_out:$MATH"     # when you built with the E5 no-Maven block
mkdir -p ~/cloudsim_fcfs
javac -nowarn -cp "$CP" -d ~/cloudsim_fcfs ~/cloudsim_setup/ResourceAllocFCFS.java
java -cp "$HOME/cloudsim_fcfs:$CP" org.cloudbus.cloudsim.examples.ResourceAllocFCFS
```

### F4. Expected output and how to explain it
With `FCFS = true`:
```text
========== OUTPUT ==========
Cloudlet    STATUS    DC    VM    Time    Start    Finish    Waited
0    SUCCESS    2    0    20    0.1    20.1    0
1    SUCCESS    2    0    40    20.1    60.1    20
2    SUCCESS    2    0    60    60.1    120.1    60
3    SUCCESS    2    0    80    120.1    200.1    120
```
- Cloudlet 1 **starts only when cloudlet 0 finishes** (start 20.1 = finish of cloudlet 0), cloudlet 2 starts at 60.1, cloudlet 3 at 120.1. Waiting time grows because earlier jobs are served first. That is FCFS.
- Set `FCFS = false`, rerun: all four cloudlets start together at about 0.1 and the short ones finish first (about 80, 140, 180, 200), proving it is *not* FCFS. The last finish stays about 200.1 in both cases.
- Try `VM_COUNT = 2`: cloudlets 0,2 go to VM 0 and 1,3 to VM 1; total time drops.
The exact decimals can differ slightly by version; the **order and "start = previous finish"** are what matter.

### F5. FCFS-specific failure fixes

| Symptom | Fix |
|---|---|
| `cannot find symbol: method setUserId(int)` | You are on CloudSim 6.x/7.x. Use the 5.0 tag (E2) or change `setUserId` to `setBrokerId` |
| `The declared package ... does not match` | Class created in the wrong folder; create it through **New -> Class** on the `org.cloudbus.cloudsim.examples` package |
| `The public type ResourceAllocFCFS must be defined in its own file` | File name and class name differ; make them identical |
| Console shows `Simulation failed` + `Creation of VM #0 failed` | The Host has too little RAM/PEs: keep `VM_COUNT` PEs in the host (code does) and `VM_MIPS <= 1000` |
| Output table empty | A cloudlet did not finish; make `VM_MIPS` and lengths smaller/larger consistently; check `Console` for `Creation of VM failed` |
| All start times equal | `FCFS` is false, or the scheduler line was changed back to `CloudletSchedulerTimeShared` |
| Eclipse shows red `CloudletScheduler` import | Library not linked; redo E3 step 4-6 and **Project -> Clean** |

### Viva questions (CloudSim)
- Main CloudSim entities: Datacenter, Host, Vm, Cloudlet, Broker, CloudSim core.
- Space-shared vs time-shared (VM scheduler vs cloudlet scheduler).
- What is a MIPS, MI, PE? How is cloudlet time computed?

---

## G) Experiment 12: CloudAnalyst (NetBeans + GUI simulation)

### Aim
Install CloudAnalyst, define user bases and data centers, run a simulation and read response-time and cost results.

### G0. "Already installed?" check
```bash
java -version 2>&1 | head -n 1
snap list 2>/dev/null | grep -i netbeans || echo "NetBeans (snap) NOT installed"
ls ~/cloudanalyst 2>/dev/null
find ~ -maxdepth 4 -iname "CloudAnalyst*" 2>/dev/null
```
Netbeans listed -> skip G1. A `CloudAnalyst` folder or zip exists -> skip the download in G2.

### G1. Install NetBeans (once)
```bash
sudo apt install -y openjdk-17-jdk             # NetBeans itself runs best on 17; your project can still use 11
sudo snap install netbeans --classic
snap list | grep netbeans
```
Snap errors: **A3/A4**. NetBeans does not start from the menu? Run `netbeans &` in a terminal to see messages; if it complains about Java: `netbeans --jdkhome /usr/lib/jvm/java-17-openjdk-amd64 &`.

### G2. Get CloudAnalyst
Use the ZIP given by your lab if you have it (copy it into `~/cloudanalyst`). Otherwise try the original link:
```bash
mkdir -p ~/cloudanalyst && cd ~/cloudanalyst
wget -c http://clouds.cis.unimelb.edu.au/cloudsim/CloudAnalyst.zip
ls -lh CloudAnalyst.zip
unzip -o CloudAnalyst.zip
find . \( -iname '*.jar' -o -iname 'nbproject' -o -iname 'build.xml' -o -iname 'run*' \) | head -n 30
```
(If the download fails, the university server is down or blocked: ask the instructor for the ZIP; do not use random download sites.)
The `find` output tells you what you have: a `nbproject` folder + `build.xml` = a NetBeans project; a `CloudAnalyst.jar` = can be run directly.

### G3. QUICK START (no IDE): run the program directly
```bash
cd ~/cloudanalyst
JAR=$(find . -iname 'CloudAnalyst*.jar' | head -n 1); echo "JAR = $JAR"
java -jar "$JAR"
```
- If it prints `no main manifest attribute`:
```bash
jar tf "$JAR" | grep -i 'GuiMain'                      # shows the main class path, e.g. cloudsim/ext/gui/GuiMain.class
LIBS=$(find . -iname '*.jar' | tr '\n' ':')
java -cp "$LIBS" cloudsim.ext.gui.GuiMain
```
The CloudAnalyst window with the **world map** opens. Continue at **G5**.

### G4. THE LAB WAY: open and run it in NetBeans (exact clicks)
1. Start NetBeans (Activities -> `NetBeans`, or `netbeans &`). Close the **Start Page** tab.
2. **Register Java 11 once:** menu **Tools -> Java Platforms** -> **Add Platform...** -> *Java Standard Edition* -> **Next** -> browse to `/usr/lib/jvm/java-11-openjdk-amd64` -> **Next** -> **Finish** -> **Close**.
3. **File -> Open Project...** (`Ctrl + Shift + O`). Browse to `~/cloudanalyst/...` and select the folder that shows a **small project icon** (the one containing `nbproject` and `build.xml`; **not** its parent folder). Click **Open Project**.
4. **Projects** panel (left) shows `CloudAnalyst`. A **red "!" / "Resolve Reference Problems"** means missing libraries: right-click the project -> **Resolve Reference Problems...** -> **Resolve...** -> browse to the missing `.jar` (look inside the project's `lib` or `jars` folder, or the extracted ZIP) -> **Open** -> **Close**.
5. Right-click the project -> **Properties** -> **Libraries**: *Java Platform* dropdown = **JDK 11** (the one you added); **Sources**: *Source/Binary Format* = **11** (or 8) -> **OK**.
6. Right-click the project -> **Clean and Build**. In the **Output** window at the bottom wait for **BUILD SUCCESSFUL**.
7. Right-click the project -> **Run** (or press `F6`). If NetBeans asks for a main class: right-click project -> **Properties -> Run -> Main Class -> Browse...** -> pick **cloudsim.ext.gui.GuiMain** -> **Select Main Class -> OK**, then run again.
8. The CloudAnalyst window opens. Your lab PDF says an error line in the Output window may be ignored **if the window opens**. A build that fails and shows no window is not ignorable; use the fixes in G7.

### G5. Using the CloudAnalyst window (exact order)
The main window shows a world map and buttons **Configure Simulation**, **Define Internet Characteristics**, **Run Simulation**, **Exit**.
Regions are numbered **0 = N. America, 1 = S. America, 2 = Europe, 3 = Asia, 4 = Africa, 5 = Oceania**.

**Step 1: click Configure Simulation.** A window with three tabs opens: **Main Configuration**, **Data Center Configuration**, **Advanced**.

**Step 2: tab "Main Configuration"**
- **Simulation Duration:** leave `60.0 min` (do not set 24 hours; it makes the run very slow).
- **User bases** table: click **Add New** for each row and fill:

| Name | Region | Requests per user per hr | Data size per request (bytes) | Peak start (GMT) | Peak end (GMT) | Avg peak users | Avg off-peak users |
|---|---|---|---|---|---|---|---|
| UB1 | 0 | 60 | 100 | 3 | 9 | 1000 | 100 |
| UB2 | 2 | 60 | 100 | 13 | 15 | 1000 | 100 |
| UB3 | 3 | 60 | 100 | 20 | 22 | 1000 | 100 |

- **Application Deployment Configuration:** **Service Broker Policy** dropdown = **Closest Data Center**. Click **Add New** under the deployment table, choose data center `DC1`, VMs `5`, Image size `10000`, Memory `512`, BW `1000`. Add another row for `DC2` the same way.
  (A data center name must exist in the next tab; if the dropdown is empty, do Step 3 first, then come back.)

**Step 3: tab "Data Center Configuration"**
- Click **Add New**. Fill row 1: **Name** `DC1`, **Region** `0`, **Arch** `x86`, **OS** `Linux`, **VMM** `Xen`, **Cost per VM $/Hr** `0.1`, **Memory cost $/s** `0.05`, **Storage cost $/s** `0.1`, **Data transfer cost $/Gb** `0.1`, **Physical HW Units** `1`.
- Row 2: `DC2`, **Region** `3`, same values.
- Select a data center row and click **Physical HW Details** (button below the table) and set: **Memory (Mb)** `204800`, **Storage (Mb)** `100000000`, **Available BW** `1000000`, **Number of processors** `4`, **Processor speed (MIPS)** `10000`, **VM Policy** `Time Shared`; confirm with **OK**/**Done**. Do it for both data centers.

**Step 4: tab "Advanced"**
- Leave **User grouping factor** `1000`, **Request grouping factor** `100`, **Executable instruction length** `250`.
- **Load balancing policy across VMs in a single data center:** choose **Round Robin** first (other options: *Equally spread current execution load*, *Throttled*).

**Step 5: click Done** (bottom of the configuration window). If an error box appears, read it: it names the missing field (usually a data center used in the deployment table but not defined).

**Step 6: back on the main window click Run Simulation.** Wait for the progress bar to complete (seconds to a minute). A **Simulation Results** window opens.

### G6. Results to copy into your record
The results window shows:
- **Overall response time summary:** Average, Minimum, Maximum (ms).
- **Response time by region:** one row per user base (UB1, UB2, UB3).
- **Data center request servicing times:** average/min/max per data center (DC1, DC2).
- **Cost:** total virtual machine cost, data transfer cost, grand total ($).
Take screenshots of all sections.

**Comparison run (usually asked in viva):** change only one thing, run again, fill:

| Run | Service Broker Policy | Load balancing | Avg response time (ms) | Total cost ($) |
|---|---|---|---|---|
| 1 | Closest Data Center | Round Robin | ... | ... |
| 2 | Optimize Response Time | Round Robin | ... | ... |
| 3 | Closest Data Center | Throttled | ... | ... |

Reopen **Configure Simulation**, change only the dropdown (policy on **Main Configuration**, load balancing on **Advanced**), click **Done**, **Run Simulation**.
Typical observations: a user base in the same region as a data center gets the lowest response time; more VMs reduce data-center processing time but increase VM cost.

### G7. CloudAnalyst failure fixes

| Symptom | Fix |
|---|---|
| `java: command not found` / `UnsupportedClassVersionError` | Wrong Java. Run with Java 11 explicitly: `/usr/lib/jvm/java-11-openjdk-amd64/bin/java -jar CloudAnalyst.jar` or **A8** |
| `no main manifest attribute` | Use the `-cp ... cloudsim.ext.gui.GuiMain` form in G3 |
| `Error: Could not find or load main class cloudsim.ext.gui.GuiMain` | Class path missing the jars: use the `find ... | tr` LIBS line from G3 from the correct folder; confirm with `jar tf` where the main class is |
| NetBeans: `package org.jfree.chart does not exist` or `cannot find symbol` for library classes | Missing JARs: project **Properties -> Libraries -> Compile -> Add JAR/Folder...** and add every `.jar` from the ZIP's `lib`/`jars` folder |
| NetBeans: project opens with red "!" | Right-click -> **Resolve Reference Problems** (G4 step 4) |
| NetBeans says *"Project requires JDK X"* or build errors about source/target | **Properties -> Libraries -> Java Platform = JDK 11**, **Sources -> Source/Binary Format = 11** (or 8) |
| `Open Project` shows nothing selectable | You selected the parent folder; go one level inside until the folder has the project icon (contains `nbproject`) |
| NetBeans will not start | `sudo apt install -y openjdk-17-jdk`; `netbeans --jdkhome /usr/lib/jvm/java-17-openjdk-amd64 &` |
| `Cannot connect to X server` / `No X11 DISPLAY variable was set` | Start it from a terminal **inside the VM desktop**, not from SSH; or `export DISPLAY=:0` |
| Window opens but the map is blank/grey | `java -Dsun.java2d.opengl=false -Dsun.java2d.xrender=false -jar CloudAnalyst.jar`; or enable **3D acceleration** in VirtualBox Display settings |
| **Done** shows an error | A data center used in the deployment table is not defined (or region number/field empty): fix and click Done again |
| Simulation never finishes / `OutOfMemoryError` | Keep duration 60 min, reduce users (e.g. peak 500), run with `java -Xmx1024m -jar CloudAnalyst.jar` |
| Results window did not open | Re-open **Run Simulation**; watch the Output/terminal for exceptions; if `NullPointerException`, a required table (user base/DC) is empty |
| Download link dead (`404`, `could not resolve host`) | Use the lab-provided ZIP; do not download JARs from unofficial sites |

### Viva questions (CloudAnalyst)
- What does CloudAnalyst model? (geographically distributed user bases, data centers, response time, cost; built on CloudSim)
- What do the Service Broker Policies do? What is Round Robin vs Throttled load balancing?
- Why does response time change when a data center is moved to another region? (network latency)

---

## H) Final checklist and lab-record helper

### H1. The night before the lab (10 minutes)
```bash
bash ~/fixapt.sh                  # only if apt complained earlier
bash ~/labcheck.sh                # [MISSING] lines = what to install
start-dfs.sh && start-yarn.sh && jps            # Hadoop: 5 daemons
mpirun --oversubscribe -np 2 ~/lab9/mpi_hello   # MPI works
~/lab9/omp_hello                                # OpenMP works
echo 'hello spark hello' | spark-shell --master "local[1]" 2>&1 | tail -n 3   # Spark starts
```
Also confirm: VM has 4 GB RAM and 2+ CPUs, 10+ GB free disk (`df -h /`), internet works (`ping -c 2 archive.ubuntu.com`), Eclipse and NetBeans open once, and your VM snapshot is saved (VirtualBox: **Machine -> Take Snapshot**). A snapshot lets you restore a working VM in one click if something breaks during the lab.

### H2. During the lab: five habits that save marks
1. **Read the last 10 lines of the error**, not the first. Copy the exact message into the matching fix table in this sheet.
2. **Run `bash ~/labcheck.sh`** if something "was working yesterday".
3. **Never run `hdfs namenode -format` or `rm -rf` unless a fix above tells you to.**
4. **Take a screenshot after each successful step** (command + output visible).
5. If a long download fails, the same command with `wget -c` resumes it.

### H3. Record format
```text
Aim:        one line.
Software:   names and versions (java -version, hadoop version, spark-submit --version, mpirun --version).
Procedure:  numbered steps (copy from the section "Steps").
Program:    code or commands.
Output:     screenshot or pasted real output.
Result:     Thus, the experiment was executed successfully and the output was verified.
```

---
****### 📥 One-Liner to Download this Cheat Sheet inside your Ubuntu VM:
```bash
wget -c "https://raw.githubusercontent.com/Abpokelove/cloudlab/main/sheet.md" -O lab_sheet.md
```****
Last updated: 2026-10-22