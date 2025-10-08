# FastAPI Security Tutorial

## Overview

This document transcribes the FastAPI security tutorial covering OAuth2, JWT tokens, password hashing, and authentication patterns.

## Security Fundamentals

Security, authentication and authorization can be complex topics. In many frameworks, handling security takes a significant amount of effort and code (often 50% or more of all code written).

FastAPI provides several tools to help you deal with security easily, rapidly, in a standard way, without having to study and learn all the security specifications.

### Key Concepts

**OAuth2** is a specification that defines several ways to handle authentication and authorization. It's an extensive specification covering several complex use cases, including ways to authenticate using a "third party" (like "login with Facebook, Google, X (Twitter), GitHub").

**OAuth 1** was very different from OAuth2, more complex, and included direct specifications on how to encrypt communication. It's not very popular nowadays. OAuth2 doesn't specify how to encrypt communication - it expects you to have your application served with HTTPS.

**OpenID Connect** is another specification based on OAuth2. It extends OAuth2 by specifying some things that are relatively ambiguous in OAuth2, to try to make it more interoperable. Google login uses OpenID Connect (which underneath uses OAuth2), but Facebook login doesn't support OpenID Connect.

**OpenAPI** (previously known as Swagger) is the open specification for building APIs. FastAPI is based on OpenAPI, which makes it possible to have multiple automatic interactive documentation interfaces, code generation, etc.

OpenAPI defines the following security schemes:
- `apiKey`: an application specific key that can come from a query parameter, header, or cookie
- `http`: standard HTTP authentication systems, including `bearer` (a header `Authorization` with a value of `Bearer` plus a token) and HTTP Basic authentication
- `oauth2`: all the OAuth2 ways to handle security (called "flows")
- `openIdConnect`: defines how to discover OAuth2 authentication data automatically

## First Steps with Security

### Basic OAuth2 Setup

Let's start with a simple example using OAuth2 with the Password flow and Bearer tokens.

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

### The Password Flow

The `password` flow is one of the ways defined in OAuth2 to handle security and authentication. Here's how it works:

1. The user types username and password in the frontend and hits Enter
2. The frontend sends that username and password to a specific URL in our API (declared with `tokenUrl="token"`)
3. The API checks the username and password, and responds with a "token"
4. The frontend stores that token temporarily
5. When the user needs to access another section, the frontend sends a header `Authorization` with a value of `Bearer` plus the token

### OAuth2PasswordBearer

When we create an instance of the `OAuth2PasswordBearer` class, we pass in the `tokenUrl` parameter. This parameter contains the URL that the client will use to send the username and password to get a token.

```python
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
```

The `oauth2_scheme` variable is an instance of `OAuth2PasswordBearer`, but it's also a "callable" that can be used with `Depends`.

## Creating User Models and Dependencies

### User Model

First, create a Pydantic user model:

```python
from pydantic import BaseModel

class User(BaseModel):
    username: str
    email: str | None = None
    full_name: str | None = None
    disabled: bool | None = None
```

### Get Current User Dependency

Create a dependency `get_current_user` that will have a dependency with the same `oauth2_scheme`:

```python
def fake_decode_token(token):
    return User(
        username=token + "fakedecoded", 
        email="john@example.com", 
        full_name="John Doe"
    )

async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]):
    user = fake_decode_token(token)
    return user

@app.get("/users/me")
async def read_users_me(current_user: Annotated[User, Depends(get_current_user)]):
    return current_user
```

## Simple OAuth2 Implementation

### Getting Username and Password

OAuth2 specifies that when using the "password flow", the client/user must send `username` and `password` fields as form data. The spec also says the client can send another form field called `scope`.

### OAuth2PasswordRequestForm

Use `OAuth2PasswordRequestForm` as a dependency to get username and password:

```python
from fastapi.security import OAuth2PasswordRequestForm

@app.post("/token")
async def login(form_data: Annotated[OAuth2PasswordRequestForm, Depends()]):
    user_dict = fake_users_db.get(form_data.username)
    if not user_dict:
        raise HTTPException(status_code=400, detail="Incorrect username or password")
    user = UserInDB(**user_dict)
    hashed_password = fake_hash_password(form_data.password)
    if not hashed_password == user.hashed_password:
        raise HTTPException(status_code=400, detail="Incorrect username or password")
    
    return {"access_token": user.username, "token_type": "bearer"}
```

### Password Hashing

"Hashing" means converting content (a password) into a sequence of bytes that looks like gibberish. You get exactly the same gibberish for exactly the same content, but you cannot convert from the gibberish back to the password.

**Why use password hashing**: If your database is stolen, the thief won't have your users' plaintext passwords, only the hashes. The thief won't be able to try to use those same passwords in another system.

### Complete Simple Implementation

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel

fake_users_db = {
    "johndoe": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "johndoe@example.com",
        "hashed_password": "fakehashedsecret",
        "disabled": False,
    },
    "alice": {
        "username": "alice",
        "full_name": "Alice Wonderson",
        "email": "alice@example.com",
        "hashed_password": "fakehashedsecret2",
        "disabled": True,
    },
}

app = FastAPI()

def fake_hash_password(password: str):
    return "fakehashed" + password

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class User(BaseModel):
    username: str
    email: str | None = None
    full_name: str | None = None
    disabled: bool | None = None

class UserInDB(User):
    hashed_password: str

def get_user(db, username: str):
    if username in db:
        user_dict = db[username]
        return UserInDB(**user_dict)

def fake_decode_token(token):
    user = get_user(fake_users_db, token)
    return user

async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]):
    user = fake_decode_token(token)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return user

