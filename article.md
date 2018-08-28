# So you want to use types with Sequelize?

Once a team first takes the leap into the world of types with Typescript, it seems there's no going back. Typescript has become common to see in front-end applications, so why aren't we using all those juicy compile-time type checks on the server? At Vivacity, we've adopted Typescript on *both* the front- and back-end for one of our new projects. We're currently building a new data set management tool using Typescript, Express and Sequelize. This guide will step you through how to set this up so you can create, read, update and delete without ever getting a type wrong.

> The are *some* guides out there already, such as [this useful guide][zalecki] by Michal Zalecki. While that can get you started, it doesn't tell you how to deal with associations, which get pretty tricky. We also hope this guides acts as a more comprehensive tutorial that guides you from start to being able to use Sequlize with Typescript fairly fluently.

Usually, adding types is a matter of adding a `@types` package, but for Sequelize, it's a more difficult endeavour. We have to know how to structure our app so that we get the best type inference; we have to understand how to define different types which the Sequelize functions will take to generate their own typings; and do lots of boilerplate to get support for associations.

## Let's get started

We're about to embark on a rollercoaster adventure. At first, adding Typescript will seem like nothing but a cumbersome waste of time. Hours lost to just littering your Sequelize models with a slew of verbose type annotations. But, persevere! By the time we reach the end, your models will be so well typed that your api routes will practically be writing themselves. Here's a sneak peak.

![img6]

Now you're enticed, let's get started. We will be creating a simple JSON api for a blog post website. Apparently, there's a big gap in the market for a site where tech nerds, croissant enthusiasts and techno heads can come together to post blogs. So, our website will allow blogs in these three different categories. Here's the database schema.

| Users           | Posts                                          | Comments         | PostUpvotes       |
|-----------------|------------------------------------------------|------------------|-------------------|
| id: number      | id: number                                     | id: number       | PostId: number    |
| name: string    | AuthorId: number                               | PostId: number   | UpvoterId: number |
| createdAt: date | title: string                                  | AuthorId: number | createdAt: date   |
| updatedAt: date | text: string                                   | text: string     | updatedAt: date   |
|                 | category: enum("tech", "croissants", "techno") | createdAt: date  |                   |
|                 | createdAt: date                                | updatedAt: date  |                   |
|                 | updatedAt: date                                |                  |                   |

Let's break down the associations. A `User` has many `Post`'s which they have authored. This is the `AuthorId` column on `Posts`. This is a one-to-many relationship. A `User` also has a many-to-many relationship with `Posts` via `PostUpvotes`: a user can upvote many posts, and a post can be upvoted by many users. Finally, `Comments` have both a `User`, via `AuthorId`, and a `Post`. These are both one-to-many.

Here's the equivalent diagram.

![schema]

## Finally, some code

Let's create a simple ExpressJS server. This article will follow along with [this][repo] repo. We use tags so you can see what the code should look
like at each stage. Let's clone the initial stage, which is just a very simple Express server written in Typescript.

```shell
$ git clone git@github.com:ahmerb/ts-sqlize-code.git
$ cd ts-sqlize-code
$ git checkout 0.start
```

We can run the app with

```shell
$ yarn install
$ yarn start
```

We now have a very simple Express app. Take a look at `src/app.ts`. This is the only source file. It just defines a single request handler that returns the json `{ message: 'hello, world' }` and then tells Express to listen on port 3000. We've also set up a basic Typescript project with tslint too. These are set up in the files `tsconfig.json` and `tslint.json`. We've also installed the packages `@types/sequelize` and `@types/express`. These are the Typescript definitions for Sequelize and Express. We've also installed the `pg` and `pg-hstore` packages because we'll be using postgres, but feel free to use something else. 

## Models, models, models

Now, we want to setup Sequelize models that reflect the db schema we defined earlier. We will create a file for each model in the directory `src/models/`. Each file will export two types: `ModelAttributes` and `ModelInstance`. They'll also export a `ModelFactory` function which will actually set up the model and register it with Sequelize. At first, we're going to ignore associations. We'll deal with these later.

