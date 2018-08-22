# So you want to use types with Sequelize?

Once a team first takes the leap into the world of types with Typescript, it seems there's no going back. Typescript has become common to see in front-end applications, so why aren't we using all those juicy compile-time type checks on the server? At Vivacity, we've adopted Typescript on *both* the front- and back-end for new projects. We're currently building a new data set management tool using Typescript, Express and Sequelize. This guide will step you through how set this up, so you can create, read, update and delete without ever getting a type wrong.

## Plan

1. Explain the website and schema we are going to make
2. Setup: First, set up a default project with simple stuff.
3. Api design: Set up project to use controllers.
4. talk about how we're going to have each model have its own file. Except, rather than making it a default export, we're going to export a ModelFactory function, and two interfaces, ModelAttributes and ModelInstance. We avoid default exports because it makes type inference worse in typescript.
5. Model loader: We then have an index.ts file, similar to with js, that loads them up. Except, we also need a DbInterface type. It does this and doesn't load all files in a directory, because otherwise you'd lose type inference
6. Model definitions: Explain ModelAttrs, ModelInstance, SequelizeAttrs
7. Associations and the mixins
8. Glorious final result: writing api routes with lots of beautiful type inference (screenshot of intellisense)

### Schema

- Users: firstName, lastName
- Posts: AuthorId, title, text, category: enum("tech", "croissants", "techno")
- Comments: PostId, AuthorId, text
- PostUpvotes: PostId, UpvoterId

Users (Author) 1:m Posts  
Users (Author) 1:m Comments  
Posts 1:m Comments  
User (Upvoter) n:m Comments  

## Let's get started

We're about to embark on a rollercoaster adventure. At first, adding Typescript will seem like nothing but a cumbersome waste of time. Hours lost to just littering your Sequelize models with a slew of verbose type annotations. But, peservere! By the time we reach the end, your models will be so well typed that your api routes will practically be writing themselves. Here's a sneak peak.

![image TODO][todo]

Now you're enticed, let's get started. We'll create a simple JSON api for a blog post website. Apparently, there's a big gap in the market for a site where tech nerds, croissant enthusiasts and techno heads can come together to post blogs. So, our website will allow blogs in these three different categories. Here's the database schema.

> TODO: Probably, in future, change this to a schema picture like the one Martin made for persp.

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

## Finally, some code

Let's create a simple ExpressJS server. This article will follow along with [this][repo] repo. We use tags so you can see what the code should look
like at each stage. Let's clone the initial stage, which is just a very simple Express server written in typescript.

```shell
$ git clone git@github.com:ahmerb/ts-sqlize-code.git
$ git checkout 0.start
```

We can run the app with

```shell
$ cd ts-sqlize-code && yarn install
$ yarn start
```

We now have a very simple Express app. Take a look at `app.ts`. It just defines a single request handler, that returns the json `{ message: 'hello, world' }` and then tells Express to listen on port 3000.

## Models, models, models

Now, we want to setup Sequelize models that reflect the db schema we defined earlier. We will create a file for each model in the directory
`src/models/`. Each file will export two types: `ModelAttributes` and `ModelInstance`. They'll also export a `ModelFactory` function, which will
take your Sequelize connection instance and actually set up the model. At first, we're going to ignore associations. We'll deal with these later.

Let's start with the `Users` model. Firstly, we'll define `UserAttributes`. This interface
defines all the attributes you specify when creating a new instance of the model. In our db schema, we defined that a `User` has
`id`, `name`, `createdAt` and `updatedAt` attributes. Let's see how that looks.

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
forced to specify these fields. Instead, we want Sequelize to handle automatically setting them.

The second type each model needs is a `ModelInstance`. This represents a Sequelize instance for an actual database row. For example, the function `User.findById(1)` returns an instance of the `User` model. It will have the `ModelAttributes` we defined (i.e., the table columns) as well as more Sequelize instance methods such as `.validate`, `.save`, `.update`, et cetera. For `Users`, the interface is defined as follows.

```typescript
// src/models/User.ts
export interface UserInstance extends Sequelize.Instance<UserAttributes>, UserAttributes {
  // At the moment, there's nothing more to add apart from the methods and attributes that the types Sequelize.Instance<UserAttributes> and UserAttributes give us. We'll add more here when we get on to adding associations.
};
```

