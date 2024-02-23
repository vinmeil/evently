# Step by step project creation process

Below will be the step by step process on how to create this project, starting from the initial set up all the way until the complete product.

## Initial Setup

First, create the `Next.js` application:

```bash
npx create-next-app@latest ./
```

Set up the `Next.js` application based on your preferences. After that, initialize your Dockerfile and other docker related files:

```bash
docker init
```

Set up the `Dockerfile` based on your needs and preferences, then create a Github repository and initialize it to the current app directory:

```bash
git init
git add .
git commit -m "first commit"
git remote add origin <github repo link>
git branch -M main
git push -u origin main
```

Create `(root)` and `(auth)` paths in the `/app` directory to allow implementation of the `layout.tsx` in `Next.js`

```
app
├── (auth)
├── (root)
├── favicon.ico
├── globals.css
└── layout.tsx
```

Make sure to properly change the default `layout.tsx` in the `/app` directory to suit your needs. Don't forget to create `layout.tsx` files in all frontend related folders, such as `(auth)` and `(root)`:

```
app
├── (auth)
    └── layout.tsx
├── (root)
    └── layout.tsx
├── favicon.ico
├── globals.css
└── layout.tsx
```

With this, the initial setup for `Next.js` is done, feel free to do anything else such as create new folders (e.g. a `shared` folder containing `Header.tsx`, `Footer.tsx`, etc) or install other packages

OPTIONAL STEPS (Installing `shadcn` and `uploadthing`)

For easy UI management, install `shadcn`:

```bash
npx shadcn-ui@latest init
```

NOTE: For `shadcn` related things, visit the [shadcn docs](https://ui.shadcn.com/docs) for more info. 

For easy file uploads, install `uploadthing`:

```bash
npm install uploadthing @uploadthing/react
```


## Setup Clerk

Visit the [clerk](https://dashboard.clerk.com/) website and create a new application, set it up as needed. Select the `Next.js` quickstart option and copy the API keys `env` variables to your `.env.local` file. When done with all of that, go to the [docs](https://clerk.com/docs/quickstarts/nextjs) on setting up `clerk` with `Next.js`.

After setting up `clerk`, go to your `middleware.ts` file and set it up such that the public can access specific routes even if they are not logged in:

```ts
// Example authMiddleware:
export default authMiddleware({
  publicRoutes: [
    "/",
    "/events/:id",
    "/api/webhook/clerk", // don't forget to expose your api routes as well!
    "/api/webhook/stripe",
    "/api/uploadthing",
  ],
  ignoredRoutes: [ // all api routes should go into your ignoredRoutes
    "/api/webhook/clerk",
    "/api/webhook/stripe",
    "/api/uploadthing",
  ]
});
```

Look at the rest of the `clerk` documentation to see if anything is missing or if there are any other features you want to implement in your app (scroll to the bottom of the previously opened docs). After all that, go to the `clerk` dashboard for your project, go to
`User & Authentication > Email, Phone, Username` and set up your user and authentication based on your needs.

EXTRA STEPS:

If you require getting a `userId`, which is automatically given as `_id` by `mongoose`, in your project, there is 1 extra step that must be done in the `clerk` dashboard. Go to your project's `clerk` dashboard and go to `Sessions > Customize session token`. Click on edit and add the following code:

```ts
{
  "userId": "{{user.public_metadata.userId}}"
}
```

This ensures that you get back the actual `userId` string instead of getting a `JSON` in the form of:

```ts
{
  "userId": "<userId goes here>",
}
```

## Setup MongoDB Database Connection

Install `mongodb` and `mongoose` in your project:
```bash
npm install mongodb mongoose
```

Create a new folder in `/lib` exclusively for your `mongodb` database, and inside it create an `index.ts` file where we will handle connecting to the `mongodb` database via `mongoose`:

```
├── app
├── components
├── constants
└── lib
    ├── utils.ts
    └── database
        └── index.ts
```

Create your `mongodb` database on [MongoDB Atlas](https://www.mongodb.com/atlas) and connect it to your application by following the instructions. Then, use the following code for your `index.ts` which allows us to cache a database connection over serverless api routes:

```ts
import mongoose from "mongoose";

const MONGODB_URI = process.env.MONGODB_URI;

let cached = (global as any).mongoose || { conn: null, promise: null };

// use this function with the await keyword in other files relating to the database
export const connectToDatabase = async () => {
  // if the connection has been cached, return the connection
  if (cached.conn) return cached.conn;
  
  if (!MONGODB_URI) throw new Error("MONGODB_URI is missing.");
  
  // set the promise to itself or a new connection if there is no promise (a.k.a not connected to database)
  cached.promise = cached.promise || mongoose.connect(MONGODB_URI, {
    dbName: "change the name later",
    bufferCommands: false,
  })
  
  cached.conn = await cached.promise;

  return cached.conn;
}
```

After creating and connecting the `mongodb` database, create a new folder in your `/database` directory to store all your data models:

```
├── app
├── components
├── constants
└── lib
    ├── utils.ts
    └── database
        ├── index.ts
        └── models
```

Create the necessary `schema` or data models for your database with the template of:

```ts
import { Document, Schema, model, models } from "mongoose";

// interface is optional, it only helps when developing the frontend to know what properties something should have
export interface IDataModel extends Document {
  _id: string; // automatically generated, comes from mongoose
  parameter1: string;
  parameter2: string;
}

const DataModelSchema = new Schema({
  parameter1: {
    type: String,
    required: true,
    unique: true,
    default: ""
  },
  parameter2: {
    type: String,
  },
})

const DataModel = models.DataModel || model("DataModel", DataModelSchema)

export default DataModel;
```

Note: When creating a `schema` for users, do not set last name to required. Should the user sign up using gmail or something else that allows the user to not have a last name, it will cause errors.

## Setup Webhooks

Follow the steps in the [clerk documentation](https://clerk.com/docs/users/sync-data) to sync clerk data to our backend using webhooks. In this documentation, you should skip step 1 and 2 if your website has not been deployed yet. In the case that your website hasn't been deployed yet, it is recommended that you follow steps 3-6 and then deploy your website right after, followed by steps 1 and 2.

After that, create a new folder in the `/lib` directory for all the possible actions that may happen in your app, in this case we will make the `user.actions.ts` file:

```
├── app
├── components
├── constants
└── lib
    ├── utils.ts
    ├── database
    └── actions
        └── user.actions.ts
```

In the `user.actions.ts` file, create asynchronous functions related to actions that can happen to the `users` in your database, such as creating a new user, deleting a user, updating a user, etc. Ensure that you use the `connectToDatabase()` function created in your `index.ts` file inside of your action functions:

```ts
// EXAMPLE ACTION FUNCTION
import { CreateUserParams } from '@/types'

export async function createUser(user: CreateUserParams) {
  try {
    await connectToDatabase()

    const newUser = await User.create(user)
    return JSON.parse(JSON.stringify(newUser))
  } catch (error) {
    handleError(error)
  }
}
```

## Create Other Parts of Your Application

You can create whatever else you need in your app after all of the above steps are said and done. The most complex aspect of this project will be adding the functionality of creating and updating events, in which we will be using a form. Below will be instructions on how to create forms using `shadcn` as that is what is being used in this project.

Visit the [`shadcn` form documentation](https://ui.shadcn.com/docs/components/form). Any additional steps will be listed below.
