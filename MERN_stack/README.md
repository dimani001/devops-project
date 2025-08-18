## MERN Stack Todo Application

This project is a MERN stack Todo app deployed on an AWS EC2 Ubuntu server with MongoDB Atlas as the database. It covers Node.js + Express backend, MongoDB for persistence, and a React frontend. It shows how to build a simple Todo list where you can add, view, and delete tasks.
It uses:
Express.js (Backend API) → handles data requests
MongoDB (Database) → stores todos
React.js (Frontend) → displays the app in the browser

## Prerequisites
- An AWS EC2 Ubuntu 24.04 server
- A MongoDB Atlas account (for storing todos online)
- A Postman account (to test API requests)
- Node.js & npm installed (to run JavaScript apps)
- A Security Group (Steghub-SG) with these inbound rules:
. Port 22 → SSH (connect to server)
. Port 80 → HTTP
. Port 3000 → React frontend
. Port 5000 → Express backend

## Steps to Deploy

1. Connect to AWS EC2 Instance

```bash
cd /c/Users/H.P\ I5\ 8TH\ GEN/Downloads/
ssh -i steghub.pem ubuntu@13.48.133.133
```

Reason: This command securely connects your local machine to the EC2 instance using your SSH key.
cd ... → go to the folder where your key file (steghub.pem) is stored
ssh -i ... → log into your EC2 server using that key

2. Update Packages

```bash
sudo apt update
sudo apt upgrade
```

Reason: Ensures all existing packages are up-to-date for security and compatibility.

3. Install Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v 
npm -v 
```

Reason: Installs Node.js runtime and npm (package manager) to run backend JavaScript code; the last two commands confirm versions.

4. Create Project Directory

```bash
mkdir Todo
ls
ls -lih
cd Todo
npm init
```
Reason: Creates a folder for the app and initializes a Node.js project (package.json).
mkdir Todo → create a folder for your app
ls / ls -lih → list files (regular and detailed views) to confirm it exists
cd Todo → move into the project folder
npm init → create a package.json (project config)

5. Upgrade Node.js to v20 (latest)

```bash
sudo apt remove -y nodejs npm
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
npm install -g npm@11.5.2
npm -v
```

Reason: Removes old version and installs the latest Node.js + npm for better compatibility with React.

6. Install Express & Setup 

```bash
npm install express
touch index.js
npm install dotenv
```

Reason: Express is used to build APIs. dotenv allows environment variable management.

Create index.js:
```js
const express = require('express');
require('dotenv').config();

const app = express();
const port = process.env.PORT || 5000;

app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
  next();
});

