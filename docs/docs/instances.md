## Building a not-persistant instance

In order to create instances of defined classes just do as follows&period; You might recognize the syntax if you coded Ruby in the past&period; Using the `build`-method will return an unsaved object&comma; which you explicitly have to save&period;
    
```js
var project = Project.build({
  title: 'my awesome project',
  description: 'woot woot. this will make me a rich man'
})
 
var task = Task.build({
  title: 'specify the project idea',
  description: 'bla',
  deadline: new Date()
})
```

Built instances will automatically get default values when they were defined&colon;

```js
// first define the model
var Task = sequelize.define('Project', {
  title: Sequelize.STRING,
  rating: { type: Sequelize.STRING, defaultValue: 3 }
})
 
// now instantiate an object
var task = Task.build({title: 'very important task'})
 
task.title  // ==> 'very important task'
task.rating // ==> 3
```

To get it stored in the database&comma; use the `save`-method and catch the events ... if needed&colon;

```js
project.save().success(function() {
  // my nice callback stuff
})
 
task.save().error(function(error) {
  // mhhh, wth!
})
 
// you can also build, save and access the object with chaining:
Task
  .build({ title: 'foo', description: 'bar', deadline: new Date() })
  .save()
  .success(function(anotherTask) {
    // you can now access the currently saved task with the variable anotherTask... nice!
  }).error(function(error) {
    // Ooops, do some error-handling
  })
```

## Creating persistant instances

Besides constructing objects&comma; that needs an explicit save call to get stored in the database&comma; there is also the possibility to do all those steps with one single command&period; It's called `create`.
    
```js
Task.create({ title: 'foo', description: 'bar', deadline: new Date() }).success(function(task) {
  // you can now access the newly created task via the variable task
})
```

It is also possible to define which attributes can be set via the create method&period; This can be especially very handy if you create database entries based on a form which can be filled by a user&period; Using that would for example allow you to restrict the `User` model to set only a username and an address but not an admin flag&colon;
    
```js
User.create({ username: 'barfooz', isAdmin: true }, [ 'username' ]).success(function(user) {
  // let's assume the default of isAdmin is false:
  console.log(user.values) // => { username: 'barfooz', isAdmin: false }
})
```

## Updating / Saving / Persisting an instance

Now lets change some values and save changes to the database&period;&period;&period; There are two ways to do that&colon;
    
```js
// way 1
task.title = 'a very different title now'
task.save().success(function() {})
 
// way 2
task.updateAttributes({
  title: 'a very different title now'
}).success(function() {})
```

It's also possible to define which attributes should be saved when calling`save`&comma; by passing an array of column names&period; This is useful when you set attributes based on a previously defined object&period; E&period;g&period; if you get the values of an object via a form of a web app&period; Furthermore this is used internally for `updateAttributes`&period; This is how it looks like&colon;
    
```js
task.title = 'foooo'
task.description = 'baaaaaar'
task.save(['title']).success(function() {
 // title will now be 'foooo' but description is the very same as before
})
 
// The equivalent call using updateAttributes looks like this:
task.updateAttributes({ title: 'foooo', description: 'baaaaaar'}, ['title']).success(function() {
 // title will now be 'foooo' but description is the very same as before
})
```

## Destroying / Deleting persistant instances

Once you created an object and got a reference to it&comma; you can delete it from the database&period; The relevant method is `destroy`&colon;

```js
Task.create({ title: 'a task' }).success(function(task) {
  // now you see me...
 
  task.destroy().success(function() {
    // now i'm gone :)
  })
})
```

If the `paranoid` options is true, the object will not be deleted, instead the `deletedAt` column will be set to the current timestamp. To force the deletion, you can pass `force: true` to the destroy call:
    
```js
task.destroy({ force: true })
```

## Working in bulk (creating, updating and destroying multiple rows at once)

In addition to updating a single instance, you can also create, update, and delete multiple instances at once. The functions you are looking for are called

* `Model.bulkCreate`
* `Model.update`
* `Model.destroy`

Since you are working with multiple models, the callbacks will not return DAO instances. BulkCreate will return an array of model instances/DAOs, they will however, unlike `create`, not have the resulting values of autoIncrement attributes.`update` and `destroy` will return the number of affected rows.

First lets look at bulkCreate
    
```js
User.bulkCreate([
  { username: 'barfooz', isAdmin: true },
  { username: 'foo', isAdmin: true },
  { username: 'bar', isAdmin: false }
]).success(function() { // Notice: There are no arguments here, as of right now you'll have to...
  User.findAll().success(function(users) {
    console.log(users) // ... in order to get the array of user objects
  })
})
```

To update several rows at once:
    
```js
Task.bulkCreate([
  {subject: 'programming', status: 'executing'},
  {subject: 'reading', status: 'executing'},
  {subject: 'programming', status: 'finished'}
]).success(function() {
  Task.update(
    {status: 'inactive'} /* set attributes' value */, 
    {subject: 'programming'} /* where criteria */
  ).success(function(affectedRows) {
    // affectedRows will be 2
    Task.findAll().success(function(tasks) {
      console.log(tasks) // the 'programming' tasks will both have a status of 'inactive'
    })
  })
})
```

