# Cloud Computing Lab Cheat Sheet

This file is a practical lab cheat sheet reconstructed from the PDF experiments in this folder. Some parts of the PDFs are cut off or partially missing, so this guide fills the gaps with working commands, code templates, and the usual lab flow.

Important:
- If a command fails, try the backup command shown in the same section.
- Use exact project names and ports shown in the examples.
- For all Node.js apps: run `npm install` first.
- If `node` or `npm` is missing, install Node.js LTS.

---

## 1) Quick Setup Commands

### Ubuntu/Linux common setup
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget git unzip build-essential
node -v
npm -v
java -version
```

### If Java is missing
```bash
sudo apt install -y openjdk-17-jdk
java -version
```

### If Node.js is missing
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

### If npm install fails due to permission
```bash
sudo npm install -g npm@latest
```

---

## 2) Experiment 1: Study of Different Computing Paradigms and MERN Stack

### Aim
Study distributed, parallel, cluster, grid, utility, edge, fog, and cloud computing; understand MERN stack.

### Core theory points
- Distributed computing: multiple systems work together on a single problem.
- Parallel computing: tasks are executed concurrently on multiple processors.
- Cluster computing: multiple computers work as one system.
- Grid computing: many systems form a virtual supercomputer.
- Utility computing: resources are provided on-demand and paid per use.
- Edge computing: processing is moved closer to users/devices.
- Fog computing: middleware layer between cloud and edge devices.
- Cloud computing: on-demand delivery of servers, storage, database, networking, and services.

### MERN stack
- MongoDB: database
- Express.js: backend framework
- React.js: frontend library
- Node.js: runtime

### Quick commands
```bash
npm init -y
npm install express mongoose cors dotenv react react-dom
```

### Simple MERN folder structure
```text
project/
  client/
  server/
  .env
```

### Backend Express template
```js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 5000;

app.use(cors());
app.use(express.json());