app.use((req, res, next) => {
  res.send('Welcome to Express');
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`)
});
```

Run:

```bash
node index.js
```

Reason: Starts a simple Express server on port 5000.

7. Open Port in AWS Security Group

Go to AWS Console → EC2 → Security Groups.

Add inbound rule: Custom TCP, Port 5000, Source 0.0.0.0/0

Reason: Allows external access to the server on port 5000.
 Test on your browser: http://13.62.50.189:5000/

 8. Create Routes

 ```bash
mkdir routes
cd routes
touch api.js
```

Open routes/api.js and paste:

```js
const express = require ('express');
const router = express.Router();

router.get('/todos', (req, res, next) => {});
router.post('/todos', (req, res, next) => {});
router.delete('/todos/:id', (req, res, next) => {});

module.exports = router;
```

Reason: Sets up GET, POST, DELETE endpoints for todos.

9. # Add MongoDB (models) and Mongoose
```bash
cd ..
npm install mongoose
mkdir models
ls
cd models
touch todo.js
```

Open models/todo.js and paste:
```js
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const TodoSchema = new Schema({
  action: {
    type: String,
    required: [true, 'The todo text field is required']
  }
});

const Todo = mongoose.model('todo', TodoSchema);
module.exports = Todo;
```
Reason: Defines Todo schema & connects Express routes to MongoDB.


```bash
cd ..
cd routes
vim api.js
```
Replace routes/api.js
```js
const express = require ('express');
const router = express.Router();
const Todo = require('../models/todo');

router.get('/todos', (req, res, next) => {

//this will return all the data, exposing only the id and action field to the client
Todo.find({}, 'action')
.then(data => res.json(data))
.catch(next)
});

router.post('/todos', (req, res, next) => {
if(req.body.action){
  Todo.create(req.body)
  .then(data => res.json(data))
  .catch(next)
}else {
  res.json({
    error: "The input field is empty"
  })
}
});

router.delete('/todos/:id', (req, res, next) => {
  Todo.findOneAndDelete({"_id": req.params.id})
  .then(data => res.json(data))
  .catch(next)
})

module.exports = router;
```
Why:
-GET returns all todos (only _id and action)
-POST creates a new todo if action is provided
-DELETE removes a todo by its ID

# Setup MongoDB Atlas (Cloud Database)

We’ll use MongoDB Atlas instead of installing MongoDB locally.

Create Cluster

# Sign up at MongoDB Atlas.

Create a free M0 cluster → select AWS as provider.

Choose a region near your EC2 (e.g., eu-north-1 Stockholm).

# Create DB User

Username: todo_user

Password: todo123456

Role: Read/Write

Reason: This is the user your Node app will use to connect.

## Configure Network Access

Go to Network Access → Add IP Address → 0.0.0.0/0
Reason: Allow access from anywhere (for testing). Later, restrict to EC2 public IP.

Get Connection URI

In Atlas → Connect → Connect your application.
Example URI:
```bash
mongodb+srv://todo_user:todo123456@todo-cluster.1pggpkn.mongodb.net/todoDB?retryWrites=true&w=majority&appName=Todo-Cluster
```

# Create the environment file for MongoDB Atlas
Save in .env
```bash
touch .env
vi .env
```
Paste:

```bash
DB=mongodb+srv://todo_user:todo123456@todo-cluster.1pggpkn.mongodb.net/todoDB?retryWrites=true&w=majority&appName=Todo-Cluster
```

Reason: Keeps secrets out of source code.

# Replace index.js with the DB-connected server

```bash
vim index.js
```
Replace the entire file with:

```js
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const routes = require('./routes/api');
const path = require('path');
require('dotenv').config();

const app = express();

const port = process.env.PORT || 5000;

//connect to the database
mongoose.connect(process.env.DB, { useNewUrlParser: true, useUnifiedTopology: true })
.then(() => console.log(`Database connected successfully`))
.catch(err => console.log(err));

//since mongoose promise is depreciated, we overide it with node's promise
mongoose.Promise = global.Promise;

app.use((req, res, next) => {
res.header("Access-Control-Allow-Origin", "*");
res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
next();
});

app.use(bodyParser.json());

app.use('/api', routes);

app.use((err, req, res, next) => {
console.log(err);
next();
});

app.listen(port, () => {
console.log(`Server running on port ${port}`)
});
```
Run:
```bash
node index.js
```
Why: Connects to MongoDB Atlas and exposes /api/todos endpoints that your frontend and Postman can use.

12) # Test the API with Postman (start from signup)

-Sign up and install
- Go to https://www.postman.com/
- Create a free account
- Install the Postman desktop app (or use the web version)
A)  POST – Create a new todo
-Method: POST
-URL: http://13.51.157.36:5000/api/todos
Headers:
-Content-Type: application/json
-Body → raw → JSON:
```json
{
  "action": "Finish project 8 and 9"
}
```
Reason: Sends a new todo to your API; it gets saved in MongoDB and returns the created object (with _id).

B) GET – Retrieve all todos
- Method: GET
- URL: http://13.51.157.36:5000/api/todos
-Body: none
Reason:Fetches a list of all todos currently stored in your database.

C) DELETE – Remove one todo by its ID
- Method: DELETE
- URL: http://13.51.157.36:5000/api/todos/68a24a1af925444a201a51d7
- (Replace the ID with a real one from your GET response.)
Expected sample response:

```json
{
  "_id": "68a24a1af925444a201a51d7",
  "action": "Finish project 8 and 9",
  "__v": 0
}
```
Reason: Deletes exactly one todo; a new GET will no longer show it.

13) Create the React frontend

From your Todo directory:

```bash
npx create-react-app client
npm install concurrently --save-dev
npm install nodemon --save-dev
```
Reason: Sets up the React app in a client folder and installs tools to run client + server together.

Open Todo/package.json and set scripts:

```json
{
  "name": "todo",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "start-watch": "nodemon index.js",
    "dev": "concurrently \"npm run start-watch\" \"cd client && npm start\""
  },
  "author": "ANI CHINECHEREM C",
  "license": "ISC",
  "description": "",
  "dependencies": {
    "dotenv": "^17.2.1",
    "express": "^5.1.0",
    "mongoose": "^8.17.1"
  },
  "devDependencies": {
    "concurrently": "^9.2.0",
    "nodemon": "^3.1.10"
  }
}
```
Reason: npm run dev starts both the backend (port 5000) and the React app (port 3000).

14) Configure the React proxy

```bash
cd client
vi package.json
```
Add this key/value:
```bash
"proxy": "http://localhost:5000"
```
Reason: Lets the React app call /api/todos without writing the full server URL.
React will forward those calls to http://localhost:5000.

Go back:
```bash
cd ..
```

Run both:

```bash
npm run dev
```
Reason: Launches your backend and frontend together.

The React app typically opens at http://localhost:3000.

To access from the internet, open TCP 3000 in your EC2 Security Group (Steghub-SG).

15) Build React components

From Todo:
```bash
cd client
cd src
mkdir components
cd components
touch Input.js ListTodo.js Todo.js
```
Reason: Creates a components folder and the three React files used in your UI.

