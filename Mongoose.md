Mongoose
===

Mongoose sits on top of the `mongodb` node.js driver  (meaning it uses it and provides you the developer
with a different coding API) and 
introduces model semantics (enforces data expectations through 
model validation).

## Setup

### Initialize

You need to set the Promise library mongoose will
use or you will see a deprecation warning. 

Do this in initialization phase (usually initial "require" of a
`setup-mongoose.js`, `connect.js`, etc. type module:

```js
mongoose.Promise = Promise;
```

In tests, you can use a global `before`:

```js
before(() => {
    require('mongoose').Promise = Promise;
});
```

### Connect

Connect using the `mongoose.connect()` method. Use the environment variable `process.env.MONGODB_URI` with a default
for local mongo when developing:

```js
const dbUri = process.env.MONGODB_URI || 'mongodb://localhost:27017/my-dev-db';
mongoose.connect(dbUri);
```

The format for the mongo connection URI is `mongodb://username:password@localhost:27017/mythical-animals` where:

* protocol: `mongodb`
* credentials: `username:password@` (optional, can be omitted locally on unsecure db)
* server: `localhost` (for dev)
* port: `27107` (default)
* database name: `mythical-animals`

After connecting, the connection is available via `mongoose.connection` which supports
events including `connected` and `error`. You can explicitly close the connection using
`mongoose.connection.close()`

You can also have your module export a function that gets called with the db uri. In which
case each caller can control how the db uri gets sets.

## Schemas

### Definition

Define data using `mongoose.Schema`:

```js
const schema = new Schema({
    name: {
        type: String,
        required: true
    },
    type: {
        type: String,
        enum: ['dog', 'cat', 'bird', 'lizard']
    }
});
```

Schema does not have to be flat, you can have sub-objects:

```js
new Schema({
    name: String,
    address: {
        line1: String,
        line2: String,
        city: String,
        state: String,
        zip: String
    }
});
```

And lists of things, arrays are defined by using the Array brackets `[]` and 
defining the contained items:

```js
const storeSchema = new Schema({
    name: String,
    hours: [{
        day: String,
        start: Number,
        end: Number
    }]
});
```

### Validation

Some of the most common validations are `required`, `enum` (limit to set of values), 
`min` and `max` type settings. You can also define custom validation functions:

```js
legs: {
        type: Number,
        // ultimate custom validation, write your own function!
        validate: {
            validator(value) {
                return value > 0 && value < 7;
            },
            message: '{VALUE} is the ___wrong__ number of legs for a pet!'
        }
    }
```

### Options

Include general schema options like timestamps or dynamic fields as the second parameter:

```js
const schema = new Schema({
    // fields go here
}, {
    // options go here:
    timestamps: true
});
```

## Models

While the schema _defines_ the data, creating a model from that schema enables us to have an
exportable object that can be used to work with that type of data:

```js
const Store = mongoose.model('Store', storeSchema);
module.exports = Store;
```

The string name passed as the first parameter is critical as it is used:

1. To define the collection name used in mongodb (default will be to lowercase and
pluralize the provided model name).
1. To refer to this model from other schemas when defining model definitions (see
next section).

### Relationships

To have one model refer to another model, define a field of type `ObjectId` and 
the `ref` name of the model it refers to (often called a "foreign key"):

```js
const petSchema = new Schema({
    name: String,
    store: {
        type: Schema.Types.ObjectId,
        ref: 'Store'
    }
});
```

#### One-to-many

The above example is a one-to-many (also called parent-child) relationship between 
stores (the one parent) and the pets at the store (the many children).

#### Many-to-Many

You can define a many-to-many by having an array of "foreign keys":

```js
const movieSchema = new Schema({
    title: String,
    actors: [{
        type: Schema.Types.ObjectId,
        ref: 'Actor'
    }]
});
```

Each movie can have multiple actors, and each actor can be in multiple movies.

You can add attributes (aka fields, properties or paths) to a many-to-many by creating an object as the child of the array:

```js
const movieSchema = new Schema({
    title: String,
    cast: [{
        actor: {
            type: Schema.Types.ObjectId,
            ref: 'Actor'
        },
        role: String,
        minutesOnScreen: Number
    }]
});
```

#### Sub-Documents

Sub-documents allow you to create relationships between models, that only exist
in our javascript code, when the data is saved to mongo the subdocument becomes
part of the parent document:

```js
const movieSchema = new Schema({
    title: String,
    // either:
    reviews: [reviewSchema],
    // or
    reviews: [{
        type: Schema.Types.ObjectId,
        ref: 'Review'
    }]
});
```

### Working with Models

#### Model Instances

Use the Model as a Constructor Function (or class) to create a new model and save it:

```js
const model = new MyModel(initialData);
// optionally continue to work with the model:
model.foo = 'foo';
// save the model to the database, returns a promise:
model.save().then(model => res.send(model));
```

#### Static Methods

For working with the mongo database when you don't have a specific instance of a model,
but would like to retrieve model(s) or update them in the database, use the methods 
defined on the Model:

```js
Pets.find({ type: 'bird' })
    .then(birds => ...);
```

#### Shaping Data

The `find` static methods return query objects that can be further modified before
the query is executed (sent to the database to actually get the data).

##### `select()`

Use select to explicit define the fields you do (or do not) want returned:

```js
Stores.find()
    .select('name address.city')
    .then(stores => ...)
```

##### `lean()`

By default, mongoose returns full document instances (meaning they have the `save()`, 
`validate()` and other instance methods on them). For a `GET` operation, we can skip
this step and return the raw data by adding `.lean()` to the query:

```js
Stores.find()
    .select('name address.city')
    .lean()
    .then(stores => ...)
```

##### `populate()`

Use populate for fields that refer to other models (mongo documents) and to have mongoose
fetch the corresponding related data and put it in place of the id in the current document:

```js
Pets.find()
    .populate('store')
    .then(pets => ...)
```

There is an expanded form of populate that allows you to specify fields to select as well
as allowing nested populates (See the docs).

`populate()` only works if the key (_id) of the other document is in the document you are retrieving. 

If the key _of_ the document you are retrieving is in the other document collection, in order  to "populate" a parent with child data, you need to manually fetch the data:

```js
Promise.all([
    Store.findById(storeId).lean(),
    Pets.find({ store: storeId }).lean()
])
.then(([store, pets]) => {
    store.pets = pets;
    res.send(store);
});
```

#### Custom methods

You can move logic from you route to the model by defining custom methods.

##### Static methods

Add top-level static methods by defining on the schema `statics` property:

```js
// assign a function to the "statics" object of our petSchema
petSchema.statics.findByName = function(name) {
  return this.find({ name: new RegExp(name, 'i') });
};

// use in routes via:
Pet.findByName('fido')
    .then(pets => ...);
});
```


##### Instance methods

Add model instance methods by defining on the schema `methods` property:

```js
// assign a function to the "statics" object of our petSchema
userSchema.virtual('password').set(function(password) {
    this.hash = bcrypt.hashSync(password, 8);
});

// use in routes via:
User.findOne({ username: credentials.username })
    .then(user => {
        if(!user.comparePassword(credentials.password)) {
            throw { code: 401 };
        }
        ...
    })
});
```

##### Other helpers

See also virtual properties and custom queries
[in the docs](http://mongoosejs.com/docs/guide.html)