Let's start with the `Users` model. Firstly, we'll define an interface `UserAttributes`. This interface defines all the attributes you specify when creating a new instance of the model. In our database schema, we defined that a `User` has `id`, `name`, `createdAt` and `updatedAt` attributes. Let's see how that looks.

```typescript
// src/models/User.ts
export interface UserAttributes {
  id?: number;
  name: string;
  createdAt?: Date;
  updatedAt?: Date;
};
```

Simple. Note that `id`, `createdAt` and `updatedAt` are optional. This is because, when creating a new instance of a model, we don't want to be
forced to specify these fields. Instead, we usually want Sequelize to automatically set them.

The second type each model needs is a `ModelInstance`. This represents a Sequelize instance for an actual database row. For example, the function `User.find({ id: 1 })` returns an instance of the `User` model. It will have the `ModelAttributes` we defined (i.e., the table columns) as well as more Sequelize instance methods such as `.validate`, `.save`, `.update`, et cetera. For `Users`, the interface is defined as follows.

```typescript
// src/models/User.ts
export interface UserInstance extends Sequelize.Instance<UserAttributes>, UserAttributes {
  // At the moment, there's nothing more to add apart
  // from the methods and attributes that the types 
  // `Sequelize.Instance<UserAttributes>` and
  // `UserAttributes` give us. We'll add more here when
  //  we get on to adding associations.
};
```

Now, we have the two types required. Next, we want to create a function that will actually define a Sequelize model for the `Users` table. The model should correspond to the `UserAttributes` and `UserInstance` types that we've already defined. We call this function `UserFactory`. In typical Sequelize manner, this function will take your Sequelize instance and a `DataTypes` object, and return the created Sequelize model. To define models in Sequelize, we use the `sequelize.define<TInstance, TAttributes>()` function. The function takes an object which specifies the columns to create. 

```typescript
// src/models/User.ts
export const UserFactory = (sequelize: Sequelize.Sequelize, DataTypes: Sequelize.DataTypes): Sequelize.Model<UserInstance, UserAttributes> => {
  const attributes: SequelizeAttributes<UserAttributes> = {
    name: {
      type: DataTypes.STRING
    }
  };

  const User = sequelize.define<UserInstance, UserAttributes>('User', attributes);

  return User;
};
```

Let's take a look at the `SequelizeAttributes<T>` type we use. To ensure that we pass `sequelize.define` an object that has definitions for all the attributes we specified in `UserAttributes`, we can create our own type `SequelizeAttributes<T>`. This is defined as follows. We'll put it in a `src/typings/` directory. We got this type definition from the great [article][zalecki] we mentioned by Michal Zalecki.

```typescript
// src/typings/SequelizeAttributes/index.d.ts
import { DataTypeAbstract, DefineAttributeColumnOptions } from "sequelize";

type SequelizeAttribute = string | DataTypeAbstract | DefineAttributeColumnOptions;

export type SequelizeAttributes<T extends { [key: string]: any }> = {
  [P in keyof T]: SequelizeAttribute
};
```

That's it! We've now created our first model file using Sequelize and Typescript. We can create similar files for `Posts` and `Comments`. However, we won't do `PostUpvotes` yet as that's a join table and we are still yet to dive into the depths of writing associations. Here's the three new files.

```typescript
// src/models/User.ts
import * as Sequelize from 'sequelize';
import { CommentAttributes, CommentInstance } from 'models/Comment';
import { PostAttributes, PostInstance } from 'models/Post';
import { SequelizeAttributes } from 'typings/SequelizeAttributes';

export interface UserAttributes {
  id?: number;
  name: string;
  createdAt?: Date;
  updatedAt?: Date;
};

export interface UserInstance extends Sequelize.Instance<UserAttributes>, UserAttributes {
};

export const UserFactory = (sequelize: Sequelize.Sequelize, DataTypes: Sequelize.DataTypes): Sequelize.Model<UserInstance, UserAttributes> => {
  const attributes: SequelizeAttributes<UserAttributes> = {
    name: {
      type: DataTypes.STRING
    }
  };

  const User = sequelize.define<UserInstance, UserAttributes>('User', attributes);

  return User;
};
```