Open Input.js:
```js
import React, { Component } from 'react';
import axios from 'axios';

class Input extends Component {

state = {
action: ""
}

addTodo = () => {
const task = {action: this.state.action}

    if(task.action && task.action.length > 0){
      axios.post('/api/todos', task)
        .then(res => {
          if(res.data){
            this.props.getTodos();
            this.setState({action: ""})
          }
        })
        .catch(err => console.log(err))
    }else {
      console.log('input field required')
    }

}

handleChange = (e) => {
this.setState({
action: e.target.value
})
}

render() {
let { action } = this.state;
return (
<div>
<input type="text" onChange={this.handleChange} value={action} />
<button onClick={this.addTodo}>add todo</button>
</div>
)
}
}

export default Input
```
Install Axios (from project root you already did npm init etc.; here you install for client):

```bash
cd ..
cd ..
npm install axios
cd src/components
```
Reason: Axios is the HTTP client React uses to call your API.

Open ListTodo.js:
```js
import React from 'react';

const ListTodo = ({ todos, deleteTodo }) => {

return (
<ul>
{
todos &&
todos.length > 0 ?
(
todos.map(todo => {
return (
<li key={todo._id} onClick={() => deleteTodo(todo._id)}>{todo.action}</li>
)
})
)
:
(
<li>No todo(s) left</li>
)
}
</ul>
)
}

export default ListTodo
```

Open Todo.js:
```js
import React, {Component} from 'react';
import axios from 'axios';

import Input from './Input';
import ListTodo from './ListTodo';

class Todo extends Component {

state = {
todos: []
}

componentDidMount(){
this.getTodos();
}

getTodos = () => {
axios.get('/api/todos')
.then(res => {
if(res.data){
this.setState({
todos: res.data
})
}
})
.catch(err => console.log(err))
}

deleteTodo = (id) => {

    axios.delete(`/api/todos/${id}`)
      .then(res => {
        if(res.data){
          this.getTodos()
        }
      })
      .catch(err => console.log(err))

}

render() {
let { todos } = this.state;

    return(
      <div>
        <h1>My Todo(s)</h1>
        <Input getTodos={this.getTodos}/>
        <ListTodo todos={todos} deleteTodo={this.deleteTodo}/>
      </div>
    )

}
}

export default Todo;
```

16) App wrapper and styling

Move to the src folder
```bash
cd ..
vi App.js
```
Paste:
```js
import React from 'react';

import Todo from './components/Todo';
import './App.css';

const App = () => {
return (
<div className="App">
<Todo />
</div>
);
}

export default App;
```
Open App.css:
```bash
vi App.css
```
paste: 
```css
.App {
text-align: center;
font-size: calc(10px + 2vmin);
width: 60%;
margin-left: auto;
margin-right: auto;
}

input {
height: 40px;
width: 50%;
border: none;
border-bottom: 2px #101113 solid;
background: none;
font-size: 1.5rem;
color: #787a80;
}

input:focus {
outline: none;
}

button {
width: 25%;
height: 45px;
border: none;
margin-left: 10px;
font-size: 25px;
background: #101113;
border-radius: 5px;
color: #787a80;
cursor: pointer;
}

button:focus {
outline: none;
}

ul {
list-style: none;
text-align: left;
padding: 15px;
background: #171a1f;
border-radius: 5px;
}

li {
padding: 15px;
font-size: 1.5rem;
margin-bottom: 15px;
background: #282c34;
border-radius: 5px;
overflow-wrap: break-word;
cursor: pointer;
}

@media only screen and (min-width: 300px) {
.App {
width: 80%;
}

input {
width: 100%
}

button {
width: 100%;
margin-top: 15px;
margin-left: 0;
}
}

@media only screen and (min-width: 640px) {
.App {
width: 60%;
}

input {
width: 50%;
}

button {
width: 30%;
margin-left: 10px;
margin-top: 0;
}
}
```

Open index.css:
```js
vim index.css
```
paste:
```css
body {
margin: 0;
padding: 0;
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Roboto", "Oxygen",
"Ubuntu", "Cantarell", "Fira Sans", "Droid Sans", "Helvetica Neue",
sans-serif;
-webkit-font-smoothing: antialiased;
-moz-osx-font-smoothing: grayscale;
box-sizing: border-box;
background-color: #282c34;
color: #787a80;
}

code {
font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New",
monospace;
}
```

17. # Run everything and test the UI

Back to project root:
```bash
cd ../..
npm run dev
```

Open in your browser:

http://13.51.157.36:3000/


Reason: Shows the React app, which talks to your Express API, which stores data in MongoDB Atlas.


Backend (Express) → http://13.51.157.36:5000/api/todos

Frontend (React) → http://13.51.157.36:3000

Database (MongoDB Atlas) → stores your todos in then:



