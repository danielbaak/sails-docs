# Policies
### Overview
Policies are like any other system for authentication control. You can allow or deny access in fine granularity with policies.

Your app's ACL (access control list) is located in **config/policies.js**.

#### Applying a Policy

##### To a Specific Action

To apply a policy to a specific action in particular, you should specify it on the right-hand side of that action:

```javascript
{
  ProfileController: {
      edit: 'isLoggedIn'
  }
}
```


##### To an Entire Controller

To set the default policy mapping for a controller, use the `*` notation:
> **Note:** Default policy mappings do not "cascade" or "trickle down."  Specified mappings for the controller's actions will override the default mapping.  In this example, `isLoggedIn` is overriding `false`.

```javascript
{
  ProfileController: {
    '*': false,
    edit: 'isLoggedIn'
  }
}
```

##### Globally
> **Note:** Global policy mappings do not "cascade" or "trickle down" either.  Specified mappings, whether they're default controller mappings or for specific actions, will **ALWAYS** override the global mapping.  In this example, `isLoggedIn` is overriding `false`.

```javascript
{

  // Anything you don't see here (the unmapped stuff) is publicly accessible
  '*': true,

  ProfileController: {
    '*': false,
    edit: 'isLoggedIn'
  }
}
```


##### Built-in policies

###### true

> This is the default policy mapped to all controllers and actions in a new project.  In production, it's good practice to set this to `false` to prevent access to any logic you might have inadvertently exposed.

Allow public access to the mapped controller/action.  This will allow any request to get through, no matter what.

```javascript
module.exports = {
  UserController: {

    // login should always be accessible
    login: true

  }
}
```


###### false

**NO** access to the mapped controller/action.  No requests get through.  Period.

```javascript
module.exports = {
  MathController: {

    // This fancy algorithm we're working on isn't done yet
    // so we set it to false to disable it
    someFancyAlgorithm: false

  }
}
```


##### Custom policies

You can apply one or more policies to a given controller or action.  Any file in your `/policies` folder (e.g. `authenticated.js`) is referable in your ACL (`config/policies.js`) by its filename minus the extension, (e.g.  `'authenticated'`).


```javascript
module.exports = {
  FileController: {
    upload: ['isAuthenticated', 'canWrite', 'hasEnoughSpace']
  }
}
```

##### Multiple Policies

To apply two or more policies to a given action, (order matters!) you can specify an array, each referring to a specific policy. 

```javascript
UserController: {
    lock: ['isLoggedIn', 'isAdmin']
}
```

In each of the policies, the next policy in the chain will only be run if `next()`, the third argument, is called.  When and if the last policy calls `next()`, the requested controller action is run.







So, you don&rsquo;t want your mom to access your secret stash of ... code?  Here's how you can make that happen. 

## What Are Policies?

Policies in Sails are versatile tools for authorization and access control-- they let you allow or deny access to your controllers down to a fine level of granularity.  For example, if you were building Dropbox, before letting a user upload a file to a folder, you might check that she `isAuthenticated`, then ensure that she `canWrite` (has write permissions on the folder.)  Finally, you'd want to check that the folder she's uploading into `hasEnoughSpace`.

Policies can be used for anything: HTTP BasicAuth, 3rd party single-sign-on, OAuth 2.0, or your own custom authorization/authentication scheme.


## Writing Your First Policy

Policies are files defined in the `api/policies` folder in your Sails app.  Each policy file should contain a single function.

When it comes down to it, policies are really just Connect/Express middleware functions which run **before** your controllers.  You can chain as many of them together as you like-- in fact they're designed to be used this way.  Ideally, each middleware function should really check just *one thing*.

For example, the `canWrite` policy mentioned above might look something like this:

```javascript
// policies/canWrite.js
module.exports = function canWrite (req, res, next) {
  var targetFolderId = req.param('id');
  var userId = req.session.user.id;
  
  Permission
  .findOneByFolderId( targetFolderId )
  .exec( function foundPermission (err, permission) {

    // Unexpected error occurred-- skip to the app's default error (500) handler
    if (err) return next(err);

    // No permission exists linking this user to this folder.  Maybe they got removed from it?  Maybe they never had permission in the first place?  Who cares?
    if ( ! permission ) return res.redirect('/notAllowed');
    
    // OK, so a permission was found.  Let's be sure it's a "write".
    if ( permission.type !== 'write' ) return res.redirect('/notAllowed');

    // If we made it all the way down here, looks like everything's ok, so we'll let the user through
    next();
  });
};
```


## How do I protect my controllers with policies?

Sails has a built in ACL (access control list) located in `config/policies.js`.  This file is used to map policies to your controllers.  

This file is  *declarative*, meaning it describes *what* the permissions for your app should look like, not *how* they should work.  Declarative programming has many benefits, but in particular, it is both conventional and adaptable.  This makes it easier for new developers to jump in and understand what's going on, plus it makes your app more flexible as your requirements inevitably change over time.

You can apply one or more policies to a given controller or action.  Any file in your `/policies` folder (e.g. `authenticated.js`) is referable in your ACL (`config/policies.js`) by its filename minus the extension, (e.g.  `'authenticated'`).  

Additionally, there are a few special, built-in policy mappings:
  + `true`: public access  (allows anyone to get to the mapped controller/action)
  +  `false`: **NO** access (allows **no-one** to access the mapped controller/action)

 `'*': true` is the default policy for all controllers and actions.  In production, it's good practice to set this to `false` to prevent access to any logic you might have inadvertently exposed.

### Here&rsquo;s an example of adding some policies to a controller:
```javascript
  RabbitController: {

    // Apply the `false` policy as the default for all of RabbitController's actions
    // (`false` prevents all access, which ensures that nothing bad happens to our rabbits)
    '*': false,
  
    // For the action `nurture`, apply the 'isRabbitMother' policy 
    // (this overrides `false` above)
    nurture : 'isRabbitMother',
  
    // Apply the `isNiceToAnimals` AND `hasRabbitFood` policies
    // before letting any users feed our rabbits
    feed : ['isNiceToAnimals', 'hasRabbitFood']
  }
```

Here&rsquo;s what the `isNiceToAnimals` policy from above might look like: (this file would be located at `policies/isNiceToAnimals.js`)

We&rsquo;ll make some educated guesses about whether our system will consider this user someone who is nice to animals.
```javascript
module.exports = function isNiceToAnimals (req, res, next) {
  
  // `req.session` contains a set of data specific to the user making this request.
  // It's kind of like our app's "memory" of the current user.
  
  // If our user has a history of animal cruelty, not only will we 
  // prevent her from going even one step further (`return`), 
  // we'll go ahead and redirect her to PETA (`res.redirect`).
  if ( req.session.user.hasHistoryOfAnimalCruelty ) {
    return res.redirect('http://PETA.org');
  }

  // If the user has been seen frowning at puppies, we have to assume that
  // they might end up being mean to them, so we'll 
  if ( req.session.user.frownsAtPuppies ) {
    return res.redirect('http://www.dailypuppy.com/');
  }

  // Finally, if the user has a clean record, we'll call the `next()` function
  // to let them through to the next policy or our controller
  next();
};
```

#### Besides protecting rabbits (while a noble cause, no doubt), here are a few other use cases for policies:
+ cookie-based authentication
+ role-based access control
+ limiting file uploads based on MB quotas
+ any other kind of authentication scheme you can imagine


## What about me?  I'm using Passport?!

Passport works great with Sails!  In general, since Sails uses Connect/Express at its core, all of the Connect/Express-oriented things work pretty well.  In fact, Sails has no problem interpreting most Express middleware to work with socket.io.

There are a few good examples of this floating around.  

[Using Passport.JS with Sails.JS](http://jethrokuan.github.io/2013/12/19/Using-Passport-With-Sails-JS.html)

Here's a good one (hasn't been tested in v0.9.x yet): https://gist.github.com/theangryangel/5060446