mongoose.connect(process.env.MONGODB_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB error:', err));

app.get('/', (req, res) => {
  res.json({ message: 'MERN backend is running' });
});

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

### Frontend React template
```bash
npx create-react-app client
cd client
npm install axios react-router-dom
```

### Failure backup
If React app creation fails:
```bash
npx create-react-app@latest client --template cra
```

---

## 3) Experiment 2A: Building RESTful API with Express.js

### Aim
Explore RESTful API design with Express.js.

### Commands
```bash
mkdir api-demo
cd api-demo
npm init -y
npm install express
```

### Working Express REST API template
```js
const express = require('express');
const app = express();
const PORT = 3000;

app.use(express.json());

let users = [
  { id: 1, name: 'Alice', email: 'alice@example.com' },
  { id: 2, name: 'Bob', email: 'bob@example.com' }
];

app.get('/', (req, res) => {
  res.send('REST API is running');
});

app.get('/api/users', (req, res) => {
  res.json(users);
});

app.get('/api/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const user = users.find(u => u.id === id);
  if (!user) return res.status(404).json({ message: 'User not found' });
  res.json(user);
});

app.post('/api/users', (req, res) => {
  const { name, email } = req.body;
  if (!name || !email) return res.status(400).json({ message: 'Name and email required' });

  const newUser = { id: users.length ? users[users.length - 1].id + 1 : 1, name, email };
  users.push(newUser);
  res.status(201).json(newUser);
});

app.put('/api/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = users.findIndex(u => u.id === id);
  if (index === -1) return res.status(404).json({ message: 'User not found' });

  users[index] = { ...users[index], ...req.body };
  res.json(users[index]);
});

app.delete('/api/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = users.findIndex(u => u.id === id);
  if (index === -1) return res.status(404).json({ message: 'User not found' });

  const deleted = users.splice(index, 1)[0];
  res.json({ message: 'User deleted', user: deleted });
});

app.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);
});
```

### Run it
```bash
node app.js
```

### Browser/HTTP tests
```bash
curl http://localhost:3000/api/users
```

---

## 4) Experiment 2B: Consuming Third-Party APIs with Express

### Aim
Integrate a third-party API into an Express app.

### Example: NASA APOD API

#### Install dependencies
```bash
npm init -y
npm install express dotenv
```

#### `.env`
```env
PORT=5000
NASA_API_KEY=YOUR_API_KEY_HERE
```

#### `server.js`
```js
require('dotenv').config({ quiet: true });
const express = require('express');
const path = require('path');
const app = express();
const PORT = process.env.PORT || 5000;
const API_URL = 'https://api.nasa.gov/planetary/apod';
const API_KEY = process.env.NASA_API_KEY;

if (!API_KEY) {
  console.error('NASA API key is missing in .env file');
  process.exit(1);
}

app.use(express.static(path.join(__dirname, 'public')));

app.get('/api/apod', async (req, res) => {
  try {
    const selectedDate = req.query.date || '2024-01-01';
    const url = new URL(API_URL);
    url.searchParams.append('api_key', API_KEY);
    url.searchParams.append('date', selectedDate);
    url.searchParams.append('thumbs', 'true');

    const response = await fetch(url);
    const data = await response.json();

    if (!response.ok) {
      return res.status(response.status).json({ error: data.error?.message || data.msg || 'NASA request failed' });
    }

    res.json(data);
  } catch (error) {
    console.error('NASA API error:', error.message);
    res.status(500).json({ error: 'Unable to retrieve NASA data' });
  }
});

app.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);
});
```

#### Simple frontend `public/index.html`
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>NASA APOD Explorer</title>
</head>
<body>
  <h2>NASA Astronomy Picture of the Day</h2>
  <input type="date" id="dateInput" value="2024-01-01" />
  <button onclick="getNasaData()">Load</button>
  <p id="message"></p>
  <div id="result" hidden>
    <h3 id="title"></h3>
    <p id="date"></p>
    <p id="mediaType"></p>
    <img id="apodImage" width="600" hidden />
    <a id="mediaLink" href="#">Open media</a>
    <p id="explanation"></p>
  </div>

  <script>
    const d = document.getElementById('dateInput');
    const m = document.getElementById('message');
    const r = document.getElementById('result');
    const t = document.getElementById('title');
    const dt = document.getElementById('date');
    const mt = document.getElementById('mediaType');
    const i = document.getElementById('apodImage');
    const l = document.getElementById('mediaLink');
    const e = document.getElementById('explanation');

    async function getNasaData() {
      m.innerText = 'Loading...';
      r.hidden = true;
      try {
        const res = await fetch(`/api/apod?date=${d.value}`);
        const data = await res.json();
        if (!res.ok) throw new Error(data.error || 'Fetch failed');

        t.innerText = data.title;
        dt.innerText = data.date;
        mt.innerText = data.media_type;
        e.innerText = data.explanation;

        if (data.media_type === 'image') {
          i.src = data.url;
          i.hidden = false;
          l.href = data.hdurl || data.url;
          l.innerText = 'Open Image';
        } else {
          i.hidden = true;
          l.href = data.url;
          l.innerText = 'Open Video';
        }

        m.innerText = 'NASA data loaded successfully.';
        r.hidden = false;
      } catch (err) {
        m.innerText = err.message;
      }
    }
  </script>
</body>
</html>
```

### Backup if fetch fails
```bash
npm install node-fetch
```
Then replace `fetch` with `node-fetch` import if needed.

---

## 5) Experiment 2C: RESTful API with CRUD and Postman Testing

### Aim
Build a CRUD API using Express.js, validate request data, and test with Postman.

### Working compatible code
```js
const express = require('express');
const Joi = require('joi');
const app = express();
const PORT = 8080;

app.use(express.json());

const movieSchema = Joi.object({
  title: Joi.string().required(),
  genre: Joi.string().required(),
  director: Joi.string().required(),
  releaseYear: Joi.number().integer().min(1900).max(2100).required(),
  rating: Joi.number().min(0).max(10).required(),
  status: Joi.string().required()
});

let movies = [
  {
    id: 1,
    title: 'Interstellar',
    genre: 'Science Fiction',
    director: 'Christopher Nolan',
    releaseYear: 2014,
    rating: 9,
    status: 'Watched'
  },
  {
    id: 2,
    title: 'The Martian',
    genre: 'Science Fiction',
    director: 'Ridley Scott',
    releaseYear: 2015,
    rating: 8.5,
    status: 'Favourite'
  }
];

app.get('/', (req, res) => {
  res.send('Welcome to the CineTrack Movie Management API');
});

app.get('/api/movies', (req, res) => {
  res.status(200).json(movies);
});

app.get('/api/movies/:id', (req, res) => {
  const movieId = parseInt(req.params.id);
  const movie = movies.find(m => m.id === movieId);
  if (!movie) return res.status(404).json({ message: 'Movie not found' });
  res.status(200).json(movie);
});

app.post('/api/movies', (req, res) => {
  const { error, value } = movieSchema.validate(req.body);
  if (error) {
    return res.status(400).json({ error: error.details[0].message });
  }

  const newMovie = { id: movies.length ? movies[movies.length - 1].id + 1 : 1, ...value };
  movies.push(newMovie);
  res.status(201).json(newMovie);
});

app.put('/api/movies/:id', (req, res) => {
  const movieId = parseInt(req.params.id);
  const index = movies.findIndex(m => m.id === movieId);
  if (index === -1) return res.status(404).json({ message: 'Movie not found' });

  const { error, value } = movieSchema.validate(req.body);
  if (error) return res.status(400).json({ error: error.details[0].message });

  movies[index] = { ...movies[index], ...value };
  res.status(200).json(movies[index]);
});

app.delete('/api/movies/:id', (req, res) => {
  const movieId = parseInt(req.params.id);
  const index = movies.findIndex(m => m.id === movieId);
  if (index === -1) return res.status(404).json({ message: 'Movie not found' });

  const deletedMovie = movies.splice(index, 1)[0];
  res.status(200).json({ message: 'Movie deleted successfully', movie: deletedMovie });
});

app.use((req, res) => {
  res.status(404).json({ error: 'API endpoint not found' });
});

app.listen(PORT, () => {
  console.log(`CineTrack API running at http://localhost:${PORT}`);
});
```

### Install required package
```bash
npm install express joi
```

### Postman tests
- GET `http://localhost:8080/api/movies`
- GET `http://localhost:8080/api/movies/1`
- POST to `http://localhost:8080/api/movies` with JSON body
- PUT to same endpoint with updated object
- DELETE `http://localhost:8080/api/movies/1`

### Example POST body
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

### Failure backup
If Postman not available, use curl:
```bash
curl -X POST http://localhost:8080/api/movies -H "Content-Type: application/json" -d '{"title":"Avatar","genre":"Sci-Fi","director":"James Cameron","releaseYear":2009,"rating":8.8,"status":"Favourite"}'
```

---

## 6) Experiment 3: Firebase Cloud Services Integration

### Aim
Integrate Firebase Storage, Hosting, Authentication, and Firestore Database.

### Setup steps
1. Go to Firebase Console.
2. Create project.
3. Add web app.
4. Enable Authentication -> Google sign in.
5. Create Firestore Database.
6. Enable Firebase Storage.
7. Install Firebase SDK in Node.js project.

### Install
```bash
npm init -y
npm install firebase
```

### `firebase.js`
```js
const { initializeApp } = require('firebase/app');
const { getFirestore } = require('firebase/firestore');
const { getStorage } = require('firebase/storage');
const { getAuth, GoogleAuthProvider } = require('firebase/auth');

const firebaseConfig = {
  apiKey: 'YOUR_API_KEY',
  authDomain: 'YOUR_PROJECT.firebaseapp.com',
  projectId: 'YOUR_PROJECT_ID',
  storageBucket: 'YOUR_PROJECT.appspot.com',
  messagingSenderId: 'YOUR_SENDER_ID',
  appId: 'YOUR_APP_ID'
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const storage = getStorage(app);
const auth = getAuth(app);
const provider = new GoogleAuthProvider();

module.exports = { db, storage, auth, provider };
```

### Firestore add/get code
```js
const { db } = require('./firebase');
const { collection, addDoc, getDocs } = require('firebase/firestore');

async function saveStudent(student) {
  await addDoc(collection(db, 'students'), student);
}

async function getStudents() {
  const snapshot = await getDocs(collection(db, 'students'));
  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}
```

### Express + Firebase CRUD form template
```js
const express = require('express');
const app = express();
app.use(express.json());

const { db } = require('./firebase');
const { collection, addDoc, getDocs, doc, updateDoc, deleteDoc, getDoc } = require('firebase/firestore');

app.get('/students', async (req, res) => {
  const snapshot = await getDocs(collection(db, 'students'));
  res.json(snapshot.docs.map(d => ({ id: d.id, ...d.data() })));
});

app.post('/students', async (req, res) => {
  const docRef = await addDoc(collection(db, 'students'), req.body);
  res.status(201).json({ id: docRef.id, ...req.body });
});

app.put('/students/:id', async (req, res) => {
  const ref = doc(db, 'students', req.params.id);
  await updateDoc(ref, req.body);
  res.json({ message: 'Updated' });
});

app.delete('/students/:id', async (req, res) => {
  await deleteDoc(doc(db, 'students', req.params.id));
  res.json({ message: 'Deleted' });
});

app.listen(3000, () => console.log('Firebase app running on 3000'));
```

### Common failure fixes
- Firebase config keys missing -> check web app settings.
- Firestore permissions blocked -> set rules to allow read/write for testing.
- Storage not enabled -> enable Storage from Firebase Console.

### Backup test rule
```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;
    }
  }
}
```

---

## 7) Experiment 4: Dropbox API Storage as a Service

### Aim
Upload files securely to Dropbox using the Dropbox API.

### Setup commands
```bash
pip install dropbox
```

### Python code template
```python
import dropbox

TOKEN = 'YOUR_DROPBOX_ACCESS_TOKEN'

dbx = dropbox.Dropbox(TOKEN)

local_path = r'C:\path\to\file.txt'
dropbox_path = '/myfile.txt'

with open(local_path, 'rb') as f:
    dbx.files_upload(f.read(), dropbox_path)

print('File uploaded to Dropbox successfully!')
```

### List files backup
```python
import dropbox

TOKEN = 'YOUR_DROPBOX_ACCESS_TOKEN'
dbx = dropbox.Dropbox(TOKEN)

for entry in dbx.files_list_folder('').entries:
    print(entry.name)
```

### Failure backup
- If token invalid: create a new app in Dropbox Developer Console.
- If permission denied: set app permissions to Files and folders -> All file types or app folder.
- If upload fails: check path and file accessibility.

---

## 8) Experiment 5: MongoDB Atlas with CRUD and Node.js

### Aim
Connect an app to MongoDB Atlas and perform CRUD operations.

### Requirements
- MongoDB Atlas account
- M0 free cluster
- Database user and IP access

### Commands
```bash
npm init -y
npm install express mongoose cors
```

### `.env`
```env
MONGODB_URI=mongodb+srv://<username>:<password>@cluster0.xxxxxx.mongodb.net/codesnipper?retryWrites=true&w=majority
PORT=5000
```

### App code template
```js
import express from 'express';
import mongoose from 'mongoose';
import cors from 'cors';

const app = express();
app.use(cors());
app.use(express.json());

const mongoUri = process.env.MONGODB_URI;

mongoose.connect(mongoUri)
  .then(() => console.log('Connected to MongoDB Atlas successfully!'))
  .catch((error) => console.error('MongoDB connection failed:', error));

const snippetSchema = new mongoose.Schema({
  title: String,
  language: String,
  code: String,
  createdAt: { type: Date, default: Date.now }
});

const Snippet = mongoose.model('Snippet', snippetSchema);

app.get('/snippets', async (req, res) => {
  const snippets = await Snippet.find();
  res.json(snippets);
});

app.post('/snippets', async (req, res) => {
  const snippet = new Snippet(req.body);
  await snippet.save();
  res.status(201).json(snippet);
});

app.get('/snippets/:id', async (req, res) => {
  const snippet = await Snippet.findById(req.params.id);
  res.json(snippet);
});

app.put('/snippets/:id', async (req, res) => {
  const snippet = await Snippet.findByIdAndUpdate(req.params.id, req.body, { new: true });
  res.json(snippet);
});

app.delete('/snippets/:id', async (req, res) => {
  const snippet = await Snippet.findByIdAndDelete(req.params.id);
  res.json({ message: 'Deleted', snippet });
});

app.listen(5000, () => console.log('Server running on 5000'));
```

### Frontend HTML fetch template
```html
<!DOCTYPE html>
<html>
<body>
  <h2>CodeSnipper CRUD</h2>
  <input id="title" placeholder="Title" />
  <input id="language" placeholder="Language" />
  <textarea id="code" placeholder="Code"></textarea>
  <button onclick="saveSnippet()">Save</button>
  <button onclick="loadSnippets()">Load</button>
  <div id="output"></div>

  <script>
    async function saveSnippet() {
      const body = {
        title: document.getElementById('title').value,
        language: document.getElementById('language').value,
        code: document.getElementById('code').value
      };

      const res = await fetch('http://localhost:5000/snippets', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body)
      });
      const data = await res.json();
      console.log(data);
      loadSnippets();
    }

    async function loadSnippets() {
      const res = await fetch('http://localhost:5000/snippets');
      const data = await res.json();
      document.getElementById('output').innerHTML = data.map(item => `<p><strong>${item.title}</strong><br>${item.code}</p>`).join('');
    }
  </script>
</body>
</html>
```

### Common issues and backups
- If Atlas connection fails: check username/password and IP whitelist.
- If `MONGODB_URI` not found: use `.env` and `dotenv` package.
- If app uses old MongoDB driver syntax, use `mongoose.connect()` with proper options.

---

## 9) Experiment 6: Auth0 SSO Implementation

### Aim
Allow one login for multiple applications using Auth0.

### Install
```bash
npm install @auth0/auth0-react
```

### React main file template
```jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import { Auth0Provider } from '@auth0/auth0-react';

ReactDOM.createRoot(document.getElementById('root')).render(
  <Auth0Provider
    domain="YOUR_AUTH0_DOMAIN"
    clientId="YOUR_AUTH0_CLIENT_ID"
    authorizationParams={{
      redirect_uri: window.location.origin
    }}
  >
    <App />
  </Auth0Provider>
);
```

### App component template
```jsx
import React from 'react';
import { useAuth0 } from '@auth0/auth0-react';

function App() {
  const { loginWithRedirect, logout, user, isAuthenticated, isLoading } = useAuth0();

  if (isLoading) return <p>Loading...</p>;

  return (
    <div>
      {isAuthenticated ? (
        <>
          <h2>Welcome {user?.name}</h2>
          <p>Email: {user?.email}</p>
          <button onClick={() => logout({ returnTo: window.location.origin })}>Logout</button>
        </>
      ) : (
        <button onClick={() => loginWithRedirect()}>Login with Auth0</button>
      )}
    </div>
  );
}

export default App;
```

### Auth0 dashboard settings
- Allowed Callback URLs: `http://localhost:5173` or `/callback`
- Allowed Logout URLs: `http://localhost:5173`
- Allowed Web Origins: `http://localhost:5173`

### Backend JWT validation (Express)
```js
const express = require('express');
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');
const app = express();

const client = jwksClient({
  jwksUri: 'https://YOUR_AUTH0_DOMAIN/.well-known/jwks.json'
});

function getKey(header, callback) {
  client.getSigningKey(header.kid, function(err, key) {
    const signingKey = key.publicKey || key.rsaPublicKey;
    callback(null, signingKey);
  });
}

app.get('/api/profile', (req, res) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Token missing' });

  jwt.verify(token, getKey, { algorithms: ['RS256'] }, (err, decoded) => {
    if (err) return res.status(401).json({ error: 'Invalid token' });
    res.json({ user: decoded });
  });
});

app.listen(4000, () => console.log('Auth backend running on 4000'));
```

### Install backend packages
```bash
npm install express jsonwebtoken jwks-rsa
```

### Backup if Auth0 not configured
- Use dummy JWT tokens for testing only.
- Check domain and client ID.
- Ensure callback URLs exactly match.

---

## 10) Experiment 7: Google Cloud Computing Foundations and Skill Badges

### Aim
Understand Google Cloud networking, VM setup, SQL, load balancing, and app dev environment.

### Useful commands
```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
gcloud services list
gcloud compute instances list
```

### Create VM
```bash
gcloud compute instances create vm-demo \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

### SSH to VM
```bash
gcloud compute ssh vm-demo --zone=us-central1-a
```

### Create VPC
```bash
gcloud compute networks create custom-vpc --subnet-mode=custom
gcloud compute networks subnets create subnet-1 --network=custom-vpc --range=10.0.0.0/24 --region=us-central1
```

### Allow firewall rule
```bash
gcloud compute firewall-rules create allow-http \
  --network custom-vpc \
  --allow tcp:80,tcp:443 \
  --source-ranges 0.0.0.0/0
```

### Backup if project not active
```bash
gcloud projects list
gcloud config set project YOUR_PROJECT_ID
```

### Typical result
- Secure network built
- VM created
- firewall configured
- load balancer and app dev environment configured

---

## 11) Experiment 8: Hadoop and MapReduce Word Count

### Aim
Install Hadoop 3.x and run a word count MapReduce job.

### Ubuntu commands
```bash
sudo apt update
sudo apt install -y openjdk-11-jdk
java -version
```

### Edit `.bashrc`
```bash
nano ~/.bashrc
```
Add:
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
Then:
```bash
source ~/.bashrc
```

### Download and extract Hadoop
```bash
cd ~/Downloads
wget https://downloads.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
tar -xzf hadoop-3.3.6.tar.gz
sudo mv hadoop-3.3.6 /usr/local/hadoop
```

### `hadoop-env.sh`
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

### `core-site.xml`
```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
</configuration>
```

### `hdfs-site.xml`
```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/usr/local/hadoop/hadoop_data/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/usr/local/hadoop/hadoop_data/hdfs/datanode</value>
  </property>
</configuration>
```

### `mapred-site.xml`
```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

### `yarn-site.xml`
```xml
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```

### Run Hadoop
```bash
hdfs namenode -format
start-dfs.sh
start-yarn.sh
jps
```

### Put input file
```bash
mkdir -p ~/input
nano ~/input/file.txt
hdfs dfs -mkdir -p /input
hdfs dfs -put ~/input/file.txt /input/
```

### WordCount Java program
```java
import java.io.IOException;
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
            String[] words = value.toString().split("\\s+");
            for (String w : words) {
                word.set(w);
                context.write(word, one);
            }
        }
    }

    public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        private IntWritable result = new IntWritable();

        public void reduce(Text key, Iterable<IntWritable> values, Context context)
                throws IOException, InterruptedException {
            int sum = 0;
            for (IntWritable val : values) {
                sum += val.get();
            }
            result.set(sum);
            context.write(key, result);
        }
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");
        job.setJarByClass(WordCount.class);
        job.setMapperClass(TokenizerMapper.class);
        job.setReducerClass(IntSumReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

### Compile and run
```bash
javac -classpath $(hadoop classpath) WordCount.java
jar cf wc.jar WordCount*.class
hadoop jar wc.jar WordCount /input/file.txt /output/wc
hdfs dfs -cat /output/wc/part-r-00000
```

### Failure backups
- If port 9000 already in use: stop old daemons and format again.
```bash
stop-dfs.sh
stop-yarn.sh
```
- If Java mismatch: ensure `JAVA_HOME` correctly points to installed JDK.
```bash
readlink -f $(which java)
```

---

## 12) Experiment 9: MPI and OpenMP Parallel Programming

### Aim
Use MPI for process-based parallelism and OpenMP for thread-based parallelism.

### Install MPI
```bash
sudo apt update
sudo apt install -y openmpi-bin libopenmpi-dev
mpirun --version
```

### MPI Hello World code
```c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    printf("Hello from process %d of %d\n", rank, size);

    MPI_Finalize();
    return 0;
}
```

### Compile and run MPI
```bash
mpicc mpi_hello.c -o mpi_hello
mpirun -np 4 ./mpi_hello
```

### OpenMP sample
```c
#include <omp.h>
#include <stdio.h>

