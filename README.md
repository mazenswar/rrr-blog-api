# Part 1: Setting up the Rails API

In this part, I will be using Ruby on Rails as an API. Rails is dynamic, intelligent and opinionated. Rails is easy to set-up and has very descriptive errors which make debugging issues a much more pleasant experience.

Since our project will only focus on auth, we will be only creating one Model, which is the `User` model.

## Creating the rails project

run `rails new rr-auth-api --api -T -d postgresql`

This command will generate a new rails api project.

The `--api` flag allows you to create a rails project with a limited subset of rails features that are geared toward api functionality.

The `-T` flag is a shorthand that equals `--skip-test` or `--no-skip-test` which will skip creating test files when creating the project. This flag is optional. Since I won't be using the test files, I want to keep my project setup as small as possible.

The `-d postgresql` command tells rails that we want to set up our project with Postgres as our database. The `-d` flag is shorthand for `--database=DATABASE`

After running this command make sure you navigate to the project folder by typing in `cd <project-name>` and in this case `cd rr--auth-api`

## Setting up gems:

1. 'jwt'
   A ruby implementation of the RFC 7519 OAuth JSON Web Token (JWT) standard.
   https://github.com/jwt/ruby-jwt

2. 'bcrypt'
   Handles salting, hashing and authenticating user passwords.
   https://github.com/codahale/bcrypt-ruby

3. 'cors'
   "Rack::Cors provides support for Cross-Origin Resource Sharing (CORS) for Rack compatible web applications.
   The CORS spec allows web applications to make cross domain AJAX calls without using workarounds such as JSONP. See Cross-domain Ajax with Cross-Origin Resource Sharing"
   https://github.com/cyu/rack-cors

After adding those gems to `Gemfile.rb` in your root directory (rack-cors and bcrypt are already there but commented out), we will go into our terminal and run `bundle install` which will install the gem dependencies for our project.

## Cors set up:

In the root of the app, we need to navigate to `config/initializers/cors.rb` and uncomment some code to make sure cors is up and running. If you don't see the `cors.rb` file, it may mean that you forgot to add `--api` flag when initializing your project with `rails new`.

In `cors.rb` our code should look something like this:

https://gist.github.com/mazenswar/8ce3fc43cb6343cb873304d2654fe6ee

Please note that this set-up should only be used in development. In production, you need to make sure that you specify the `origins` that are allowed to make requests to your api and the actions that request should be able to hit.

## Setting up our User resource

Next, we are going to go back to our terminal to create our User resource. In this case, I will utilize the `rails g resource <resource-name>` generator which will give us everything we need to get up and running. As you will be able to see in your terminal and in your project folder, this command will generate a migration, a model, a controller and a route.

So, we will create a User resource as follows:
`rails g resource User username password_digest`

Please note that since we are working with bcrypt, we need the password column in our table to be called `password_digest`. Brcypt looks for that attribute in our table when we try to authenticate user password or create new users and hash their passwords.

You should now have a
`app/controllers/users_controller.rb`

`app/models/user.rb`

`db/migrate/<migration_number>_create_users.rb`

And in `config/routes.rb` you should see `resources :users`

This will give your API RESTFUL endpoints(routes) to handle requests.

## Setting up the postgresql database

1. Before we can start writing code, we need to go to our terminal and:

- Create our database: Since we are using Postgres, Rails does not create the database automatically so we first need to make sure that our postgres application is running.
- run `rails db:create`
- Next, we will migrate our tables using `rails db:migrate`

2. The first thing we will do is adding `has_secure_password` method (which comes from ActiveModel) in our user model. This method provides us with a number of methods to set up and authenticate user passwords with bcrypt.
3. While we are in the User model, we will also set up a validation to make sure all usernames are unique. To do this, we add this line to our model `validates :username, uniqueness: true`
4. Next, we will move on to the `UsersController` and set up our actions. Since we want to be able to create users, show users, update users and delete users, we will have four actions `show`, `create`, `update` and `destroy`.
5. Since I want to 'log in' the users automatically when they create an account, I will generate a token for the user once their account is created. To do this, we will use the `JWT.encode` method. We will give the method three arguments:
   1. Payload: our payload will be an object that has the key `user_id` that points to the user's id.
   2. Secret: can be a string of your choosing. In this example I will use "thisIsMySecret".
      NOTE: I will define a method in the application controller named `secret` which I will be using to be consistent and modular. You should hide your secret in an environment file in production. Check out this link to see how you can do that in Rails 5.2 and up (https://www.viget.com/articles/storing-secret-credentials-in-rails-5-2-and-up/)
   3. Hashing algorithm: in this instance we will use 'HS256'.
6. The return value of the above method will be our token that we will send in our response and store in the browser to maintain the user's session.
7. Your controller should look like the following: https://gist.github.com/mazenswar/da61f2cca58600fca2523e905cb369e2
8. You will notice in the private method defined at the bottom that I used the key `:password` and not `:password_digest`. Bcrypt looks in our table for columns that have the `<word>_digest` where 'word' could be any key you choose, and in this case it's password. So Bcrypt knows that `:password` corresponds to our `:password_digest` column in our users' table.

## Testing our our table

To test our code, we will go to our console and create a user to test out our set-up. Navigate to your terminal and fire up the console using `rails c`.

We will then create our first user by running `User.create(username: "user1", password: "123")`
The command should return the instance of the User model that we created. You will notice that the `password_digest` key points to a long jumbled up string. This is the user's hashed password.

## Setting up JWT and testing with Postman

1. With our users' controller set up to deal with creating, showing, updating and deleting users, we will set up a different controller which I will call `auth_controller.rb` to handle logging in and persisting the user's session using JWT. My reasoning for separating this logic from the users' controller is that I want to separate my concerns. I want my users' controller to handle CRUD actions related to the users' table and my auth controller will strictly handle logging in and persisting the user.

2. So in `app/controllers` let's create a new file `auth_controller.rb`. In this controller we will define two actions `login` and `persist`.
3. `login`: this action will handle the following:

- This action will receive a user's username and password through params.
- It will try to find a user from the users' table.
- If a user is found, it will call the method `authenticate` on the user instance while passing in the user's password from params
- If there is a user and the user is authenticated, we will generate a token for our user and render a json object that has both our user instance and that user's token.
- Otherwise, we can render a json object with an error message.

4. `persist`: this action should do the following:

- Check if the headers in the request have a `Authorization` key
- If it does, then we will grab the token from the headers and decode it and use it to find the user instance that this token belongs to and render it as our response.
- If there is no `Authorization` key, then this action will do nothing.

5. This controller will not correspond to any tables in our database. And since we manually created the file, we need to define the routes. So, we will navigate to `config/routes.rb` and add two routes:

- A `POST` route to `/login` which will send requests to `auth#login`
- A `GET` route to `/auth` which will send requests to `auth#persist`

Now we can test our code and endpoints to make sure that our code is working as expected. I used Postman to make api calls all my endpoints and test them out. I will not go into details of how to test your api here. If you haven't used postman before, you can check out this helpful post (https://medium.com/aubergine-solutions/api-testing-using-postman-323670c89f6d)
