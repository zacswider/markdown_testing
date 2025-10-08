# FastAPI Security Tutorial - Complete Guide

## Table of Contents
1. [Security Overview](#security-overview)
2. [Security First Steps](#security-first-steps)
3. [Get Current User](#get-current-user)
4. [Simple OAuth2 with Password and Bearer](#simple-oauth2-with-password-and-bearer)
5. [OAuth2 with Password (and hashing), Bearer with JWT tokens](#oauth2-with-password-and-hashing-bearer-with-jwt-tokens)

---

## Security Overview

There are many ways to handle security, authentication and authorization. In many frameworks and systems just handling security and authentication takes a big amount of effort and code (in many cases it can be 50% or more of all the code written).

**FastAPI** provides several tools to help you deal with **Security** easily, rapidly, in a standard way, without having to study and learn all the security specifications.

### OAuth2

OAuth2 is a specification that defines several ways to handle authentication and authorization. It is quite an extensive specification and covers several complex use cases. It includes ways to authenticate using a "third party".

That's what all the systems with "login with Facebook, Google, X (Twitter), GitHub" use underneath.

#### OAuth 1

There was an OAuth 1, which is very different from OAuth2, and more complex, as it included direct specifications on how to encrypt the communication. It is not very popular or used nowadays.

OAuth2 doesn't specify how to encrypt the communication, it expects you to have your application served with HTTPS.

### OpenID Connect

OpenID Connect is another specification, based on **OAuth2**. It just extends OAuth2 specifying some things that are relatively ambiguous in OAuth2, to try to make it more interoperable.

For example, Google login uses OpenID Connect (which underneath uses OAuth2). But Facebook login doesn't support OpenID Connect. It has its own flavor of OAuth2.

#### OpenID (not "OpenID Connect")

There was also an "OpenID" specification. That tried to solve the same thing as **OpenID Connect**, but was not based on OAuth2. So, it was a complete additional system. It is not very popular or used nowadays.

### OpenAPI

OpenAPI (previously known as Swagger) is the open specification for building APIs (now part of the Linux Foundation). **FastAPI** is based on **OpenAPI**.

OpenAPI has a way to define multiple security "schemes". By using them, you can take advantage of all these standard-based tools, including these interactive documentation systems.

OpenAPI defines the following security schemes:

- **`apiKey`**: an application specific key that can come from:
  - A query parameter.
  - A header.
  - A cookie.
- **`http`**: standard HTTP authentication systems, including:
  - `bearer`: a header `Authorization` with a value of `Bearer` plus a token. This is inherited from OAuth2.
  - HTTP Basic authentication.
  - HTTP Digest, etc.
- **`oauth2`**: all the OAuth2 ways to handle security (called "flows").
  - Several of these flows are appropriate for building an OAuth 2.0 authentication provider (like Google, Facebook, X (Twitter), GitHub, etc):
    - `implicit`
    - `clientCredentials`
    - `authorizationCode`
  - But there is one specific "flow" that can be perfectly used for handling authentication in the same application directly:
    - `password`: some next chapters will cover examples of this.
- **`openIdConnect`**: has a way to define how to discover OAuth2 authentication data automatically.
  - This automatic discovery is what is defined in the OpenID Connect specification.

### FastAPI utilities

FastAPI provides several tools for each of these security schemes in the `fastapi.security` module that simplify using these security mechanisms.

---

## Security First Steps

Let's imagine that you have your **backend** API in some domain. And you have a **frontend** in another domain or in a different path of the same domain (or in a mobile application). And you want to have a way for the frontend to authenticate with the backend, using a **username** and **password**.

We can use **OAuth2** to build that with **FastAPI**.

### How it looks

Let's first just use the code and see how it works, and then we'll come back to understand what's happening.

### Create `main.py`

```python
from typing import Annotated
from fastapi import Depends, FastAPI
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.get("/items/")
async def read_items(token: Annotated[str, Depends(oauth2_scheme)]):
    return {"token": token}
```

### Run it

The `python-multipart` package is automatically installed with **FastAPI** when you run the `pip install "fastapi[standard]"` command. However, if you use the `pip install fastapi` command, the `python-multipart` package is not included by default.

To install it manually:
```bash
pip install python-multipart
```

This is because **OAuth2** uses "form data" for sending the `username` and `password`.

Run the example with:
```bash
fastapi dev main.py
```

### Check it

Go to the interactive docs at: http://127.0.0.1:8000/docs

You will see:
- A shiny new "Authorize" button
- Your *path operation* has a little lock in the top-right corner that you can click
- A little authorization form to type a `username` and `password` (and other optional fields)

Note: It doesn't matter what you type in the form, it won't work yet.

### The `password` flow

The `password` "flow" is one of the ways ("flows") defined in OAuth2, to handle security and authentication.

OAuth2 was designed so that the backend or API could be independent of the server that authenticates the user. But in this case, the same **FastAPI** application will handle the API and the authentication.

Let's review it from that simplified point of view:

1. The user types the `username` and `password` in the frontend, and hits `Enter`
2. The frontend (running in the user's browser) sends that `username` and `password` to a specific URL in our API (declared with `tokenUrl="token"`)
3. The API checks that `username` and `password`, and responds with a "token"
   - A "token" is just a string with some content that we can use later to verify this user
   - Normally, a token is set to expire after some time
4. The frontend stores that token temporarily somewhere
5. The user clicks in the frontend to go to another section of the frontend web app
6. The frontend needs to fetch some more data from the API
   - But it needs authentication for that specific endpoint
   - So, to authenticate with our API, it sends a header `Authorization` with a value of `Bearer` plus the token
   - If the token contains `foobar`, the content of the `Authorization` header would be: `Bearer foobar`

### FastAPI's `OAuth2PasswordBearer`

**FastAPI** provides several tools, at different levels of abstraction, to implement these security features.

In this example we are going to use **OAuth2**, with the **Password** flow, using a **Bearer** token. We do that using the `OAuth2PasswordBearer` class.

A "bearer" token is not the only option, but it's the best one for our use case. And it might be the best for most use cases, unless you are an OAuth2 expert and know exactly why there's another option that better suits your needs.

When we create an instance of the `OAuth2PasswordBearer` class we pass in the `tokenUrl` parameter. This parameter contains the URL that the client (the frontend running in the user's browser) will use to send the `username` and `password` in order to get a token.

```python
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
```

Here `tokenUrl="token"` refers to a relative URL `token` that we haven't created yet. As it's a relative URL, it's equivalent to `./token`.

Using a relative URL is important to make sure your application keeps working even in an advanced use case like Behind a Proxy.

This parameter doesn't create that endpoint / *path operation*, but declares that the URL `/token` will be the one that the client should use to get the token. That information is used in OpenAPI, and then in the interactive API documentation systems.

The `oauth2_scheme` variable is an instance of `OAuth2PasswordBearer`, but it is also a "callable". It could be called as: `oauth2_scheme(some, parameters)`. So, it can be used with `Depends`.

### Use it

Now you can pass that `oauth2_scheme` in a dependency with `Depends`:

```python
@app.get("/items/")
async def read_items(token: Annotated[str, Depends(oauth2_scheme)]):
    return {"token": token}
```

**FastAPI** will know that it can use this dependency to define a "security scheme" in the OpenAPI schema (and the automatic API docs).

### What it does

It will go and look in the request for that `Authorization` header, check if the value is `Bearer` plus some token, and will return the token as a `str`.

If it doesn't see an `Authorization` header, or the value doesn't have a `Bearer` token, it will respond with a 401 status code error (`UNAUTHORIZED`) directly.

You don't even have to check if the token exists to return an error. You can be sure that if your function is executed, it will have a `str` in that token.

---

## Get Current User

In the previous chapter the security system (which is based on the dependency injection system) was giving the *path operation function* a `token` as a `str`.

But that is still not that useful. Let's make it give us the current user.

### Create a user model

First, let's create a Pydantic user model. The same way we use Pydantic to declare bodies, we can use it anywhere else:

```python
from pydantic import BaseModel

class User(BaseModel):
    username: str
    email: str | None = None
    full_name: str | None = None
    disabled: bool | None = None
```

### Create a `get_current_user` dependency

Let's create a dependency `get_current_user`. Remember that dependencies can have sub-dependencies?

`get_current_user` will have a dependency with the same `oauth2_scheme` we created before.

```python
async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]):
    user = fake_decode_token(token)
    return user
```

### Get the user

`get_current_user` will use a (fake) utility function we created, that takes a token as a `str` and returns our Pydantic `User` model:

```python
def fake_decode_token(token):
    return User(
        username=token + "fakedecoded", email="john@example.com", full_name="John Doe"
    )
```

### Inject the current user

So now we can use the same `Depends` with our `get_current_user` in the *path operation*:

```python
@app.get("/users/me")
async def read_users_me(current_user: Annotated[User, Depends(get_current_user)]):
    return current_user
```

Notice that we declare the type of `current_user` as the Pydantic model `User`. This will help us inside of the function with all the completion and type checks.

### Other models

You can now get the current user directly in the *path operation functions* and deal with the security mechanisms at the **Dependency Injection** level, using `Depends`.

And you can use any model or data for the security requirements (in this case, a Pydantic model `User`).

But you are not restricted to using some specific data model, class or type.

Do you want to have an `id` and `email` and not have any `username` in your model? Sure. You can use these same tools.

Do you want to just have a `str`? Or just a `dict`? Or a database class model instance directly? It all works the same way.

You actually don't have users that log in to your application but robots, bots, or other systems, that have just an access token? Again, it all works the same.

Just use any kind of model, any kind of class, any kind of database that you need for your application. **FastAPI** has you covered with the dependency injection system.

---

## Simple OAuth2 with Password and Bearer

Now let's build from the previous chapter and add the missing parts to have a complete security flow.

### Get the `username` and `password`

We are going to use **FastAPI** security utilities to get the `username` and `password`.

OAuth2 specifies that when using the "password flow" (that we are using) the client/user must send a `username` and `password` fields as form data.

And the spec says that the fields have to be named like that. So `user-name` or `email` wouldn't work.

But don't worry, you can show it as you wish to your final users in the frontend. And your database models can use any other names you want.

But for the login *path operation*, we need to use these names to be compatible with the spec (and be able to, for example, use the integrated API documentation system).

The spec also states that the `username` and `password` must be sent as form data (so, no JSON here).

#### `scope`

The spec also says that the client can send another form field "`scope`".

The form field name is `scope` (in singular), but it is actually a long string with "scopes" separated by spaces.

Each "scope" is just a string (without spaces).

They are normally used to declare specific security permissions, for example:
- `users:read` or `users:write` are common examples
- `instagram_basic` is used by Facebook / Instagram
- `https://www.googleapis.com/auth/drive` is used by Google

In OAuth2 a "scope" is just a string that declares a specific permission required. It doesn't matter if it has other characters like `:` or if it is a URL. Those details are implementation specific. For OAuth2 they are just strings.

### Code to get the `username` and `password`

Now let's use the utilities provided by **FastAPI** to handle this.

#### `OAuth2PasswordRequestForm`

First, import `OAuth2PasswordRequestForm`, and use it as a dependency with `Depends` in the *path operation* for `/token`:

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

@app.post("/token")
async def login(form_data: Annotated[OAuth2PasswordRequestForm, Depends()]):
    # ...
```

`OAuth2PasswordRequestForm` is a class dependency that declares a form body with:
- The `username`
- The `password`
- An optional `scope` field as a big string, composed of strings separated by spaces
- An optional `grant_type`
- An optional `client_id` (we don't need it for our example)
- An optional `client_secret` (we don't need it for our example)

The OAuth2 spec actually *requires* a field `grant_type` with a fixed value of `password`, but `OAuth2PasswordRequestForm` doesn't enforce it.

If you need to enforce it, use `OAuth2PasswordRequestFormStrict` instead of `OAuth2PasswordRequestForm`.

#### Use the form data

The instance of the dependency class `OAuth2PasswordRequestForm` won't have an attribute `scope` with the long string separated by spaces, instead, it will have a `scopes` attribute with the actual list of strings for each scope sent.

Now, get the user data from the (fake) database, using the `username` from the form field. If there is no such user, we return an error saying "Incorrect username or password".

#### Check the password

At this point we have the user data from our database, but we haven't checked the password.

Let's put that data in the Pydantic `UserInDB` model first.

You should never save plaintext passwords, so, we'll use the (fake) password hashing system.

If the passwords don't match, we return the same error.

##### Password hashing

"Hashing" means: converting some content (a password in this case) into a sequence of bytes (just a string) that looks like gibberish.

Whenever you pass exactly the same content (exactly the same password) you get exactly the same gibberish.

But you cannot convert from the gibberish back to the password.

###### Why use password hashing

If your database is stolen, the thief won't have your users' plaintext passwords, only the hashes.

So, the thief won't be able to try to use those same passwords in another system (as many users use the same password everywhere, this would be dangerous).

##### About `**user_dict`

`UserInDB(**user_dict)` means: *Pass the keys and values of the `user_dict` directly as key-value arguments*.

### Return the token

The response of the `token` endpoint must be a JSON object.

It should have a `token_type`. In our case, as we are using "Bearer" tokens, the token type should be "`bearer`".

And it should have an `access_token`, with a string containing our access token.

For this simple example, we are going to just be completely insecure and return the same `username` as the token.

By the spec, you should return a JSON with an `access_token` and a `token_type`, the same as in this example.

This is something that you have to do yourself in your code, and make sure you use those JSON keys.

It's almost the only thing that you have to remember to do correctly yourself, to be compliant with the specifications.

For the rest, **FastAPI** handles it for you.

### Update the dependencies

Now we are going to update our dependencies.

We want to get the `current_user` *only* if this user is active.

So, we create an additional dependency `get_current_active_user` that in turn uses `get_current_user` as a dependency.

Both of these dependencies will just return an HTTP error if the user doesn't exist, or if is inactive.

So, in our endpoint, we will only get a user if the user exists, was correctly authenticated, and is active.

The additional header `WWW-Authenticate` with value `Bearer` we are returning here is also part of the spec.

Any HTTP (error) status code 401 "UNAUTHORIZED" is supposed to also return a `WWW-Authenticate` header.

In the case of bearer tokens (our case), the value of that header should be `Bearer`.

You can actually skip that extra header and it would still work. But it's provided here to be compliant with the specifications.

Also, there might be tools that expect and use it (now or in the future) and that might be useful for you or your users, now or in the future.

That's the benefit of standards...

### See it in action

Open the interactive docs: http://127.0.0.1:8000/docs

#### Authenticate

Click the "Authorize" button. Use the credentials:
- User: `johndoe`
- Password: `secret`

#### Get your own user data

Now use the operation `GET` with the path `/users/me`.

You will get your user's data. If you click the lock icon and logout, and then try the same operation again, you will get an HTTP 401 error of: `{"detail": "Not authenticated"}`

#### Inactive user

Now try with an inactive user, authenticate with:
- User: `alice`
- Password: `secret2`

And try to use the operation `GET` with the path `/users/me`.

You will get an "Inactive user" error.

---

## OAuth2 with Password (and hashing), Bearer with JWT tokens

Now that we have all the security flow, let's make the application actually secure, using JWT tokens and secure password hashing.

This code is something you can actually use in your application, save the password hashes in your database, etc.

We are going to start from where we left in the previous chapter and increment it.

### About JWT

JWT means "JSON Web Tokens".

It's a standard to codify a JSON object in a long dense string without spaces. It looks like this:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

It is not encrypted, so, anyone could recover the information from the contents. But it's signed. So, when you receive a token that you emitted, you can verify that you actually emitted it.

That way, you can create a token with an expiration of, let's say, 1 week. And then when the user comes back the next day with the token, you know that user is still logged in to your system.

After a week, the token will be expired and the user will not be authorized and will have to sign in again to get a new token. And if the user (or a third party) tried to modify the token to change the expiration, you would be able to discover it, because the signatures would not match.

If you want to play with JWT tokens and see how they work, check https://jwt.io.

### Install `PyJWT`

We need to install `PyJWT` to generate and verify the JWT tokens in Python.

Make sure you create a virtual environment, activate it, and then install `pyjwt`:

```bash
pip install pyjwt
```

If you are planning to use digital signature algorithms like RSA or ECDSA, you should install the cryptography library dependency `pyjwt[crypto]`.

### Password hashing

"Hashing" means converting some content (a password in this case) into a sequence of bytes (just a string) that looks like gibberish.

Whenever you pass exactly the same content (exactly the same password) you get exactly the same gibberish. But you cannot convert from the gibberish back to the password.

#### Why use password hashing

If your database is stolen, the thief won't have your users' plaintext passwords, only the hashes.

So, the thief won't be able to try to use that password in another system (as many users use the same password everywhere, this would be dangerous).

### Install `pwdlib`

pwdlib is a great Python package to handle password hashes. It supports many secure hashing algorithms and utilities to work with them. The recommended algorithm is "Argon2".

Make sure you create a virtual environment, activate it, and then install pwdlib with Argon2:

```bash
pip install "pwdlib[argon2]"
```

With `pwdlib`, you could even configure it to be able to read passwords created by **Django**, a **Flask** security plug-in or many others.

So, you would be able to, for example, share the same data from a Django application in a database with a FastAPI application. Or gradually migrate a Django application using the same database.

And your users would be able to login from your Django app or from your **FastAPI** app, at the same time.

### Hash and verify the passwords

Import the tools we need from `pwdlib`. Create a PasswordHash instance with recommended settings - it will be used for hashing and verifying passwords.

pwdlib also supports the bcrypt hashing algorithm but does not include legacy algorithms - for working with outdated hashes, it is recommended to use the passlib library.

For example, you could use it to read and verify passwords generated by another system (like Django) but hash any new passwords with a different algorithm like Argon2 or Bcrypt. And be compatible with all of them at the same time.

Create a utility function to hash a password coming from the user. And another utility to verify if a received password matches the hash stored. And another one to authenticate and return a user.

### Handle JWT tokens

Import the modules installed. Create a random secret key that will be used to sign the JWT tokens.

To generate a secure random secret key use the command:
```bash
openssl rand -hex 32
```

And copy the output to the variable `SECRET_KEY` (don't use the one in the example).

Create a variable `ALGORITHM` with the algorithm used to sign the JWT token and set it to `"HS256"`.

Create a variable for the expiration of the token. Define a Pydantic Model that will be used in the token endpoint for the response. Create a utility function to generate a new access token.

### Update the dependencies

Update `get_current_user` to receive the same token as before, but this time, using JWT tokens.

Decode the received token, verify it, and return the current user. If the token is invalid, return an HTTP error right away.

### Update the `/token` *path operation*

Create a `timedelta` with the expiration time of the token. Create a real JWT access token and return it.

#### Technical details about the JWT "subject" `sub`

The JWT specification says that there's a key `sub`, with the subject of the token.

It's optional to use it, but that's where you would put the user's identification, so we are using it here.

JWT might be used for other things apart from identifying a user and allowing them to perform operations directly on your API.

For example, you could identify a "car" or a "blog post". Then you could add permissions about that entity, like "drive" (for the car) or "edit" (for the blog). And then, you could give that JWT token to a user (or bot), and they could use it to perform those actions (drive the car, or edit the blog post) without even needing to have an account, just with the JWT token your API generated for that.

Using these ideas, JWT can be used for way more sophisticated scenarios.

In those cases, several of those entities could have the same ID, let's say `foo` (a user `foo`, a car `foo`, and a blog post `foo`).

So, to avoid ID collisions, when creating the JWT token for the user, you could prefix the value of the `sub` key, e.g. with `username:`. So, in this example, the value of `sub` could have been: `username:johndoe`.

The important thing to keep in mind is that the `sub` key should have a unique identifier across the entire application, and it should be a string.

### Check it

Run the server and go to the docs: http://127.0.0.1:8000/docs

You'll see the user interface with an "Authorize" button. Authorize the application using the credentials:
- Username: `johndoe`
- Password: `secret`

Notice that nowhere in the code is the plaintext password "`secret`", we only have the hashed version.

Now use the operation `GET` with the path `/users/me/`. You will get your user's data.

If you open the developer tools, you could see how the data sent only includes the token, the password is only sent in the first request to authenticate the user and get that access token, but not afterwards.

Notice the header `Authorization`, with a value that starts with `Bearer`.

### Advanced usage with `scopes`

OAuth2 has the notion of "scopes".

You can use them to add a specific set of permissions to a JWT token. Then you can give this token to a user directly or a third party, to interact with your API with a set of restrictions.

You can learn how to use them and how they are integrated into **FastAPI** later in the **Advanced User Guide**.

### Recap

With what you have seen up to now, you can set up a secure **FastAPI** application using standards like OAuth2 and JWT.

In almost any framework handling the security becomes a rather complex subject quite quickly.

Many packages that simplify it a lot have to make many compromises with the data model, database, and available features. And some of these packages that simplify things too much actually have security flaws underneath.

**FastAPI** doesn't make any compromise with any database, data model or tool. It gives you all the flexibility to choose the ones that fit your project the best.

And you can use directly many well maintained and widely used packages like `pwdlib` and `PyJWT`, because **FastAPI** doesn't require any complex mechanisms to integrate external packages.

But it provides you the tools to simplify the process as much as possible without compromising flexibility, robustness, or security.

And you can use and implement secure, standard protocols, like OAuth2 in a relatively simple way.

You can learn more in the **Advanced User Guide** about how to use OAuth2 "scopes", for a more fine-grained permission system, following these same standards. OAuth2 with scopes is the mechanism used by many big authentication providers, like Facebook, Google, GitHub, Microsoft, X (Twitter), etc. to authorize third party applications to interact with their APIs on behalf of their users.