int main() {
    #pragma omp parallel
    {
        int tid = omp_get_thread_num();
        printf("Hello from thread %d\n", tid);
    }
    return 0;
}
```

### Compile OpenMP
```bash
gcc -fopenmp omp_hello.c -o omp_hello
./omp_hello
```

### Common OpenMP features
- `parallel` block
- `for` loop parallelism
- `reduction` for sum
- `private` / `firstprivate` / `lastprivate`
- `critical` section

### Example reduction
```c
#include <stdio.h>
#include <omp.h>

int main() {
    int sum = 0;
    int arr[10] = {1,2,3,4,5,6,7,8,9,10};

    #pragma omp parallel for reduction(+:sum)
    for (int i = 0; i < 10; i++) {
        sum += arr[i];
    }

    printf("Sum = %d\n", sum);
    return 0;
}
```

### Backup for missing MPI/OpenMP packages
```bash
sudo apt install -y openmpi-bin libopenmpi-dev libomp-dev
```

---

## 13) Experiment 10: Apache Spark on HDFS

### Aim
Install Spark, configure PATH, and run word count using RDD.

### Install Java and Scala
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk scala
java -version
scala -version
```

### Download Spark
```bash
cd ~/Downloads
wget https://dlcdn.apache.org/spark/spark-4.2.0/spark-4.2.0-bin-hadoop3.tgz
tar -xvf spark-4.2.0-bin-hadoop3.tgz
sudo mv spark-4.2.0-bin-hadoop3 /usr/local/spark
```