```typescript
// src/models/Post.ts
import * as Sequelize from 'sequelize';
import { CommentAttributes, CommentInstance } from 'models/Comment';
import { UserAttributes, UserInstance } from 'models/User';
import { SequelizeAttributes } from 'typings/SequelizeAttributes';

export interface PostAttributes {
  id?: number;
  name: string;
  title: string;
  text: string;
  category: 'tech' | 'croissants' | 'techno';
  createdAt?: Date;
  updatedAt?: Date;
};

export interface PostInstance extends Sequelize.Instance<PostAttributes>, PostAttributes {
};

export const PostFactory = (sequelize: Sequelize.Sequelize, DataTypes: Sequelize.DataTypes): Sequelize.Model<PostInstance, PostAttributes> => {
  const attributes: SequelizeAttributes<PostAttributes> = {
    name: {
      type: DataTypes.STRING
    },
    title: {
      type: DataTypes.STRING
    },
    text: {
      type: DataTypes.STRING(5000) // extra long length
    },
    category: {
      type: DataTypes.ENUM('tech', 'croissants', 'techno')
    }
  };

  const Post = sequelize.define<PostInstance, PostAttributes>('Post', attributes);

  return Post;
};
```

```typescript
// src/models/Comment.ts
import * as Sequelize from 'sequelize';
import { PostAttributes, PostInstance } from 'models/Post';
import { UserAttributes, UserInstance } from 'models/User';
import { SequelizeAttributes } from 'typings/SequelizeAttributes';

export interface CommentAttributes {
  id?: number;
  text: string;
  createdAt?: Date;
  updatedAt?: Date;
};

export interface CommentInstance extends Sequelize.Instance<CommentAttributes>, CommentAttributes {
};

export const CommentFactory = (sequelize: Sequelize.Sequelize, DataTypes: Sequelize.DataTypes): Sequelize.Model<CommentInstance, CommentAttributes> => {
  const attributes: SequelizeAttributes<CommentAttributes> = {
    text: {
      type: DataTypes.STRING(1000)
    }
  };

  const Comment = sequelize.define<CommentInstance, CommentAttributes>('Comment', attributes);

  return Comment;
};

```

## But how do I use these?

Great. We have defined a load of factories and interfaces, but how do we actually use these and connect this up to a database? We'll do this in a file `src/models/index.ts`. This file will export a function `createModels` that will take a Sequelize config and create the models we've defined. We'll import this function in `app.ts` and trigger it there. We'll assume you have available some sequelize config for connecting to your database. If not, the Sequelize [docs][sqlize_config_docs] will get you started. Here's the new stuff we'll add to `app.ts`. 

```typescript
// src/app.ts
import { createModels } from 'models';
const sequelizeConfig = require('config/sequelizeConfig.json');
const db = createModels(sequelizeConfig);
db.sequelize.sync(); // tells Sequelize to sync all defined models to db
```

Now, let's make the `createModels` function. Typically, people create an `index.ts` file that has the function that creates models as a default export. The function then just calls `sequelize.import` with every file in the directory, which would be the model files. The problem with that approach is that there's no way to maintain typechecking. To keep those juicy types, we'll create an interface called `DbInterface`. Then, `createModels` will return an object `db` of type `DbInterface` which will have the Sequelize models attached to them. The important bit is that it will be explicitly defined in `DbInterface` which models are defined and what attributes and methods each model has. The `db` object can then be passed to api route handlers, which can use the models to make Sequelize queries.

Let's define `DbInterface`.

```typescript
// src/typings/DbInterface/index.d.ts
import * as Sequelize from "sequelize";
import { CommentAttributes, CommentInstance } from "src/models/Comment";
import { PostAttributes, PostInstance } from "src/models/Post";
import { UserAttributes, UserInstance } from "src/models/User";

export interface DbInterface {
  sequelize: Sequelize.Sequelize;
  Sequelize: Sequelize.SequelizeStatic;
  Comment: Sequelize.Model<CommentInstance, CommentAttributes>;
  Post: Sequelize.Model<PostInstance, PostAttributes>;
  User: Sequelize.Model<UserInstance, UserAttributes>;
}
```