And delete them:
    
```js
Task.bulkCreate([
  {subject: 'programming', status: 'executing'},
  {subject: 'reading', status: 'executing'},
  {subject: 'programming', status: 'finished'}
]).success(function() {
  Task.destroy(
    {subject: 'programming'} /* where criteria */,
    {truncate: true /* truncate the whole table, ignoring where criteria */} /* options */
  ).success(function(affectedRows) {
    // affectedRows will be 2
    Task.findAll().success(function(tasks) {
      console.log(tasks) // no programming, just reading :(
    })
  })
})
```

If you are accepting values directly from the user, it might be beneficial to limit the columns that you want to actually insert.`bulkCreate()`accepts an options object as the second parameter. The object can have a `fields` parameter, &lpar;an array&rpar; to let it know which fields you want to build explicitly
    
```js
User.bulkCreate([
  { username: 'foo' },
  { username: 'bar', admin: true}
], { fields: ['username'] }).success(function() {
  // nope bar, you can't be admin! 
})
```

`bulkCreate` was originally made to be a mainstream&sol;fast way of inserting records&comma; however&comma; sometimes you want the luxury of being able to insert multiple rows at once without sacrificing model validations even when you explicitly tell Sequelize which columns to sift through&period; You can do by adding a `validate: true` property to the optons object.
    
```js
var Tasks = sequelize.define('Task', {
  name: {
    type: Sequelize.STRING,
    validate: {
      notNull: { args: true, msg: 'name cannot be null' }
    }
  },
  code: {
    type: Sequelize.STRING,
    validate: {
      len: [3, 10]
    }
  }
})
 
Tasks.bulkCreate([
  {name: 'foo', code: '123'},
  {code: '1234'},
  {name: 'bar', code: '1'}
], { validate: true }).error(function(errors) {
  /* console.log(errors) would look like:
  [
    { record: 
    ... 
    errors: 
      { name: 'SequelizeValidationError',
        message: 'Validation error',
        errors: [Object] } },
    { record: 
      ...
      errors: 
        { name: 'SequelizeValidationError',
        message: 'Validation error',
        errors: [Object] } } 
  ]  
  */
})
```

## Values of an instance

If you log an instance you will notice&comma; that there is a lot of additional stuff&period; In order to hide such stuff and reduce it to the very interesting information&comma; you can use the`values`-attribute&period; Calling it will return only the values of an instance&period;
    
```js
Person.create({
  name: 'Rambow',
  firstname: 'John'
}).success(function(john) {
  console.log(john.values)
})
 
// result:
 
// { name: 'Rambow',
//   firstname: 'John',
//   id: 1,
//   createdAt: Tue, 01 May 2012 19:12:16 GMT,
//   updatedAt: Tue, 01 May 2012 19:12:16 GMT
// }
```

**Hint&colon;**You can also transform an instance into JSON by using `JSON.stringify(instance)`&period; This will basically return the very same as `values`&period;

## Reloading instances

If you need to get your instance in sync&comma; you can use the method`reload`&period; It will fetch the current data from the database and overwrite the attributes of the model on which the method has been called on&period;

```js
Person.find({ where: { name: 'john' } }).success(function(person) {
  person.name = 'jane'
  console.log(person.name) // 'jane'
 
  person.reload().success(function() {
    console.log(person.name) // 'john'
  })
})
```

## Incrementing certain values of an instance

In order to increment values of an instance without running into concurrency issues&comma; you may use `increment`&period;

First of all you can define a field and the value you want to add to it&period;
    
```js
User.find(1).success(function(user) {
  user.increment('my-integer-field', 2).success(/* ... */)
})
```

Second&comma; you can define multiple fields and the value you want to add to them&period;
    
```js
User.find(1).success(function(user) {
  user.increment([ 'my-integer-field', 'my-very-other-field' ], 2).success(/* ... */)
})
```

Third&comma; you can define an object containing fields and its increment values&period;
    
```js
User.find(1).success(function(user) {
  user.increment({
    'my-integer-field':    2,
    'my-very-other-field': 3
  }).success(/* ... */)
})
```

## Decrementing certain values of an instance

In order to decrement values of an instance without running into concurrency issues&comma; you may use `decrement`&period;

First of all you can define a field and the value you want to add to it&period;
    
```js
User.find(1).success(function(user) {
  user.decrement('my-integer-field', 2).success(/* ... */)
})
```

Second&comma; you can define multiple fields and the value you want to add to them&period;
    
```js
User.find(1).success(function(user) {
  user.decrement([ 'my-integer-field', 'my-very-other-field' ], 2).success(/* ... */)
})
```

Third&comma; you can define an object containing fields and its decrement values&period;

```js
User.find(1).success(function(user) {
  user.decrement({
    'my-integer-field':    2,
    'my-very-other-field': 3
  }).success(/* ... */)
})
```