## Book Register – MEAN Stack Project (Ubuntu 22.04 on AWS EC2)

A step-by-step, README that documents *everything I did* to build and troubleshoot a simple MEAN-stack style app (MongoDB + Express + AngularJS + Node.js). It includes **every command**, **why each was used**, **AWS/EC2 & Security Group setup notes**, and **all fixes** I applied along the way.

This guide mirrors the exact flow you followed on an **Ubuntu 22.04 ** EC2 instance, connecting from Windows, and ends with a working web app available on **TCP port 3300**.
**Book Register** is a minimal MEAN-style web app that lets you:
- Create a book (name, ISBN, author, pages)
- List all books
- Delete a book by ISBN

It uses:
- **MongoDB** for data storage
- **Express.js** for the backend API
- **AngularJS** for a quick front-end
- **Node.js** to run the server


- `server.js` starts Express, serves the frontend, and wires the routes.
- `apps/routes.js` defines `/book` CRUD endpoints.
- `apps/models/book.js` defines the Book schema.
- `public/` contains `index.html` and `script.js` (AngularJS app).

## Prerequisites

### Accounts & Cloud
- **AWS Account**
- **EC2 Instance**: Ubuntu Server **22.04 LTS (Ubuntu 24.04 LTS)**
<img width="1920" height="1080" alt="Screenshot (539)" src="https://github.com/user-attachments/assets/4fb51f9d-7aba-490b-8f00-2c8953ae91c3" />

### Local Machine (Windows)
- **SSH client** (e.g., Git Bash 
- **`.pem` key** downloaded (e.g., `steghub.pem`)
- **VS Code** (optional, for editing files locally via SSH or SFTP)

### Instance Access & Networking
- **Security Group (SG)** with inbound rules:
  - `TCP 22` (SSH) from your IP (required to connect)
  - `TCP 3300` (app port) from your IP / anywhere (for public testing)

<img width="1920" height="1080" alt="Screenshot (540)" src="https://github.com/user-attachments/assets/61aabbdb-8d38-46cb-b9f8-af1963c385bc" />

### Core Packages (Installed on EC2)
- `curl`, `gnupg`, `dirmngr`, `apt-transport-https`, `lsb-release`, `ca-certificates`
- **Node.js** (from NodeSource, v18.x used)
- **npm** (comes with Node.js from NodeSource)
- **MongoDB 7.0** (official repo)
- **AngularJS** (loaded via CDN in `index.html`)


## EC2 & OS Setup

```bash
cd /c/Users/H.P\ I5\ 8TH\ GEN/Downloads/
```

SSH into EC2 using your PEM key and public IP:
```bash
ssh -i steghub.pem ubuntu@16.170.231.38
```
<img width="1920" height="1080" alt="Screenshot (522)" src="https://github.com/user-attachments/assets/52e874ab-4686-45d5-9ace-f07a138f993c" />

Update and upgrade the instance:
```bash
sudo apt update
sudo apt upgrade
```

Install base utilities used later for secure repo setup and downloads:
```bash
sudo apt -y install curl dirmngr apt-transport-https lsb-release ca-certificates
```

# Install Node.js, npm, and MongoDB
Node.js (from NodeSource 18.x)
Using NodeSource ensures a clean, up-to-date Node.js + npm:
```bash
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```
<img width="1920" height="1080" alt="Screenshot (525)" src="https://github.com/user-attachments/assets/f16732dc-ad53-4768-ab8e-a323e0c5fe9d" />

(If needed) Ensure curl & gnupg are present
```bash
sudo apt-get install -y gnupg curl
```
<img width="1920" height="1080" alt="Screenshot (526)" src="https://github.com/user-attachments/assets/b0e0801a-7bb6-40f4-9bfd-8a35eccc659c" />

# MongoDB 7.0 (official repo)
Import MongoDB’s GPG key and set up its apt source:
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
   sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

Install MongoDB:
```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
```

Start & check status (note the service name is mongod, not mongodb):

```bash
sudo service mongod start
sudo systemctl status mongod
```
Why: mongod is the daemon (service) that runs MongoDB. Using the wrong unit name (mongodb) returns “Unit not found.”
<img width="1920" height="1080" alt="Screenshot (529)" src="https://github.com/user-attachments/assets/a01bb29d-fe2b-46ce-97c9-1b4794e2e6ac" />

Install body-parser (middleware for JSON)
```bash
sudo npm install body-parser
```
<img width="1920" height="1080" alt="Screenshot (531)" src="https://github.com/user-attachments/assets/4cfe976c-39ff-4d23-a195-a139f5e75c92" />

# Create the Project (Books)

Create and enter the project folder:
```bash
mkdir Books && cd Books
```

Initialize npm:
```bash
npm init
```

<img width="1920" height="1080" alt="Screenshot (533)" src="https://github.com/user-attachments/assets/5c961019-2084-439d-a35c-23b336ac64a4" />

 Tip: If you accidentally type shell commands inside the npm init prompt (e.g., vi server.js), npm will complain about invalid package names. If that happens, hit CTRL + C and re-run npm init -y.

 Create the server entry file:
```bash
vi server.js
```

Paste the following baseline server :
```js
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const path = require('path');

const app = express();
const PORT = process.env.PORT || 3300;

// MongoDB connection
mongoose.connect('mongodb://localhost:27017/test', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB connection error:', err));

app.use(express.static(path.join(__dirname, 'public')));
app.use(bodyParser.json());

require('./apps/routes')(app);

app.listen(PORT, () => {
  console.log(`Server up: http://localhost:${PORT}`);
});
```

Install backend libraries (locking Express to v4 — see Troubleshooting):
```bash
sudo npm install express mongoose
```

I eventually used Express v4 to avoid a path-to-regexp error found in Express v5 beta. If you ever need to force v4 explicitly:

```bash
npm install express@4 mongoose
```


# Backend: Express + Mongoose

Create folder structure for routes and models:

```bash
mkdir apps && cd apps
vi routes.js
```
Paste:

```js
const Book = require('./models/book');
const path = require('path');