Next, `models/index.ts`, where `createModels` lives.

```typescript
// src/models/index.ts
import Sequelize from 'sequelize';
import { DbInterface } from'typings/DbInterface';
import { UserFactory } from './User';
import { PostFactory } from './Post';
import { CommentFactory } from './Comment';

export const createModels = (sequelizeConfig: any): DbInterface => {
  const { database, username, password, params } = sequelizeConfig;
  const sequelize = new Sequelize(database, username, password, params);

  const db: DbInterface = {
    sequelize,
    Sequelize,
    Comment: CommentFactory(sequelize, Sequelize),
    Post: PostFactory(sequelize, Sequelize),
    User: UserFactory(sequelize, Sequelize)
  };

  return db;
};

```

Woah, we can finally write our api routes! That didn't take too long. Let's do that.

```typescript
// src/app.ts
app.get('/users', (req: Request, res: Response) => {
  db.User.findAll()
    .then((users: UserInstance[]) => res.status(200).json({ users }))
    .catch(err => res.status(500).json({ err: ['oops', err] }));
});

app.get('/comments', (req: Request, res: Response) => {
  db.Comment.findAll()
    .then((comments: CommentInstance[]) => res.status(200).json({ comments }))
    .catch(err => res.status(500).json({ err: ['oops', err] }));
});

app.get('/posts', (req: Request, res: Response) => {
  db.Post.findAll()
    .then((posts: PostInstance[]) => res.status(200).json({ posts }))
    .catch(err => res.status(500).json({ err: ['oops', err] }));
});
```

Now, I encourage you to not copy & paste the above and write them yourself. Just stop and marvel at how every time you type, your text editor is able to tell you all the methods available, their types (in lots of detail!), and even documentation for each. Glorious.

As an example, when you creating a Post, just look at how autocomplete blesses you.

![img4]

We get documentation, code completion and it even supports enum types.

If you try and pass an object that isn't exactly right, Typescript will know and tell you this at compile time! At compile time!! With any good editor or IDE, you'll get these errors right in your editor. No more waiting till production before you realise that you've made a typo, or that some attribute is a string, not a number.

![img5]

If you'd like to see the repo at this point, run

```shell
$ git checkout 1.basic_models
```

## Relationships are hard

This is where it gets tricky. Let's add the associations to our Sequelize models. Typically, when using Sequelize, our model factories will also add a method `associate` to the model, which will take the `db` object and do things like calling

```typescript
Comment.belongsTo(db.User, { as: 'author', foreignKey: 'AuthorId' });
```

If you don't know how to do associations in sequelize with Javascript, it might be helpful to read their [docs][sqlize_associations_docs] on it first. In short, we define associations by using the `belongsTo`, `hasOne`, `hasMany` and `belongsToMany` functions. The `as` option specifies the alias for referring to the model. The `foreignKey` option allows you to manually specify the name of the foreign key in the table.

Recall, we said we'd set up the following associations:

- `Comments` belong to one `User`, via `AuthorId`. A `User` can have many `Comments`.
- `Comments` belong to one `Post`, via `PostId`. A `Post` can have many `Comments`.
- `Posts` belong to one `User`, via `AuthorId`. A `User` can have many `Posts`.
- `Comments` belong to many `Users` and `Users` belong to many `Comments`, via the join table `PostUpvotes`. This is a many-to-many relationship.

So, we can update our model factories to create these relationships. Take our `PostFactory` as an example.

