## Plan

1. Explain the website and schema we are going to make
2. Setup: First, set up a default project with simple stuff.
3. Api design: Set up project to use controllers.
4. talk about how we're going to have each model have its own file. Except, rather than making it a default export, we're going to export a ModelFactory function, and two interfaces, ModelAttributes and ModelInstance. We avoid default exports because it makes type inference worse in typescript.
5. Model loader: We then have an index.ts file, similar to with js, that loads them up. Except, we also need a DbInterface type. It does this and doesn't load all files in a directory, because otherwise you'd lose type inference
6. Model definitions: Explain ModelAttrs, ModelInstance, SequelizeAttrs
7. Associations and the mixins
8. Glorious final result: writing api routes with lots of beautiful type inference (screenshot of intellisense)

### Plan:  Schema

- Users: firstName, lastName
- Posts: AuthorId, title, text, category: enum("tech", "croissants", "techno")
- Comments: PostId, AuthorId, text
- PostUpvotes: PostId, UpvoterId

Users (Author) 1:m Posts  
Users (Author) 1:m Comments  
Posts 1:m Comments  
User (Upvoter) n:m Comments