# Part 06: Pre-deployment preparation 

This tutorial follows after:
[Part 05: Implementing CRUD operations: Using Forms and POST requests](https://github.com/atcs-wang/inventory-webapp-05-handling-forms-post-crud)

With all the basic CRUD routes implemented, we have a usable web-app! At this point, it is almost time to  deploy a first version of your application to a cloud-hosting service.

In this tutorial, we'll do a variety of small changes that prepare our code for deployment. The eclectic set of changes detailed below are independent of each other, and could be added in any order. Furthermore, **all of these changes could have been incorporated earlier in the app development process.** Our tutorials so far have prioritized simplicity and getting things to "just work." We skipped these improvements in favor of only learning what was necessary to use/understand at the time, but in future apps you develop, you can utilize them almost immediately.

# (6.1) `npm` scripts

The Node environment allows us to define **custom scripts** - short aliases for commonly used commands - in the `package.json` file, and run them via `npm run scriptname`.

Find the `"scripts"` section of `package.json`:
```json
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```

and replace it with this:

```json
  "scripts": {
    "start": "node app.js", 
    "initdb": "node db/db_init.js"
   },
```

Now, you can try running some of these scripts:
```
> npm run initdb            //re-initialize the database
> npm run start             //start the server
> npm start                 //start the server (alternative)
```

Note that `npm start` can also be used in place of `npm run start`. All the other scripts require the full "`npm run ____`" but `npm start` is a special command. 

**Most cloud deployment services use `npm start` to start your server, so it's vital important that we defined the `"start"` script.**

Anytime you have a common command or series of commands, creating an npm script can make it more convenient, and documents it. We will add more of these scripts in the future.

# (6.2) Using `helmet` for security

If our app is going to be on the web, it could potentially be the target of a wide variety of attacks. 

A popular `npm` package for dealing with some common security flaws is `helmet`, which sets some default HTTP headers.

> Read more about helmet at [https://www.npmjs.com/package/helmet](https://www.npmjs.com/package/helmet)

Install it with:

```
> npm install helmet
```

Notice the new content in the `package.json` file:
```json 
  "devDependencies": {
    "helmet": "^6.0.1" //version number may vary
  }
```

Update the top section of `app.js`, with the other `require` statements, add the line below:

```js
const express = require( "express" );
...
const helmet = require("helmet"); //add this
```

And right after `app` is initialized, add the line below:

```js
const app = express(); 
...
app.use(helmet()); //add this
```

# (6.3) `PORT` environment variable

We have been using an arbitrary `port` number for our application: 8080 in the sample code, but during development/testing could be any number from 1024 to 65353. 

**However, cloud hosting services need to be able to specify which port to use on their servers** (typically `80` but we shouldn't assume). Most cloud hosting services will set an environment variable named `PORT` that our app should use when deployed. 

Change the `port` variable definition at the top of `app.js` from this:

```js
const port = 8080;
```

to this:

```js
const port = process.env.PORT || 8080;
```

If a `PORT` environment variable is set, our `port` variable will use it; otherwise, the port will default to 8080.

> If you want, try setting the `PORT` environment variable by adding this in the `.env` file:
> 
>```PORT=80``` 
>
> This isn't necessary, however, as we will not be setting the `PORT` manually at deployment.

# (6.4) Connection pool (vs single conneciton) for database queries

Make a COPY of `db/db_connection.js`, and call it `db/db_pool.js`. Change the last two lines from:

```js
const connection = mysql.createConnection(dbConfig);

module.exports = connection;
```

to this:

```js
const connectionPool = mysql.createPool(dbConfig);

module.exports = connectionPool;
```

What's the difference? While `db/db_connection.js` exports a single database connection (a `mysql.Connection` object), `db/db_pool.js` exports a connection *pool* (`mysql.Pool` object), which represents a *set* of managed database connections - that is, potentially *multiple, concurrent* connections.

> Why do we want a pool instead of just a single connection? A single connection can only run one query at a time. It has been sufficient during development since we are the only user, but once deployed our server will need to be able to handle multiple concurrent clients, each who may make requests requiring queries simultaneously. If sharing a single connection, one client's long query could bottleneck other clients' queries.

To have our server use this new connection pool defined in `db/db_pool.js`, simply change the line at the top of `app.js` from:

```js
const db = require('./db/db_connection');
```
to:
```js
const db = require('./db/db_pool');
```

Now `db` throughout `app.js` is a `Pool` object, rather than a `Connection` object, but thankfully the `.execute()` method works similarly for both. When `db.execute()` is called now, it will either open a new connection *OR* use an existing connection that isn't currently busy to execute the query. The pool will also gracefully handle unexpected connection closings and timeouts, making it more robust in general than using a single connection.

> Notice that we did *NOT* update `db/db_init.js` from using a single connection to the pool. A single connection is a limitation for web servers, but actually ideal for `db/db_init.js`, beacuse its queries need to run in *sequence* - each query should complete before starting the next one. If each query was made with separate connections, we'd have no guarantee on the order of queries run.

> Read more about the use of connection pools here: [https://www.npmjs.com/package/mysql2#using-connection-pools](https://www.npmjs.com/package/mysql2#using-connection-pools) 

# (6.5) What's next?

Now you're ready to deploy your app for the first time on a cloud service!

We will not recommend a specific cloud service here, as the landscape of available services (especially those with a credit-card-less free tier) is changing constantly. If you're doing this tutorial as part of a class, your teacher will guide you through the basics of deploying with a suitable service.

> Here is an article from late 2022 with a few possibilities: 
> [https://codedamn.com/news/nodejs/free-hosting-website](https://codedamn.com/news/nodejs/free-hosting-website)
> Keep in mind, however, that MySQL database hosting
> is something you'll need to consider as well.

Afterwards, there are still many improvements to make to our app....