```typescript
// src/models/Post.ts
export const PostFactory = (sequelize: Sequelize.Sequelize, DataTypes: Sequelize.DataTypes): Sequelize.Model<PostInstance, PostAttributes> => {
  const attributes: SequelizeAttributes<PostAttributes> = {
    name: {
      type: DataTypes.STRING
    },
    title: {
      type: DataTypes.STRING
    },
    text: {
      type: DataTypes.STRING(5000) // extra long length
    },
    category: {
      type: DataTypes.ENUM('tech', 'croissants', 'techno')
    }
  };

  const Post = sequelize.define<PostInstance, PostAttributes>('Post', attributes);

  // ** this is new **
  Post.associate = models => {
    Post.hasMany(models.Comment);
    Post.belongsTo(models.User, { as: 'author', foreignKey: 'AuthorId' });
  };

  return Post;
};
```

We can also add `.associate` functions to `UserFactory` and `CommentFactory`.

```typescript
User.associate = models => {
  User.hasMany(models.Comment, { foreignKey: 'AuthorId' });
  User.hasMany(models.Post, { foreignKey: 'AuthorId' });
  User.belongsToMany(models.Comment, {
    through: 'PostUpvotes',
    as: 'upvotedComments'
  });
};
```

```typescript
Comment.associate = models => {
  Comment.belongsTo(models.Post);
  Comment.belongsTo(models.User, { as: 'author', foreignKey: 'AuthorId' });
  Comment.belongsToMany(models.User, {
    through: 'PostUpvotes',
    as: 'upvoters'
  });
};
```

Now, what we have left to do is invoke these `.associate` functions from `src/models/index.ts`. That's easy.

```typescript
// src/models/index.ts
import Sequelize from 'sequelize';
import { DbInterface } from'typings/DbInterface';
import { UserFactory } from './User';
import { PostFactory } from './Post';
import { CommentFactory } from './Comment';

export const createModels = (sequelizeConfig: any): DbInterface => {
  const { database, username, password, params } = sequelizeConfig;
  const sequelize = new Sequelize(database, username, password, params);

  const db: DbInterface = {
    sequelize,
    Sequelize,
    Comment: CommentFactory(sequelize, Sequelize),
    Post: PostFactory(sequelize, Sequelize),
    User: UserFactory(sequelize, Sequelize)
  };

  Object.keys(db).forEach(modelName => {
    if (db[modelName].associate) {
      db[modelName].associate(db);
    }
  });

  return db;
};
```

Now we're all set up! When we run our app, Sequelize will also create all the relationships in database as well as add methods onto user, comment and post instances such as `.getAuthor`, `.setPost` or `.hasUpvoters`, etc.

However, All is not as good as it seems! Let's consider the line

```typescript
Comment.belongsTo(db.User, { as: 'author', foreignKey: 'AuthorId' });
```

This adds a column `AuthorId` to the `Comments` table in the database. Sequelize then adds the methods `getAuthor`, `setAuthor` and `createAuthor` to a `CommentInstance`. But, the interface `CommentInstance` we defined doesn't have any type signatures for those methods. So, if we have a comment instance and we write `comment.getAuthor()`, then _**this will not compile!**_ Typescript doesn't know that the method `.getAuthor` exists on a `CommentInstance`, so this will fail at compile time. We need to add type signatures for these methods to the `CommentInstance` interface. What type do we give them? Sequelize provides some types with extremely convoluted names.

```typescript
import { UserAttributes, UserInstance } from './User';

export interface CommentInstance extends Sequelize.Instance<CommentAttributes>, CommentAttributes {
  // add types for Comment.BelongsTo(User, { as: 'author' }) association
  getAuthor: Sequelize.BelongsToGetAssociationMixin<UserInstance>;
  setAuthor: Sequelize.BelongsToSetAssociationMixin<UserInstance, UserInstance['id']>;
  createAuthor: Sequelize.BelongsToCreateAssociationMixin<UserAttributes>;
};
```

Well, those are a mouthful. Let's break them down.

