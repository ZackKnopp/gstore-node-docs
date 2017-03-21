# Entity Methods

## save()

After you create an entity you can persist its data to the Datastore with `entity.save()`  
This method accepts the following arguments

```js
entity.save(
    /* {Transaction} -- optional. Will execute the save operation inside this transaction */
    <transaction>,
    /* {object} -- optional. Additional config */
    <options>,
    /* {function} -- optional. The callback, if not passed a Promise is returned */
    <callback>
)
```

#### options

The options argument has a **method** property where you can set the saving method.
It default to 'upsert'.

```js
{
  method: 'upsert|insert|update', // default: 'upsert'
}
```

Example:
```js
// blog-post.model.js
const gstore = require('gstore-node');

const blogPostSchema = new gstore.Schema({
  title: { type:'string' },
  createdOn: { type:'datetime', default: gstore.defaultValues.NOW }
});

module.exports = gstore.model('BlogPost', blogPostSchema);
```

```js
// blog-post.controller.js
const BlogPost = require('./blog-post.model');

const data = { title: 'My first blog post' };
const blogPostEntity = new BlogPost(data);

// Example 1: Promise
blogPostEntity.save().then((response) => {
  const entity = response[0];
  console.log(entity.entityKey.id); // auto-generated id
}).catch(err => { ... });

// Example 2: callback
blogPostEntity.save(function onBlogPostSave(err, entity) {
    if (err) { // deal with err }

    console.log(entity.entityKey.id); // auto-generated id
});

/*
 * From inside a transaction
 */
const blogPost = new BlogPost({ title: 'My new blog post' });
const transaction = gstore.transaction();

transaction.run()
           .then(() => { return user.save(transaction) })
           .then(transaction.commit)
           .then((response) => {
              const apiResponse = data[0];
              // ... transaction finished
            }).catch((err) => {
              // handle error
            });

/*
 * Changing the method to save
 */

var blogPostEntity = new BlogPost(data);
blogPostEntity.save(null, { method: 'insert' }).then( ... );

```

Note on **saving inside a Transaction**  
By default, the entity data is validated before being saved in the Datastore (you can desactivate this behavious by setting [validateBeforeSave](#validateBeforeSave) to false in the Schema definition). The validation middleware is async, which means that to be able to save inside a transaction and at the same time validate before, you need to resolve the *save* method before being able to commit the transaction.  
A solution to avoid this is to **manually validate** before saving and then desactivate the "pre" middelwares by setting **preHooksEnabled** to false on the entity.  
**Important**: This solution will bypass any other middleware that you might have defined on "save" in your Schema.

```js
const user = new User({ name: 'john' });
const transaction = gstore.transaction();

transaction.run().then() => {
	User.get(123, null, null, transaction).then((response) => {
		const user = response[0];
		user.email = 'john@domain.com';
		const valid = user.validate();
		
		if (!valid) {
		    // exit the transaction;
		}
		
		// disable pre middleware(s)
		user.preHooksEnabled = false;
		
		// save inside transaction in sync
		user.save(transaction);

		// ... any other transaction operations
		
		transaction.commit().then(() => {
		    ...
		});
	});
});
```