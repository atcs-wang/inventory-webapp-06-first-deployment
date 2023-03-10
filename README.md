# Part 06: First Deployment  

This tutorial follows after:
[Part 05: Implementing CRUD operations: Using Forms and POST requests](https://github.com/atcs-wang/inventory-webapp-05-handling-forms-post-crud)

With all the basic CRUD routes implemented, we have a usable web-app! At this point, it is time to  deploy a first version of your application to a cloud-hosting service.

In this tutorial, we'll do a variety of small changes that prepare our code for deployment. The eclectic set of changes detailed below are independent of each other, and could be added in any order. Furthermore, **all of these changes could have been incorporated earlier in the app development process.** Our tutorials so far have prioritized simplicity and getting things to "just work." We skipped these improvements in favor of only learning what was necessary to use/understand at the time, but in future apps you develop, you can utilize them almost immediately.

Finally, we'll use a beginner-friendly cloud hosting service to deploy your application to the web!

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

A popular `npm` package for web security is `helmet`, which provides middleware that set a variety of HTTP headers that protect against some common security flaws.

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
//Configure Express to use certain HTTP headers for security
//Explicitly set the CSP to allow certain sources
app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", 'cdnjs.cloudflare.com']
      }
    }
  })); 
```

One of the headers that helmet sets is `Content-Security-Policy`, which generally disallows scripts coming from external sources. This protects against a variety of injection attacks, but since we are utilizing JS from the Materialize framework (sourced from `cdnjs.cloudflare.com`), we are explicitly specifying its source is safe and allowed.

> Alternatively, instead of enabling specific external sources, we could have downloaded our fonts and the Materialize framework as local files, putting them in our `public` folder instead.

> NOTE: The `Content-Security-Policy` also disables inline JS scripts and CSS styles. If your webpages utilize inline scripts and styles, you should move them to external `.js` and `.css` files in your `public` folder instead.

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

# (6.5) Time to Deploy! (via `railway.app`)

Now you're ready to deploy your app for the first time on a cloud service!

The landscape of cloud services for web app deployment is changing constantly, and the options can be overwhelming, especially for first-time app developers. 

At the time of this tutorial's publishing, we would recommend using [railway.app](https://railway.app) for your first deployment. It sports:
1. A credit-card-less free tier
2. Beginner-friendly online dashboard
3. Easy integration with Github, with auto-deployment on new commits
4. Native support for MySQL database instances (and some other database types too)
5. Relatively succinct documentation

Railway's free-tier has limit of 500 hrs/month operation which would require some judicious "turning" off of your app to last the month. (Unchecked, you'll run out of hours after ~3 weeks of uptime).  If you're willing to add your credit card to your account, you can fairly affordably remove the limit, paying for whatever amount of compute power / runtime that you need. 

We will not provide detailed instructions on using Railway here, but here are the general steps you should take:
1. Create an account on [railway.app](https://railway.app); the most convenient is to use your Github account as your login.
2. As prompted, allow railway.app access to your repository.
3. Create a `Project` on your dashboard. Either use "Empty Project", or "Deploy from Github Repo".
4. In the new project, it should have auto-created an `Environment` called "production", which you can see at the top. (We can create multiple environments, but for now we only need one). If you did "Deploy from Github Repo" in Step 3, it might have already added a Service, in which case you can skip step 5 and 6.
5. In the "production" `Environment`, you can `Add a Service`, and choose `Github Repo`. 
6.  Select your repo; if not visible, choose `Configure Github App`, and make your repo visible, then select it.
7. You should see the `Service` on the dashboard corresponding to your Github repo. Click on it to see details.
8. In your `Service`'s details, there is a tab for `Deployments`. Every time you push a change to your repo, there should be a record of a new deployment. You can `Restart`, `Redeploy`, or `Remove` your deployment from this tab.
9. In your `Service`'s details, there is a tab for `Variables`. This is where environment variables can be set, which you need to make your deployed app able to connect to your database. You can add the variables and values from your `.env` file here, either one at a time or all at once via the `RAW Editor`. Changing the variables should cause a reployment automatically.
10. In your `Service`'s details, there is a tab for `Settings`. In the Environment section, you can click the `Generate Domain`. It will produce a domain that you can navigate to in your browser to see your web app live! 

As noted above, you only have 500 hours of runtime per month on a Free Tier, so you will want to `Remove` your deployment via the `Deployments` tab when you're not using it. It is easy enough to `Redeploy` when you need to start it up again.

## (6.5.1) OPTIONAL: Adding a production database  

At this point, using the database credentials from your `.env`, the deployed app shares the database that you've been using during development. At this time, you may find that acceptable (you're still developing after all), but you will eventually want to deploy a separate production MySQL database. 

If you want railway.app to create and manage a MySQL database, you can do the following:
1. In your "production" `Environment`, click the `+ New` button and choose `Database`, then `Add MySQL`. It should spin up a new `Service` that is a `MySQL` database.
2. The details for the `MySQL` `Service` should have rudimentary options for manually creating tables, or running arbitrary queries. You can run the setup queries that your `db_init.js` uses to get the new database set up the same way as your development database.  
3. Both the `Connect` and `Variables` tabs include the relevant database connection values, including hostname, port, username, password, and schema. Copy the relevant values and change the `Variables` for your deployed web app to use these values instead of your old `.env`'s values. (You can also use the connection info to set up a connection via MySQL Workbench if you would like to manage it directly using that tool) 

Note that this database will also count towards your 500 hours - even if your web app is down, as long as the database exists, it contributes to your runtime. The `Settings` has a button to `Delete Service` means your database will be completely removed. 

# (6.6) What's next? 

There are still many improvements to make to our app. But a pressing one, now that we have it online, is this: anyone can access our web app and cause changes to our database!

Next up - adding Authentication and Authorization.