- `Sequelize.BelongsToGetAssociationMixin<RoleInstance>` is a generic type that takes some `ModelInstance` interface and produces a type for the corresponding `getRole` function (e.g. `getAuthor`).
- `Sequelize.BelongsToSetAssociationMixin<RoleInstance, RoleId>` is a generic type that takes some `ModelInstance` interface and the type of the primary key of that model. It produces a type for the corresponding `setRole` function.
- `Sequelize.BelongsToCreateAssociationMixin<RoleAttributes>` is a generic type that takes some `ModelAttributes` interface and produces a type for the corresponding `createRole` function.

Now this is where it starts to get crazy. You'll need to add a set of mixin functions for every single association you create and on both the models involved in the association. Each type of association you define, i.e. `BelongsTo`, `BelongsToMany`, `HasOne` or `HasMany`, instructs Sequelize to add different functions to the model instances. You'll have to manually add type declarations for every association on every model. Yep. Every single one.

I'm about to show you the new instance interfaces for our schema. Please don't be scared. Rumour has it I showed these to a colleague and they were so terrified of ever having to write them that they quit web development all together and ran away to Latvia to become a swimming instructor (honestly, true story). But, don't go! Luckily, we at Vivacity Labs like to automate things so we've written a script [here][associations_script] that will generate them for you. Moreover, once I show you some lovely pictures of just how amazing getting all this type inference is when you're writing api routes, it'll all be worth it. Anyway, here they are.

```typescript
export interface CommentInstance extends Sequelize.Instance<CommentAttributes>, CommentAttributes {
  getPost: Sequelize.BelongsToGetAssociationMixin<PostInstance>;
  setPost: Sequelize.BelongsToSetAssociationMixin<PostInstance, PostInstance['id']>;
  createPost: Sequelize.BelongsToCreateAssociationMixin<PostAttributes>;

  getAuthor: Sequelize.BelongsToGetAssociationMixin<UserInstance>;
  setAuthor: Sequelize.BelongsToSetAssociationMixin<UserInstance, UserInstance['id']>;
  createAuthor: Sequelize.BelongsToCreateAssociationMixin<UserAttributes>;

  getUpvoters: Sequelize.BelongsToManyGetAssociationsMixin<UserInstance>;
  setUpvoters: Sequelize.BelongsToManySetAssociationsMixin<UserInstance, UserInstance['id'], 'PostUpvotes'>;
  addUpvoters: Sequelize.BelongsToManyAddAssociationsMixin<UserInstance, UserInstance['id'], 'PostUpvotes'>;
  addUpvoter: Sequelize.BelongsToManyAddAssociationMixin<UserInstance, UserInstance['id'], 'PostUpvotes'>;
  createUpvoters: Sequelize.BelongsToManyCreateAssociationMixin<UserAttributes, UserInstance['id'], 'PostUpvotes'>;
  removeUpvoter: Sequelize.BelongsToManyRemoveAssociationMixin<UserInstance, UserInstance['id']>;
  removeUpvoters: Sequelize.BelongsToManyRemoveAssociationsMixin<UserInstance, UserInstance['id']>;
  hasUpvoter: Sequelize.BelongsToManyHasAssociationMixin<UserInstance, UserInstance['id']>;
  hasUpvoters: Sequelize.BelongsToManyHasAssociationsMixin<UserInstance, UserInstance['id']>;
  countUpvoters: Sequelize.BelongsToManyCountAssociationsMixin;
};
```

