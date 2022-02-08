# Build a Node.js, Express & PostgreSQL REST API

## Overview

In this tutorial we will look at how to create a REST API using Node.js, Express & PostgreSQL and host it with heroku. Completing this tutorial will allow us to create APIs that will serve the ChineseLearner application.

## Installing PostgreSQL

Let's start by installing PostgreSQL locally. 

```bash
$ brew install postgresql
```

After installation, you can now start PostgreSQL.

```bash
$ brew services start postgresql
```

Login to postgres

```bash
$ psql postgress
```

Create a user and password for postgres

```bash
CREATE ROLE api_user LOGIN PASSWORD 'password';
```

Give the api_user access to create databases

```bash
ALTER ROLE api_user CREATEDB;
```

Logout of the root user

```bash
\q
```

Login to the newly created `api_user`

```bash
$ psql -d postgres -U api_user
```

Create a `chinese_learner_api` database

```bash
CREATE DATABASE chinese_learner_api;
```

Connect to the newly created database

```bash
\c chinese_learner_api
```

Create a words table with ID, chinese, pinyin and english

```bash
CREATE TABLE words (
ID SERIAL PRIMARY KEY,
chinese VARCHAR(255) NOT NULL,
pinyin VARCHAR(255) NOT NULL,
english VARCHAR(255) NOT NULL
);

Insert an entry into the new table

```bash
INSERT INTO words (chinese, pinyin, english)
VALUES ('你好','nǐ hǎo', 'hello');
```

We've now created out database, with one table to hold words, and added our first value, 你好！

## Create Express API

The Express API will setup an Express Server and route to two endpoints, `GET` and `POST`.

Create the following files:

- `.env`: file containing environment variables (does not get version controlled)
- `package.json`: information about the project and dependencies
- `init.sql`: file to intialize PostgreSQL table
- `config.js`: will create a database connection
- `index.js`: the express server

```bash
$ touch .env package.json init.sql config.js index.js
```

Set your environment variables in `.env` file

```
DB_USER=api_user
DB_PASSWORD=password
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=chinese_learner_api
```

Create a file for initializing the table with an entry. We'll use tis for the Heroku database.

```
CREATE TABLE words (
ID SERIAL PRIMARY KEY,
chinese VARCHAR(255) NOT NULL,
pinyin VARCHAR(255) NOT NULL,
english VARCHAR(255) NOT NULL
);

INSERT INTO words (chinese, pinyin, english)
VALUES ('你好','nǐ hǎo', 'hello');
```

Set up PostgreSQL connection

Use the node-postgres package to create a Pool, which will be used to make queries to the database.

Create a connection string that follows the pattern of `postgres://USER:PASSWORD@HOST:PORT/DATABASE`. We'll use the environment variable from `.env` using `process.env.VARIABLE`. Initializing with `require('dotenv').config()` will allow you to use thos environment varibale.

We should also created an `isProduction` string - in an environment like Heroku, `NODE_ENV` will be set to production so you can have a different behaviour between environments. Heroku will supply us with a string called `DATABASE_URL` for the connectionString, so we won't have to build a new one.

In `config.js` add

```javascript
require('dotenv').config()

const { Pool } = require('pg')
const isProduction = process.env.NODE_ENV === 'production'

const connectionString = `postgresql://${process.env.DB_USER}:${process.env.DB_PASSWORD}@${process.env.DB_HOST}:${process.env.DB_PORT}/${process.env.DB_DATABASE}`

const devConfig = {
  connectionString: connectionString,
  ssl: false
}

const prodConfig = {
  connectionString: process.env.DATABASE_URL,
  ssl: {
    rejectUnauthorized: false
  }
}

const pool = new Pool(isProduction ? prodConfig : devConfig)

module.exports = { pool }
```

Set up the Express server. Add the following to `index.js`

```javascript
const express = require('express')
const bodyParser = require('body-parser')
const cors = require('cors')
const { pool } = require('./config')

const app = express()

app.use(bodyParser.json())
app.use(bodyParser.urlencoded({ extended: true }))
app.use(cors())

const getWords = (request, response) => {
  pool.query('SELECT * FROM words', (error, results) => {
    if (error) {
      throw error
    }
    response.status(200).json(results.rows)
  })
}

const addword = (request, response) => {
  const { chinese, pinyin, english } = request.body

  pool.query(
    'INSERT INTO words (chinese, pinyin, english) VALUES ($1, $2)',
    [chinese, pinyin, enlglish],
    (error) => {
      if (error) {
        throw error
      }
      response.status(201).json({ status: 'success', message: 'Word added.' })
    }
  )
}

app
  .route('/words')
  // GET endpoint
  .get(getWords)
  // POST endpoint
  .post(addWord)

// Start server
app.listen(process.env.PORT || 3002, () => {
  console.log(`Server listening`)
})
```

The `package.json` file will just list your dependencies/devDependencies and other information.

- express: web server framework
- pg: PostgreSQL client for Node
- dotenv: allows you to load environment varibales from `.env` file
- cors: enbale CORS

We'll also install nodemon for development, which automatically restarts the server every time you make a change. 

Don't forget to include the `engines` property for Node version.

```json
{
  "name": "books-api",
  "version": "1.0.0",
  "private": true,
  "description": "Books API",
  "main": "index.js",
  "engines": {
    "node": "11.x"
  },
  "scripts": {
    "start": "node index.js",
    "start:dev": "nodemon index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  }
}
```

Now install all the production dependencies

```bash
$ npm i cors dotenv express pg
```

Also install the dev dependencies
```bash
$ npm i -D nodemon
```

Everything is now setup, so you can run `npm start` to start the server once, or `npm rum start:dev` to restart the server after every change.

```bash
$ npm start
```

You can test the API by making a call via curl

```bash
$ curl http://localhost:3002/books
[{"id":1,"chinese":"你好","pinyin":"nǐ hǎo","english":"hello"}]
```

# Deploy to Heroku

Go to the Heroku website and setup your account.

Next install the Heroku CLI.

```bash
$ brew install heroku/brew/heroku
```

Login to the Heroku CLI. This will open a browser window which you can use to log in.

```bash
$ heroku login
```

Create an app in Heroku.

```bash 
$ heroku create
```

Not passing a name will generate a random project name for you.

Alternatively, if you've already created an app in Heroku you can link this project to it using

```bash
$ heroku git:remote -a project
```

where project is the project name in heroku.

## Set up Heroku Postgres

Go to Heroku Add-ons and select Heroku Postgres. Click on "Instal Heroku Postgres". Click "Apply to app".

It might take up to 5 minutes to propogate. Once that time passes, check o see if your add-on exists in Heroku CLI.

```bash
heroku addons
```

You'll see your new PostgreSQL instance as some autogenerated name like `postgresql-whatever-00000`.

Log into you PostgreSQL instance.

```bash
$ heroku pg:psql postgresql-whatever-00000 --app example-node-api
```

Exit the PostgreSQL instance.

```
\q
```

From the root of your project where you have init.sql, run the following command to create your table and entries on Heroku Postgres.

## Test and deploy

At this point, everything should be set up and ready to go for Heroku. You can test this by running the following command

```bash
$ heroku local web
```

With this you can go to `http://localhost:5000/words` and see what your app will look like on Heroku.

Providing everything works good, you should now add, commit and push to Heroku.

Firstly, created a `.gitignore` file.

```bash
$ touch .gitignor
```

Add the `.env` and `node_modules` to the `.gitignore` file

```
.env
node_modules
```

Add your changes to git, commit and push

```bash
git add .
git commit -m "feat: add words route"
git push heroku master
```