Now, we have the two types required. Next, we want to create a function that will actually define a sequelize model for `Users`. The model should correspond to the `UserAttributes` and `UserInstance` types that we've already defined. We call this function `UserFactory`. In typical Sequelize manner, this function will take your Sequelize instance and a `DataTypes` object, and return the created Sequelize model. To define models in sequelize, we use the `sequelize.define<TInstance, TAttributes>()` function. The function takes an object which specifies the columns to create. 

```typescript
// src/models/User.ts
export const UserFactory = (sequelize: Sequelize.Sequelize, DataTypes: Sequelize.DataTypes): Sequelize.Model<UserInstance, UserAttributes> => {
  const attributes: SequelizeAttributes<UserAttributes> = {
    name: {
      type: DataTypes.STRING
    }
  };

  const User = sequelize.define<UserInstance, UserAttributes>("User", attributes);

  return User;
};
```

Let's take a look at the `SequelizeAttributes<T>` type we use. To ensure that we pass `sequelize.define` an object that has definitions for all the attributes we specified in `UserAttributes`, we can create our own type `SequelizeAttributes<T>`. This is defined as follows. We'll put it in a `src/typings/` directory.

```typescript
// src/typings/SequelizeAttributes/index.d.ts
import { DataTypeAbstract, DefineAttributeColumnOptions } from "sequelize";

type SequelizeAttribute = string | DataTypeAbstract | DefineAttributeColumnOptions;

export type SequelizeAttributes<T extends { [key: string]: any }> = {
  [P in keyof T]: SequelizeAttribute
};
```

That's it! We've now created our first model file using Sequelize and Typescript. We can create similar files for `Posts` and `Comments`. However, we won't do `PostUpvotes` yet, as that's a join table and we are still yet to dive into the depths of writing associations. Here's the three new files.

```typescript
// src/models/User.ts
import * as Sequelize from "sequelize";
import { SequelizeAttributes } from "src/typings/SequelizeAttributes";

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
    // id, createdAt and updatedAt are automatically added by sequelize
  };

  const User = sequelize.define<UserInstance, UserAttributes>("User", attributes);

  return User;
};
```

```typescript
// src/models/Post.ts
import * as Sequelize from "sequelize";
import { SequelizeAttributes } from "src/typings/SequelizeAttributes";

export interface PostAttributes {
  id?: number;
  name: string;
  createdAt?: Date;
  updatedAt?: Date;
};

export interface PostInstance extends Sequelize.Instance<PostAttributes>, PostAttributes {
};

export const PostFactory = (sequelize: Sequelize.Sequelize, DataTypes: Sequelize.DataTypes): Sequelize.Model<PostInstance, PostAttributes> => {
  const attributes: SequelizeAttributes<PostAttributes> = {
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

  const Post = sequelize.define<PostInstance, PostAttributes>("Post", attributes);

  return Post;
};
```

```typescript
// src/models/Comment.ts
import * as Sequelize from "sequelize";
import { SequelizeAttributes } from "src/typings/SequelizeAttributes";

export interface CommentAttributes {
  id?: number;
  name: string;
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

  const Comment = sequelize.define<CommentInstance, CommentAttributes>("Comment", attributes);

  return Comment;
};
```

## But how do I use these?

Great. We have defined a load of Factories and interfaces, but how do we actually use these and connect this up to a database? We'll do this in a file `src/models/index.ts`. We'll assume you have available some different sequelize config for connecting to your database. We'll use a sample one for a postgres database.

```typescript
import * as Sequelize from "sequelize";
import { DbInterface } from "src/typings/DbInterface";
import { UserFactory } from "./User";
import { PostFactory } from "./Post";
import { CommentFactory } from "./Comment";

export const createModels = (databaseId: string, sequelizeConfig): DbInterface => {
  const { url: dbConnectionString } = sequelizeConfig;
  const sequelize = new Sequelize(dbConnectionString, sequelizeConfig);

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

TODO: explain this, DbInterface, and how we're going to use `createModels` in app.ts

[todo]: https://i.imgur.com/OvMZBs9.jpg
[repo]: https://github.com/ahmerb/ts-sqlize-code/