async def get_current_active_user(
    current_user: Annotated[User, Depends(get_current_user)],
):
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

@app.post("/token")
async def login(form_data: Annotated[OAuth2PasswordRequestForm, Depends()]):
    user_dict = fake_users_db.get(form_data.username)
    if not user_dict:
        raise HTTPException(status_code=400, detail="Incorrect username or password")
    user = UserInDB(**user_dict)
    hashed_password = fake_hash_password(form_data.password)
    if not hashed_password == user.hashed_password:
        raise HTTPException(status_code=400, detail="Incorrect username or password")
    
    return {"access_token": user.username, "token_type": "bearer"}

@app.get("/users/me")
async def read_users_me(
    current_user: Annotated[User, Depends(get_current_active_user)],
):
    return current_user
```

## OAuth2 with JWT Tokens and Password Hashing

### About JWT

JWT means "JSON Web Tokens". It's a standard to codify a JSON object in a long dense string without spaces. It's not encrypted, so anyone could recover the information from the contents, but it's signed. When you receive a token that you emitted, you can verify that you actually emitted it.

You can create a token with an expiration of, say, 1 week. When the user comes back the next day with the token, you know that user is still logged in to your system. After a week, the token will be expired and the user will not be authorized.

### Installing Dependencies

Install PyJWT to generate and verify JWT tokens:
```bash
pip install pyjwt
```

Install pwdlib for password hashing:
```bash
pip install "pwdlib[argon2]"
```

### Password Hashing with pwdlib

```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()

def verify_password(plain_password, hashed_password):
    return password_hash.verify(plain_password, hashed_password)

def get_password_hash(password):
    return password_hash.hash(password)
```

### JWT Token Handling

Generate a secret key:
```bash
openssl rand -hex 32
```

Create JWT token functions:

```python
from datetime import datetime, timedelta, timezone
import jwt
from jwt.exceptions import InvalidTokenError

SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(data: dict, expires_delta: timedelta | None = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt
```

### Complete Secure Implementation

```python
from datetime import datetime, timedelta, timezone
from typing import Annotated
import jwt
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jwt.exceptions import InvalidTokenError
from pwdlib import PasswordHash
from pydantic import BaseModel

SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

fake_users_db = {
    "johndoe": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "johndoe@example.com",
        "hashed_password": "$argon2id$v=19$m=65536,t=3,p=4$wagCPXjifgvUFBzq4hqe3w$CYaIb8sB+wtD+Vu/P4uod1+Qof8h+1g7bbDlBID48Rc",
        "disabled": False,
    }
}

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: str | None = None

class User(BaseModel):
    username: str
    email: str | None = None
    full_name: str | None = None
    disabled: bool | None = None

class UserInDB(User):
    hashed_password: str

password_hash = PasswordHash.recommended()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

app = FastAPI()

def verify_password(plain_password, hashed_password):
    return password_hash.verify(plain_password, hashed_password)

def get_password_hash(password):
    return password_hash.hash(password)

def get_user(db, username: str):
    if username in db:
        user_dict = db[username]
        return UserInDB(**user_dict)

def authenticate_user(fake_db, username: str, password: str):
    user = get_user(fake_db, username)
    if not user:
        return False
    if not verify_password(password, user.hashed_password):
        return False
    return user

def create_access_token(data: dict, expires_delta: timedelta | None = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except InvalidTokenError:
        raise credentials_exception
    user = get_user(fake_users_db, username=token_data.username)
    if user is None:
        raise credentials_exception
    return user

async def get_current_active_user(
    current_user: Annotated[User, Depends(get_current_user)],
):
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

@app.post("/token")
async def login_for_access_token(
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()],
) -> Token:
    user = authenticate_user(fake_users_db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username}, expires_delta=access_token_expires
    )
    return Token(access_token=access_token, token_type="bearer")

@app.get("/users/me/", response_model=User)
async def read_users_me(
    current_user: Annotated[User, Depends(get_current_active_user)],
):
    return current_user

@app.get("/users/me/items/")
async def read_own_items(
    current_user: Annotated[User, Depends(get_current_active_user)],
):
    return [{"item_id": "Foo", "owner": current_user.username}]
```

## Key Points

### JWT Subject (sub)

The JWT specification says there's a key `sub` with the subject of the token. It's optional but that's where you would put the user's identification. The `sub` key should have a unique identifier across the entire application and should be a string.

### Security Best Practices

1. **Never save plaintext passwords** - always hash them
2. **Use secure secret keys** - generate them with `openssl rand -hex 32`
3. **Set token expiration** - tokens should expire for security
4. **Use HTTPS in production** - OAuth2 expects encrypted communication
5. **Validate tokens properly** - check signatures and expiration

### Token Response Format

The response of the token endpoint must be a JSON object with:
- `token_type`: should be "bearer" for Bearer tokens
- `access_token`: string containing the access token

### Advanced Usage

OAuth2 has the notion of "scopes" that can be used to add specific sets of permissions to JWT tokens. You can give tokens to users or third parties to interact with your API with a set of restrictions.

## Summary

FastAPI provides tools to implement secure authentication systems using standard protocols like OAuth2 and JWT without making compromises with databases, data models, or tools. The framework gives you flexibility to choose the best tools for your project while providing utilities to simplify the security implementation process.

With the patterns shown above, you can implement:
- OAuth2 password flow authentication
- JWT token generation and validation
- Secure password hashing
- User authentication and authorization
- Token-based API access control

The system is compatible with standard OAuth2 flows and can be extended to work with third-party authentication providers like Google, Facebook, GitHub, etc.