### Configure environment
```bash
nano ~/.bashrc
```
Add:
```bash
export SPARK_HOME=/usr/local/spark
export PATH=$PATH:$SPARK_HOME/bin
```
Then:
```bash
source ~/.bashrc
```

### Verify Spark
```bash
spark-shell
```

### Create input file
```bash
nano input.txt
cat input.txt
```

### Spark word count example
```scala
val inputFile = sc.textFile("input.txt")
val counts = inputFile.flatMap(line => line.split("\\s+")).map(word => (word, 1)).reduceByKey(_ + _)
counts.collect().foreach(println)
counts.saveAsTextFile("output")
```

### View output
```bash
ls output
cat output/part-00000
```

### Failure backup
If `spark-shell` is not found:
```bash
export SPARK_HOME=/usr/local/spark
export PATH=$PATH:$SPARK_HOME/bin
spark-shell
```

---

## 14) Experiment 11A: CloudSim Installation

### Aim
Install CloudSim 5.0 in Eclipse and add Apache Commons Math libraries.

### Steps
1. Download CloudSim 5.0.
2. Extract zip file.
3. Open Eclipse.
4. Create Java project named `CloudSim`.
5. Right click project -> Properties -> Java Build Path -> Libraries.
6. Add external JARs from `/commons-math/` folder.
7. Ensure libs appear under `Referenced Libraries`.
8. Open examples package and run sample simulation.