```typescript
export interface UserInstance extends Sequelize.Instance<UserAttributes>, UserAttributes {
  getComments: Sequelize.HasManyGetAssociationsMixin<CommentInstance>;
  setComments: Sequelize.HasManySetAssociationsMixin<CommentInstance, CommentInstance['id']>;
  addComments: Sequelize.HasManyAddAssociationsMixin<CommentInstance, CommentInstance['id']>;
  addComment: Sequelize.HasManyAddAssociationMixin<CommentInstance, CommentInstance['id']>;
  createComment: Sequelize.HasManyCreateAssociationMixin<CommentAttributes, CommentInstance>;
  removeComment: Sequelize.HasManyRemoveAssociationMixin<CommentInstance, CommentInstance['id']>;
  removeComments: Sequelize.HasManyRemoveAssociationsMixin<CommentInstance, CommentInstance['id']>;
  hasComment: Sequelize.HasManyHasAssociationMixin<CommentInstance, CommentInstance['id']>;
  hasComments: Sequelize.HasManyHasAssociationsMixin<CommentInstance, CommentInstance['id']>;
  countComments: Sequelize.HasManyCountAssociationsMixin;

  getPosts: Sequelize.HasManyGetAssociationsMixin<PostInstance>;
  setPosts: Sequelize.HasManySetAssociationsMixin<PostInstance, PostInstance['id']>;
  addPosts: Sequelize.HasManyAddAssociationsMixin<PostInstance, PostInstance['id']>;
  addPost: Sequelize.HasManyAddAssociationMixin<PostInstance, PostInstance['id']>;
  createPost: Sequelize.HasManyCreateAssociationMixin<PostAttributes, PostInstance>;
  removePost: Sequelize.HasManyRemoveAssociationMixin<PostInstance, PostInstance['id']>;
  removePosts: Sequelize.HasManyRemoveAssociationsMixin<PostInstance, PostInstance['id']>;
  hasPost: Sequelize.HasManyHasAssociationMixin<PostInstance, PostInstance['id']>;
  hasPosts: Sequelize.HasManyHasAssociationsMixin<PostInstance, PostInstance['id']>;
  countPosts: Sequelize.HasManyCountAssociationsMixin;

  getUpvotedComments: Sequelize.BelongsToManyGetAssociationsMixin<CommentInstance>;
  setUpvotedComments: Sequelize.BelongsToManySetAssociationsMixin<CommentInstance, CommentInstance['id'], 'PostUpvotes'>;
  addUpvotedComments: Sequelize.BelongsToManyAddAssociationsMixin<CommentInstance, CommentInstance['id'], 'PostUpvotes'>;
  addUpvotedComment: Sequelize.BelongsToManyAddAssociationMixin<CommentInstance, CommentInstance['id'], 'PostUpvotes'>;
  createUpvotedComment: Sequelize.BelongsToManyCreateAssociationMixin<CommentAttributes, CommentInstance['id'], 'PostUpvotes'>;
  removeUpvotedComment: Sequelize.BelongsToManyRemoveAssociationMixin<CommentInstance, CommentInstance['id']>;
  removeUpvotedComments: Sequelize.BelongsToManyRemoveAssociationsMixin<CommentInstance, CommentInstance['id']>;
  hasUpvotedComment: Sequelize.BelongsToManyHasAssociationMixin<CommentInstance, CommentInstance['id']>;
  hasUpvotedComments: Sequelize.BelongsToManyHasAssociationsMixin<CommentInstance, CommentInstance['id']>;
  countUpvotedComments: Sequelize.BelongsToManyCountAssociationsMixin;
};
```

```typescript
export interface PostInstance extends Sequelize.Instance<PostAttributes>, PostAttributes {
  getComments: Sequelize.HasManyGetAssociationsMixin<CommentInstance>;
  setComments: Sequelize.HasManySetAssociationsMixin<CommentInstance, CommentInstance['id']>;
  addComments: Sequelize.HasManyAddAssociationsMixin<CommentInstance, CommentInstance['id']>;
  addComment: Sequelize.HasManyAddAssociationMixin<CommentInstance, CommentInstance['id']>;
  createComment: Sequelize.HasManyCreateAssociationMixin<CommentAttributes, CommentInstance>;
  removeComment: Sequelize.HasManyRemoveAssociationMixin<CommentInstance, CommentInstance['id']>;
  removeComments: Sequelize.HasManyRemoveAssociationsMixin<CommentInstance, CommentInstance['id']>;
  hasComment: Sequelize.HasManyHasAssociationMixin<CommentInstance, CommentInstance['id']>;
  hasComments: Sequelize.HasManyHasAssociationsMixin<CommentInstance, CommentInstance['id']>;
  countComments: Sequelize.HasManyCountAssociationsMixin;

  getAuthor: Sequelize.BelongsToGetAssociationMixin<UserInstance>;
  setAuthor: Sequelize.BelongsToSetAssociationMixin<UserInstance, UserInstance['id']>;
  createAuthor: Sequelize.BelongsToCreateAssociationMixin<UserAttributes>;
};
```