module.exports = function(app) {
  app.get('/book', async (req, res) => {
    try {
      const books = await Book.find();
      res.json(books);
    } catch (err) {
      res.status(500).json({ message: 'Error fetching books', error: err.message });
    }
  });

  app.post('/book', async (req, res) => {
    try {
      const book = new Book({
        name: req.body.name,
        isbn: req.body.Isbn,
        author: req.body.author,
        pages: req. body.pages
      });
      const savedBook = await book.save();
      res.status(201).json({
        message: 'Successfully added book',
        book: savedBook
      });
    } catch (err) {
      res.status(400).json({ message: 'Error adding book', error: err.message });
    }
  });

  app.delete('/book/:isbn', async (req, res) => {
    try {
      const result = await Book.findOneAndDelete({ isbn: req.params.isbn });
      if (!result) {
        return res.status(404).json({ message: 'Book not found' });
      }
      res.json({
        message: 'Successfully deleted the book',
        book: result
      });
    } catch (err) {
      res.status(500).json({ message: 'Error deleting book', error: err.message });
    }
  });

  app.get('*', (req, res) => {
    res.sendFile(path.join(__dirname, '../public', 'index.html'));
  });
};
```

Create the model folder and file:

```bash
mkdir models && cd models
vi book.js
```

book.js content:

```js
const mongoose = require('mongoose');

const bookSchema = new mongoose.Schema({
  name: { type: String, required: true },
  isbn: { type: String, required: true, unique: true, index: true },
  author: { type: String, required: true },
  pages: { type: Number, required: true, min: 1 }
}, {
  timestamps: true
});

module.exports = mongoose.model('Book', bookSchema);
```

# Return to project root:

```bash
cd ../..
```

# Frontend (AngularJS)

Create the public folder:

```bash
mkdir public && cd public
```

Create script.js:
```bash
vi script.js
```

Paste:

```js
angular.module('myApp', [])
  .controller('myCtrl', function($scope, $http) {
    function fetchBooks() {
      $http.get('/book')
        .then(response => {
          $scope.books = response.data;
        })
        .catch(error => {
          console.error('Error fetching books:', error);
        });
    }

    fetchBooks();

    $scope.del_book = function(book) {
      $http.delete(`/book/${book.isbn}`)
        .then(() => {
          fetchBooks();
        })
        .catch(error => {
          console.error('Error deleting book:', error);
        });
    };

    $scope.add_book = function() {
      const newBook = {
        name: $scope.Name,
        isbn: $scope.Isbn,
        author: $scope.Author,
        pages: $scope.Pages
      };

      $http.post('/book', newBook)
        .then(() => {
          fetchBooks();
          // Clear form fields
          $scope.Name = $scope.Isbn = $scope.Author = $scope.Pages = '';
        })
        .catch(error => {
          console.error('Error adding book:', error);
        });
    };
  });
```

Create index.html:
```bash
vi index.html
```

Paste:
```html
<!DOCTYPE html>
<html ng-app="myApp" ng-controller="myCtrl">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Book Management</title>
  <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.8.2/angular.min.js"></script>
  <script src="script.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    table { border-collapse: collapse; width: 100%; }
    th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
    th { background-color: #f2f2f2; }
    input[type="text"], input[type="number"] { width: 100%; padding: 5px; }
    button { margin-top: 10px; padding: 5px 10px; }
  </style>
</head>
<body>
  <h1>Book Management</h1>
  
  <h2>Add New Book</h2>
  <form ng-submit="add_book()">
    <table>
      <tr>
        <td>Name:</td>
        <td><input type="text" ng-model="Name" required></td>
      </tr>
      <tr>
        <td>ISBN:</td>
        <td><input type="text" ng-model="Isbn" required></td>
      </tr>
      <tr>
        <td>Author:</td>
        <td><input type="text" ng-model="Author" required></td>
      </tr>
      <tr>
        <td>Pages:</td>
        <td><input type="number" ng-model="Pages" required></td>
      </tr>
    </table>
    <button type="submit">Add Book</button>
  </form>

  <h2>Book List</h2>
  <table>
    <thead>
      <tr>
        <th>Name</th>
        <th>ISBN</th>
        <th>Author</th>
        <th>Pages</th>
        <th>Action</th>
      </tr>
    </thead>
    <tbody>
      <tr ng-repeat="book in books">
        <td>{{book.name}}</td>
        <td>{{book.isbn}}</td>
        <td>{{book.author}}</td>
        <td>{{book.pages}}</td>
        <td><button ng-click="del_book(book)">Delete</button></td>
      </tr>
    </tbody>
  </table>
</body>
</html>
```

Return to project root:

```bash
cd ..
```

# Run & Test

Start the server:
```bash
node server.js
```
<img width="1920" height="1080" alt="Screenshot (536)" src="https://github.com/user-attachments/assets/b17e5d7b-99e8-4549-933d-222e2338e739" />

You should see:

```bash
Server up: http://localhost:3300
MongoDB connected
```

Local test from EC2 shell:

```bash
curl -s http://localhost:3300
```

Open TCP 3300 in your EC2 Security Group, then visit from your browser:

http://<YOUR-EC2-PUBLIC-IP>:3300
<img width="1920" height="1080" alt="Screenshot (537)" src="https://github.com/user-attachments/assets/a3e22c5a-59e2-42da-87d9-5e9aa46f45c1" />

# Security Group (SG) Rules

Minimum inbound rules to test this app:

- SSH (22/TCP): Your IP → Instance (to connect)

- App (3300/TCP): Your IP or 0.0.0.0/0 (for public access while testing)

# Troubleshooting & What We Fixed

- Service name confusion

Error: **sudo service mongodb start** → Unit mongodb.service not found.

Fix: Use the correct service name mongod:

```bash
sudo systemctl start mongod
sudo systemctl status mongod
```
<img width="1920" height="1080" alt="Screenshot (529)" src="https://github.com/user-attachments/assets/b460c692-5f5e-4837-9d62-e8ab45db4151" />

- Installing npm from Ubuntu repos caused dependency hell (on some systems)

Symptom: **apt install npm** → unmet dependencies

Fix: Use NodeSource to install Node.js which includes npm. If you already installed from Ubuntu and it worked, fine — but prefer NodeSource for reliability.

- Express path-to-regexp error

Symptom at runtime:

```bash
Missing parameter name at 1: https://git.new/pathToRegexpError
```
Cause: Express v5 beta pulled a stricter path-to-regexp parser

- Fix: Pin Express v4:
```bash
npm uninstall express
npm install express@4
```

npm EACCES permission errors

Symptom:
```bash
npm ERR! EACCES: permission denied, rename 'node_modules/...'
```
Cause: Running sudo npm install created root-owned node_modules

Fix:
```bash
sudo chown -R $USER:$USER ~/Books
rm -rf node_modules package-lock.json
npm install
```
<img width="1920" height="1080" alt="Screenshot (535)" src="https://github.com/user-attachments/assets/6cdc46a1-458a-4ce2-8f51-47bdfcfe7f55" />

Tip: Avoid sudo npm ... in project directories.

- Typing shell commands inside npm init

Symptom: Sorry, name can only contain URL-friendly characters.

Cause: Entered vi server.js inside the npm init prompt

Fix: CTRL + C to cancel, then run npm init -y, and edit files afterward.