### Example import in Java project
```java
import org.cloudbus.cloudsim.*;
```

### Common issue
If project cannot compile: ensure JARs are added in build path and Java version supported by CloudSim is used.

### Backup
- Re-import project by selecting `CloudSim` folder as existing project.
- Ensure Maven or Gradle not required if project is copied as plain Java project.

---

## 15) Experiment 11B: FCFS Resource Allocation in CloudSim

### Aim
Implement a simple first-come-first-serve resource allocation algorithm in CloudSim.

### Create Java class in package
```java
package org.cloudbus.cloudsim.examples;
```

### Working CloudSim code template
```java
import java.text.DecimalFormat;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.LinkedList;
import java.util.List;

import org.cloudbus.cloudsim.Cloudlet;
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

public class resourcealloc {
    private static List<Cloudlet> cloudletList;
    private static List<Vm> vmlist;

    public static void main(String[] args) {
        Log.printLine("Starting FCFS Resource Allocation...");

        try {
            int numUser = 1;
            Calendar calendar = Calendar.getInstance();
            boolean traceFlag = false;
            CloudSim.init(numUser, calendar, traceFlag);

            Datacenter datacenter = createDatacenter("Datacenter_0");
            DatacenterBroker broker = createBroker();
            int brokerId = broker.getId();

            vmlist = new ArrayList<Vm>();
            int vmCount = 4;
            for (int i = 0; i < vmCount; i++) {
                int vmid = i;
                int mips = 1000;
                long size = 10000;
                int ram = 512;
                long bw = 1000;
                int pesNumber = 1;
                String vmm = "Xen";

                Vm vm = new Vm(vmid, brokerId, mips, pesNumber, ram, bw, size, vmm,
                        new CloudletSchedulerTimeShared());
                vmlist.add(vm);
            }

            broker.submitVmList(vmlist);

            cloudletList = new ArrayList<Cloudlet>();
            int cloudletCount = 8;
            long length = 400000;
            long fileSize = 300;
            long outputSize = 300;
            UtilizationModel utilizationModel = new UtilizationModelFull();

            for (int i = 0; i < cloudletCount; i++) {
                Cloudlet cloudlet = new Cloudlet(i, length, 1, fileSize, outputSize,
                        utilizationModel, utilizationModel, utilizationModel);
                cloudlet.setUserId(brokerId);
                cloudletList.add(cloudlet);
            }

            broker.submitCloudletList(cloudletList);
            CloudSim.startSimulation();
            CloudSim.stopSimulation();

            List<Cloudlet> resultList = broker.getCloudletReceivedList();
            printCloudletList(resultList);
            Log.printLine("FCFS Resource Allocation simulation finished!");
        } catch (Exception e) {
            e.printStackTrace();
            Log.printLine("An error occurred during simulation.");
        }
    }

    private static Datacenter createDatacenter(String name) {
        List<Host> hostList = new ArrayList<Host>();

        for (int i = 0; i < 4; i++) {
            int hostId = i;
            int ram = 2048;
            long storage = 1000000;
            int bw = 10000;
            List<Pe> peList = new ArrayList<Pe>();
            peList.add(new Pe(0, new PeProvisionerSimple(1000)));

            Host host = new Host(hostId, new RamProvisionerSimple(ram),
                    new BwProvisionerSimple(bw), storage, peList,
                    new VmSchedulerTimeShared(peList));
            hostList.add(host);
        }

        String arch = "x86";
        String os = "Linux";
        String vmm = "Xen";
        double timeZone = 10.0;
        double cost = 3.0;
        double costPerMem = 0.05;
        double costPerStorage = 0.001;
        double costPerBw = 0.0;

        LinkedList<Storage> storageList = new LinkedList<Storage>();
        DatacenterCharacteristics characteristics = new DatacenterCharacteristics(
                arch, os, vmm, hostList, timeZone, cost, costPerMem, costPerStorage, costPerBw);

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
        }
        return broker;
    }

    private static void printCloudletList(List<Cloudlet> list) {
        int size = list.size();
        String indent = "  ";
        Log.printLine();
        Log.printLine("========== FCFS RESOURCE ALLOCATION OUTPUT ==========");
        Log.printLine("Cloudlet ID" + indent + "STATUS" + indent + "Data center ID" + indent + "VM ID" + indent + "Time" + indent + "Start Time" + indent + "Finish Time");

        DecimalFormat dft = new DecimalFormat("###.##");

        for (int i = 0; i < size; i++) {
            Cloudlet cloudlet = list.get(i);
            Log.print(indent + cloudlet.getCloudletId() + indent + indent);

            if (cloudlet.getCloudletStatus() == Cloudlet.SUCCESS) {
                Log.print("SUCCESS");
                Log.printLine(
                    indent + indent + cloudlet.getResourceId() +
                    indent + indent + cloudlet.getVmId() +
                    indent + indent + dft.format(cloudlet.getActualCPUTime()) +
                    indent + indent + dft.format(cloudlet.getExecStartTime()) +
                    indent + indent + dft.format(cloudlet.getFinishTime())
                );
            }
        }
    }
}
```