I'll let you stare at those for a second.

## What about attributes?

We also want to update our `ModelAttributes` interfaces so that 1) we can access associations on a model instance when we use eager loading, and 2) we can specify associations when we create a model. This is what we have to add.

```typescript
export interface CommentAttributes {
  id?: number;
  text: string;
  createdAt?: Date;
  updatedAt?: Date;

  // ** this is new **

  // We allow both either PostAttributes or a Post's primary key,
  // so we can specifiy either when we create a model.
  // `posts?` is optional because we don't want to force
  // specifying associations when we create a model. We also
  // want to be able to query for Comment's without also having
  // to load its posts.
  post?: PostAttributes | PostAttributes['id'];

  // Similarly, we define the field `author?`. An `author` is an
  // alias for the `User` model, so we define that `author?` can
  // either be a `UserAttributes` or a `UserAttributes['id']`.
  author?: UserAttributes | UserAttributes['id'];
  
  // `upvoters` is a BelongsToMany association, so we define that
  // a comment can have an array of User's, under the field `upvoters`.
  upvoters?: UserAttributes[] | UserAttributes['id'][];
};
```

```typescript
export interface UserAttributes {
  id?: number;
  name: string;
  createdAt?: Date;
  updatedAt?: Date;

  // ** this is new **
  comments?: CommentAttributes[] | CommentAttributes['id'][];
  posts?: PostAttributes[] | PostAttributes['id'][];
  upvotedComments?: CommentAttributes[] | CommentAttributes['id'][];
};
```

```typescript
export interface PostAttributes {
  id?: number;
  name: string;
  title: string;
  text: string;
  category: 'tech' | 'croissants' | 'techno';
  createdAt?: Date;
  updatedAt?: Date;

  // ** this is new **
  comments?: CommentAttributes[] | CommentAttributes['id'][];
  author: UserAttributes | UserAttributes['id'];
};
```

If you want to see the repo at this point, run

```shell
$ git checkout 2.associations
```

We've also added lots of example api routes to `app.ts` so you can see how to work with associations.

## Finally, the good stuff!

Okay, we've just slaved away writing these big interfaces (or, I did, but you have a nice automated tool to do it for you). What was the point? Well, just look at these.

![img3]

The types Sequelize provide are well detailed, up to the most intricate settings in options objects.

![img2]

We even get rich documentation, right in our editor.

![img1]

Even inside this `map`, the types are inferred and our editor knows which attributes we have available on the `user` object. Invaluable.

![img7]

You may need to install extensions for your editor to get type inference and linting. VS Code has built-in Typescript support and IntelliSense. We also have the `tslint` extension installed.

So, that's that. A small investment into boilerplate and setting up Typescript brings us long-term codebase scalability, free documentation and type-safety. A few weeks of catching bugs instantly in the editor, better code readability and code completion, then you'll be sure its worth it.

By Ahmer Butt, with the help of Maksis, Dimitri, Martin, Amir and Craig.

Are you interested in working on similar problems to what we've described in this blog? If you want to work in a team on the cutting edge of transport innovation, then Vivacity Labs are for you. Send us your CV!

[todo]: https://i.imgur.com/OvMZBs9.jpg
[repo]: https://github.com/ahmerb/ts-sqlize-code/
[sqlize_associations_docs]: http://docs.sequelizejs.com/manual/tutorial/associations.html
[sqlize_config_docs]: http://docs.sequelizejs.com/manual/tutorial/
[associations_script]: https://github.com/ahmerb/sequelize-typescript-associations
[zalecki]: https://michalzalecki.com/using-sequelize-with-typescript/

[schema]: schema.png
[img1]: img1.png
[img2]: img2.png
[img3]: img3.png
[img4]: img4.png
[img5]: img5.png
[img6]: img6.png
[img7]: img7.png
