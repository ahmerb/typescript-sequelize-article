# So you want to use types with Sequelize?

Once a team first takes the leap into the world of types with Typescript, it seems there's no going back. Typescript has become common to see in front-end applications, so why aren't we using all those juicy compile-time type checks on the server? At Vivacity, we've adopted Typescript on *both* the front- and back-end for new projects. We're currently building a new data set management tool using Typescript, Express and Sequelize. This guide will step you through how set this up, so you can create, read, update and delete without ever getting a type wrong.

## Plan

0. Explain the website and schema we are going to make
1. Setup: First, set up a default project with simple stuff.
2. Api design: Set up project to use controllers.
3. talk about how we're going to have each model have its own file. Except, rather than making it a default export, we're going to export a ModelFactory function, and two interfaces, ModelAttributes and ModelInstance. We avoid default exports because it makes type inference worse in typescript.
4. Model loader: We then have an index.ts file, similar to with js, that loads them up. Except, we also need a DbInterface type. It does this and doesn't load all files in a directory, because otherwise you'd lose type inference
5. Model definitions: Explain ModelAttrs, ModelInstance, SequelizeAttrs
6. Associations and the mixins
7. Glorious final result: writing api routes with lots of beautiful type inference (screenshot of intellisense)

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

Let's create a simple ExpressJS server. We'll do this using the `express-generator-ts` package. We'll use yarn throughout, but feel free to use npm.

```shell
$ yarn global add express-generator-ts
```

Now, we can create a boilerplate express app by running

```shell
$ express-ts blog-app --view=pug --git
```

The `--git` flag adds a gitignore. Note that, this command will also add the `pug` view engine for rendering HTML on the backend. We'll just be building a JSON api, so we can ignore that for now. We can run the app with

```shell
$ cd blog-app && yarn install
$ DEBUG=blog-app:* yarn start
```

We now have a very simple Express app with two routes, which are defined in the `/routes` directory. They don't do much. 




[todo]: https://i.imgur.com/OvMZBs9.jpg