### Additional extra-check
- If CloudSim examples do not compile, re-add the whole `commons-math` jar set and ensure you are using Java 8/11 in Eclipse.

---

## 16) Experiment 12: CloudAnalyst

### Aim
Install CloudAnalyst using NetBeans and run simulation.

### Steps
1. Download CloudAnalyst for NetBeans.
2. Install NetBeans.
3. Open project in NetBeans.
4. Run project.
5. Configure simulation parameters.
6. Click Run Simulation.
7. Observe results.

### Common issue
If the project shows build error, open the project directly from the folder and let NetBeans rebuild classes.

### Backup command sequence
```bash
# If using Windows:
# 1. Install JDK
# 2. Install NetBeans
# 3. Open CloudAnalyst project
# 4. Clean and Build Project
```

---

# 17) General Lab Viva/Submission Tips

### How to write the result section
Use a fixed format:
```text
Thus, the experiment was successfully completed. The required setup, configuration, and execution were performed. The output was verified, and the expected result was achieved.
```

### How to write aim section
```text
Aim: To implement/connect/configure the required cloud service or application using the given platform and verify the output.
```

### How to explain common failure conditions
- Missing Java/Node/Python package
- Wrong PATH variable
- Firewalls or IP not allowed
- Invalid API token/key
- Port busy

### Common debugging commands
```bash
java -version
node -v
npm -v
python --version
pip --version
jps
ss -tulpn | grep -E '3000|5000|8080|9000'
```

### If port is busy
Linux/macOS:
```bash
lsof -i :3000
kill -9 <PID>
```
Windows PowerShell:
```powershell
netstat -ano | findstr :3000
Stop-Process -Id <PID>
```

---

# 18) One-Page Official Lab Template You Can Copy

```text
Aim:
To [briefly state the intended objective].

Requirements:
- Software / SDK / Service required
- Platform / environment

Procedure:
1. Install required software.
2. Configure environment variables or project settings.
3. Create or import the project.
4. Write or run the required code.
5. Execute the task and verify output.
6. Record screenshots / result.

Code / Commands:
[Paste the required code or commands here]

Output:
[Write the actual observed output]

Result:
Thus, the experiment was successfully executed and the required objective was achieved.
```

---

# 19) Final Notes

This cheat sheet reconstructs the likely working lab content from the PDFs and closes the missing gaps. For final submission, use the exact experiment name, the procedure steps, and the code templates above.

If a PDF page is partially cut off, use the closest correct standard implementation pattern shown here.

Recommended approach for the lab record:
- Keep it concise.
- Add commands and relevant code snippets.
- Add a short result paragraph.
- Include screenshots where possible.
- Use the nearest working verified template from this file.

---

Last updated: 2026-10-11
