# JSON Web Tokens (JWT): History and FastAPI Implementation

## What are JSON Web Tokens?

JSON Web Tokens (JWT) are a compact, URL-safe means of representing claims to be transferred between two parties. The claims in a JWT are encoded as a JSON object that is used as the payload of a JSON Web Signature (JWS) structure or as the plaintext of a JSON Web Encryption (JWE) structure, enabling the claims to be digitally signed or integrity protected with a Message Authentication Code (MAC) and/or encrypted.

## JWT Structure

A JWT consists of three parts separated by dots (`.`):

```
header.payload.signature
```

- **Header**: Contains metadata about the token, typically the algorithm used and token type
- **Payload**: Contains the claims (statements about an entity and additional data)
- **Signature**: Used to verify the token hasn't been tampered with

## History of JWT

- **2010**: Initial concept developed as part of OAuth 2.0 and OpenID Connect specifications
- **2015**: RFC 7519 standardized JWT as an IETF standard
- **Present**: Widely adopted for stateless authentication in web applications, APIs, and microservices

## HS256 Algorithm Deep Dive

### Overview
The HS256 algorithm is a symmetric signing algorithm that uses HMAC (Hash-based Message Authentication Code) with SHA-256 cryptographic hash function.

### How HS256 Works
1. **Secret Key**: A single secret key is shared between sender and receiver
2. **Hashing**: The sender combines the message (JWT header and payload) with the secret key
3. **HMAC Generation**: The combined message is processed by SHA-256 to create a 256-bit hash
4. **MAC Creation**: This hash becomes the Message Authentication Code (MAC) appended to the message
5. **Verification**: The receiver uses the same secret key to recalculate the MAC for verification

### Key Characteristics
- **Symmetric**: Uses a single shared secret key for both signing and verification
- **HMAC-SHA256**: Combination of HMAC standard and SHA-256 hashing algorithm
- **MAC Generation**: Produces a Message Authentication Code, not a digital signature
- **Efficiency**: High computational efficiency and ease of implementation

### When to Use HS256
- **Simple Infrastructure**: Suitable for straightforward environments where performance is key
- **Shared Secrets**: Appropriate when all parties can securely manage the same secret key

### Security Considerations
- **Secret Key Management**: Critical to protect the secret key from unauthorized access
- **Algorithm Confusion Attacks**: Vulnerability when systems expect asymmetric algorithms (RS256) but receive HS256 tokens using public key as secret

## JWT in FastAPI Applications

### Core Implementation

Based on the FastAPI template security implementation:

```python
from datetime import datetime, timedelta, timezone
import jwt
from app.core.config import settings

ALGORITHM = "HS256"

def create_access_token(subject: str | Any, expires_delta: timedelta) -> str:
    expire = datetime.now(timezone.utc) + expires_delta
    to_encode = {"exp": expire, "sub": str(subject)}
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt
```

### FastAPI Security Features

FastAPI provides comprehensive OAuth2 and JWT support through:

1. **OAuth2 Password Bearer**: For handling OAuth2 "password" flow
2. **Security Dependencies**: Injectable security functions
3. **Current User Extraction**: Middleware for extracting authenticated users
4. **JWT Token Handling**: Built-in support for token validation and parsing

### Common FastAPI JWT Patterns

#### 1. Token Creation
```python
access_token_expires = timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
access_token = create_access_token(
    subject=user.email,
    expires_delta=access_token_expires
)
```

#### 2. Token Validation
```python
def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    return user
```

#### 3. Protected Endpoints
```python
@app.get("/protected")
async def protected_route(current_user: User = Depends(get_current_user)):
    return {"message": f"Hello {current_user.email}"}
```

## Best Practices for JWT in FastAPI

### Security
- Use strong, randomly generated secret keys
- Implement proper token expiration
- Store sensitive data in secure environment variables
- Use HTTPS in production
- Implement token refresh mechanisms

### Performance
- Keep payload minimal
- Use efficient serialization
- Implement proper caching strategies
- Consider token blacklisting for logout functionality

### Error Handling
- Provide clear error messages for authentication failures
- Implement proper exception handling
- Log security events appropriately

## Advantages of JWT

- **Stateless**: No need to store session data server-side
- **Scalable**: Works well in distributed systems
- **Cross-domain**: Can be used across different domains
- **Self-contained**: All necessary information is in the token
- **Standard**: Based on open standards (RFC 7519)

## Disadvantages of JWT

- **Size**: Larger than traditional session IDs
- **Revocation**: Difficult to revoke before expiration
- **Secret Management**: Requires careful handling of signing secrets
- **Information Exposure**: Claims are only base64 encoded, not encrypted

## Resources

- [FastAPI Security Tutorial](https://fastapi.tiangolo.com/tutorial/security/)
- [OAuth2 Implementation](https://fastapi.tiangolo.com/tutorial/security/simple-oauth2/)
- [JWT with FastAPI](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)
- [RFC 7519 - JSON Web Token](https://tools.ietf.org/html/rfc7519)