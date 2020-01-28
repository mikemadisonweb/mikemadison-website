---
title: Session handling with JWT
date: 2019-04-25 12:00:00
featured_image: session-handling-with-jwt.jpg
desc: JWT vs cache-based session handling comparison
tags:
- JSON
- JWT
- Session
---
Json Web Token (JWT) is a standard that defines a way how to securely transmit valuable data between distributed systems as a set of claims encoded as a JSON object ([RFC7519](https://tools.ietf.org/html/rfc7519)). JWT is not designed specifically for the user authorization, but for the secure data transmission between two parties in general. There are two the most widely used implementations of the JWT: JSON Web Signature and JSON Web Encoding.
<!--more-->
## JSON Web Signature (JWS)
The data integrity is guaranteed by the digital signature. The token, in this case, consists of three parts:
```js
// Javascript Object Signing and Encryption (JOSE) Headers
{   
    "alg":"HS256"
}
// JWT Claims (Payload)
{
    "userId": 12234,
    "firstName": "John",
    "lastName": "Doe",
    "exp": 1516239022
}
// Signature
GuoUe6tw79bJlbU1HU0ADX0pr0u2kf3r_4OdrDufSfQ
```
Headers provide meta information about the token. The payload may contain your arbitrary key-value data(private claims), but also could contain pre-defined keys, so-called [registered claim names](https://tools.ietf.org/html/rfc7519#section-4.1), for example, "iat" (Issued At), "exp" (Expiration time), "sub" (Subject) etc. The signature is a hash string generated using the mentioned in the headers algorithm and a secret key securely stored on the sever-side.

These values are encoded with URL-safe Base64 schema and concatenated together separated by dots:
```text
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.yKOB4jkGWu7twu8Ts9zju01E10_CPedLJkoJFCan5J4
```
In this form, it could be used by the client to prove his identity. The general use-cases include:
 - pass to frontend application after the username/password authentication
 - pass as a cookie, like a usual stateful session ID
 - pass between distributed RESTful APIs as an HTTP header
 
It is important to note that JWS implementation is signed and encoded but not encrypted anyhow. That way, the payload of the token could be easily read by all of the participants. However, if a malicious user would try to tamper the content, the token would be rejected by the server as it would not match the signature anymore.

## JSON Web Encoding (JWE)
The JWE scheme encrypts the content instead of signing it. That way it guarantees confidentiality. 
So in order to hide the payload content of the token, JWE could be used separately or in conjunction with JWS. To identify the JWE, one should look at the "enc" (Encryption algorithm) value in the JOSE headers. If the entry exists and not empty, it is a JWE.
There are two serialized forms to represent the encrypted payload: the JWE compact serialization and JWE JSON serialization. For those who are interested especially in JWE, I would recommend reading [this blog post](https://medium.facilelogin.com/jwt-jws-and-jwe-for-not-so-dummies-b63310d201a3).
Compact serialization contain five base64Url-encoded parts separated by dots:
 - JOSE header
 - JWE encrypted Key
 - JWE initialization vector
 - JWE ciphertext
 - JWE Authentication Tag

JSON serialization is represented as a JSON object containing some or all of these eight entries:

 - "protected" *(JWE Protected Header)*
 - "unprotected" *(JWE Shared Unprotected Header)*
 - "header" *(JWE Per-Recipient Unprotected Header)*
 - "encrypted_key" *(JWE Encrypted Key)*
 - "iv" *(JWE Initialization Vector)*
 - "ciphertext" *(JWE Ciphertext)*
 - "tag" *(JWE Authentication Tag)*
 - "aad" *(JWE Additional authenticated data)*

## Usage
This is a JWT usage example of short-term tokens needed for role-based CDN content access.
```mermaid
sequenceDiagram
    participant c as User
    participant a as Auth server
    participant b as CDN
    c->>a: POST /grant-access
    activate a
    note right of a:Check<br>the permissions<br>based on the session
    a-->>c: [200] Generated JWT
    deactivate a
    c->>b: POST /get-content
    activate b
    Note right of b: CDN validate<br>the provided JWT,<br>no session storage<br>is needed
    b-->>c: [200] Content
    deactivate b
```

## Pros and cons of JWT
#### Pros
 - All authentication logic could be isolated in the dedicated auth-server. Only this server will have the private key, and the rest of the servers will have the public key to verify the signature. If you have a microservice application, it promotes better architecture, because there is no direct connection between the auth-server and the rest of the servers. The authentication is done by the auth-server, and any internal API, which knows nothing about the user session, could do the authorization.
 - If properly implemented, the solution if fully stateless, meaning no shared session storage or sticky session solutions are needed. As a bonus, it reduces the number of queries and sometimes eliminates the need in the persistence layer at all.
 - Authorization when working with third-party services.
 - One session could be simultaneously conducted on multiple devices at the same time.

#### Cons
 - The size of the token could grow out of proportions really fast if the new data is added to the token payload without control. The size of the token is adding to the size of each request, propagated to each of the microservices, and it could be an issue for the heavy load applications. Just the token containing the userId is nearly 50 times bigger than the plainly passed userId (depends on used JOSE headers and claims), so in the best scenario, the amount of data passed in the token should be small, and the extension should be forbidden.
 - There is no way to invalidate users partially. If JWT contains user roles, for example, or other information that could change in the process, corresponding JWT would still contain the old data until the token is expired. From the other side, if the secret key was compromised by a careless or a rogue developer/administrator, all of the users need to log out for security purposes. It could be an issue for big applications.
 - When implemented badly, it is nothing more than to the same plain-old stateful session handling, only in the fancy JWT wrapper. If you need to request the database to retrieve user information in nearly every API that receives a request with JWT, it is a bad sign.

## Security-related considerations
Things that need to be done always:
 - Both headers and payload must be signed.
 - Choose a strong algorithm. The worst case is 'None', which means the signature will not be checked. One should prefer 'HS\*\*\*' to 'RS\*\*\*' algorithms, if possible.
 - Store it properly on the client-side. The only good practice is to store it in a cookie, with usual restrictions like path, scope, etc.
 - Don't put sensitive data in the JWS payload. These include passwords, since headers and payload are only base64encoded and not encrypted, so everyone can decode the content of JWT to get its cleartext data. Use JWE if sensitive data transmission is necessary.
 - Use HTTPS for all of the requests.

## Conclusion
By introducing JWT instead of the "old-fashioned" stateful solution you can benefit from the following:
 - No need in a cross-cluster replicated session storage.
 - Propagation of the same user data to the internal APIs using a single middleware. However, JWT is not a single way how to achieve this.
 - Each internal API could validate the token without any external dependency.

What I am trying to convey is to consider these points or make up your own before deciding to use JWT. If you have a stateful session handling already implemented, just consider the amount of refactoring that would be needed. It may be not worth the efforts. I am not against JWT at all. Some developers are confusing the idea of JWT to be a substitute for the stateful sessions. It is not true. JWT is not a new shiny way to handle sessions. It is a standard of creating tokens, which by the way, could be used for sessions, but not only. Stateless session handling makes sense in some cases, did not make sense in others. Just think about your specific situation first to make the right decision.
