# User authentication example project for Meteor JS

This project is an attempt at demonstrating a simple use-case of authentication and some basic authorization for an application written using the Meteor framework.

## Important notes

1. When logging in, your login and password are sent down to the server in plaintext through Meteor RPC (Meteor.call). I am open to suggestions here, but as far as I understand it, this is the job of HTTPS.

2. Some of the functionality with pub-sub depends on some bug fixes present only in > 0.3.5. See [this stackoverflow question](http://stackoverflow.com/questions/10608341/possible-meteor-bug-in-manual-publish-subscribe-scenario) for details if you're curious. As of the time of this writing, where 0.3.5 is the master branch, only the devel branch works.

3. There is currently a bug somewhere that I have yet to figure out with regards to generating a collection within Meteor.publish. When observing a collection, and a document is *removed*, that change doesn't appear to get propagated back up to the client. Additions appear to work OK... only removals appear to be affected.

## Installation

### Meteor

Since we'll soon be seeing a new version of Meteor > 0.3.5, I won't bother explaining how to install the devel branch (needed for a bug fix). Instead, follow the usual installation instructions over at <https://github.com/meteor/meteor>, and hopefully by the time you find this project, a newer version of Meteor will be out! Otherwise, get your hands dirty and setup meteor using the devel branch.

### Userauth (this project)

Userauth isn't setup as a Meteor package. Since it's a proof-of-concept, you'll need to clone the repository and play around/learn how it works, then pull it apart and adapt it for your own application. So go ahead and clone it.

### npm & bcrypt

You don't explicitly need npm for Userauth, but you *do* need bcrypt. I'm using the [public/node_modules](http://stackoverflow.com/questions/10476170/how-can-i-deploy-node-modules-in-a-meteor-app-on-meteor-com) trick, but if that doesn't work for you, you will need to use npm to install [bcrypt](https://github.com/ncb000gt/node.bcrypt.js/) under `/usr/local/meteor/lib`.

## Overview

If you want to see it in action right now, head on over to <http://userauth.meteor.com/> (*currently not working, not sure why. It may be because of the dependency on the bcrypt node module*). Usernames/passwords can be found under `server/bootstrap.js`. To get you started, try the login *mathieu* and password *password*.

I'll briefly explain what each folder and file is used for in the project.

## `client/`

Holds all files that are sent to the client (browser).

### CSS

#### `client/css/app.css`

Self-explanatory: some basic styling for the example application.

### HTML Templates

#### `client/html/main.html`

This is where you'll find the `<head>` and `<body>` tags, along with the root template, which I've called `main`. You'll notice that for each major template, there will be a corresponding .js file of the same name under the `js/` folder. This is my own unique way of organizing my client-side file structure in Meteor projects.

The `main` template essentially includes all other templates.

#### `client/html/auth.html`

This template shows a simple login form.

#### `client/html/users.html`

This template shows a list of users, and a form to add a new user. A "delete" button will appear next to your user name (once logged in).

#### `client/html/notes.html`

This template shows a list of arbitrary "notes", grouped by user. These notes are just example data. They have an `is_private` flag that helps demonstrate server-side filtering based on authorization.

There is also a simple form to add a new note.

#### `client/html/info.html`

This template shows an information box with a *dismiss* button. Messages (good or bad) that come back from RPCs are usually shoved in there. For example, when a login attempt fails, that message goes in the info box.

### JavaScript client-side code

#### `client/js/main.js`

This file is a stub for now.

#### `client/js/auth.js`

Handles sending login info.

#### `client/js/users.js`

Handles displaying/deleting users. Note the `__is_session_user` field, which is added dynamically to the "users" collection on the server end.

#### `client/js/notes.js`

Handles displaying/deleting notes. Note the `__is_owned_by_session_user` field, which is added dynamically to the "notes" collection on the server end.

#### `client/js/info.js`

Handles displaying/dismissing the info box.

#### `client/js/lib/models.js`

This is where we create our client versions of collections "users" and "notes" via `new Meteor.Collection` calls.

#### `client/js/lib/rpc.js`

These methods are RPC helpers meant to be called from within the various Meteor client functions like Template.xxx.yyy, Template.xxx.events, etc.

They literally match the RPC endpoints on the server one-to-one. I've found this pattern to be the most effective, as it keeps the necessary division between client and server simple and clear. However, I do look forward to the sugar the Meteor team is cooking up in their auth branch.

You'll notice a pattern emerging from one function to the next. It's as simple as doing a Meteor.call to the equivalent RPC endpoint on the server, and the first parameter is always getSessionToken(), which is fetching the session token from, you guessed it, the Session. Subsequent parameters are also passed depending on the function signature, simple as that.

The callback is boilerplate too. As documented by Meteor, first parameter is the error exception (if any), and the result. You'll most likely see an `if (!error)` statement with some additional action to perform if necessary, and the `else` showing the error in our info box. Easy.

#### `client/js/lib/session.js`

These are helper methods having to do with the session token and cookie. On startup, we initialize the session token by checking for its presence as a cookie (see `initializeSessionToken`). If it's there, we use it. Otherwise, consider us logged-out. I use the string "unknown" as to be explicit about the session token not being known.

`rememberSessionToken` is called on successful login, which is the only time you'll be getting a session token from the server. After that, the session token is lost. Only a hash of it exists in the DB.

`forgetSessionToken` is called on logout.

`getSessionToken` is a simple abstraction so that we're not calling `Session.get("token")` throughout our own code.

#### `client/js/lib/subscribe.js`

This is where some of the cooler Meteor magic takes place.

We subscribe to `publishedUsers` and `publishedNotes`, and pass down our session token to these. This will allow the results coming back from the server to be affected by the session token (i.e. the server will know who is asking for users/notes and will be able to filter results).

Additionally, we wrap all of this in an `autosubscribe` so as to make the whole thing reactive. Now, whenever the session token changes (log in or log out), the users/notes will get updated.

### `server/`

Holds all files that are sent to the server (NodeJS).

#### `server/startup.js`

Intended to be where the server app initializes. We create the `Auth` global, which has a few public methods. We are instructing it to use our `Users` collection to manipulate user credentials etc, and also which fields to look for on the user documents.

Next, we lock down the `users` and `notes` collections so as to prevent client from modifying them, say via console.

Finally, we call our bootstrap function, detailed next.

#### `server/bootstrap.js`

Simply put, if the database is empty, it populates it with some sample data.

#### `server/models.js`

The "users" and "notes" collections are created here.

Also defined here are the near-raw CRUD (minus the R) operations on said collections. By near-raw, I mean they don't do any fancy authorization checks. And again by near-raw, they also do some basic sanity checks, such as in the update functions (`if (properties.name) set.name = properties.name;`).

#### `server/rpc-endpoints.js`

As briefly described in the client equivalent (rpc.js), these are the RPC endpoints that our client calls via `Meteor.call`. The pattern is kept simple:

- In almost all cases, we are passed the `sessionToken` as the first parameter. Subsequent parameters are function-specific.
- We can call `Auth.getUserBySessionToken` to verify that the sessionToken provided matches up with a real user.
- Then we do our checks. These range from simple to complex, whatever you want to prevent/allow, you do it here. Throwing exceptions for errors is the recommended way to go here.

#### `server/publish.js`

This is where it gets a bit more complicated. The complicated part doesn't come from providing different sets of notes depending on privileges. The complicated part (at least in my opinion) is when you want to send back a virtual collection, i.e. a collection that you construct within your publish.

This is accomplished using the `observe` method on cursors, along with some fancy footwork with `.set`, `.flush`, `.onStop`, etc. This [post on StackOverflow](http://stackoverflow.com/questions/10565654/how-does-the-messages-count-example-in-meteor-docs-work) may help you better understand exactly what's going on.

But in simple terms, what's happening is that "notes" are returned differently depending on whether the user is logged-in or not. Additionally (and more complicated), both "notes" and "users" have virtual attributes added, which are computed on-the-fly. They are helper attributes that we use client-side (`__is_owned_by_session_user` and `__is_session_user`).

#### `server/auth/auth.js`

bcrypt, SHA256, HMAC, this is where it all happens.

I've implemented this based on my accumulated experiences, mostly from perusing StackOverflow over the past few years and in writing implementations. As such, this implementation **may be totally flawed**. This is not at all a dig against SO; it's a dig against my own comfort-level with regards to my security knowledge. *I implore any security experts to carefully review this code and make any suggestions for improvement.*

Here's how I've approached security:

##### Creating user:

1. Client sends login/password (and other user details) in plaintext to the server (ideally via https)
2. Server generates password-hash from password using bcrypt, stores said hash in DB in user's row:

		var salt = bcrypt.genSaltSync(10);
		var hash = bcrypt.hashSync(password, salt);

##### User logging in by login/password:

1. Client sends login/password in plaintext to the server (ideally via https);
2. Server looks up user in DB by login;
3. Server uses bcrypt to check password against password-hash in DB (`bcrypt.compareSync`)
4. If correct, server generates a signed token:

		var randomToken = CryptoJS.SHA256(Math.random().toString()).toString();
		var signature = CryptoJS.HmacMD5(randomToken, serverKey).toString();
		var signedToken = randomToken + ":" + signature;

5. Signed token is hashed and stored in DB in user's row;
6. Server sends signed token to client, which it can store in a cookie if it pleases. The signed token is not saved on the server. The client can then use this session token in subsequent requests to identify itself.

##### User accessing resources using session token:

1. Client sends session token in plaintext to the server;
2. Server verifies the integrity of the session token (signed by server);
3. If successful, retrieves user row by hashing the session token, and looking for this same hash in the DB (stored in step 5 above).
4. Server can do what it wants with user row, such as allowing/denying access etc.

#### `server/auth/crypto.hmac-md5.js`

[CryptoJS](http://code.google.com/p/crypto-js/) library for HMAC MD5.

#### `server/auth/crypto.sha256.js`

[CryptoJS](http://code.google.com/p/crypto-js/) library for SHA256.

#### `server/auth/nodejs_require.js`

Near identical adaptation of the `require` code by Jonathan Kingston, which I found in a [StackOverflow answer](http://stackoverflow.com/questions/10476170/how-can-i-deploy-node-modules-in-a-meteor-app-on-meteor-com).

### `public/`

Contains only `node_modules/bcrypt/`, which is the NodeJS bcrypt library used to hash passwords on the server. Yes, it's in `public/`... but hopefully a better method to package up node modules (like, under `server/`) will be devised soon.

## Acknowledgments

- Of course, props to the Meteor team for putting together such a great framework;
- Special thanks to [Jonathan Kingston](https://github.com/jonathanKingston) of Britto fame for providing me a launch platform for understanding manual pub-sub in Meteor;
- StackOverflow for help in figuring out security best practices (bcrypt, etc);
- [node.bcrypt.js](https://github.com/ncb000gt/node.bcrypt.js/);
- [CryptoJS](http://code.google.com/p/crypto-js/);
- [Darren Schnare](https://github.com/dschnare) for some help in trying to debug what turned out to be a bug in Meteor ;-)