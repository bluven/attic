---
title: "JWT vs JWS vs JWE"
date: 2018-01-15T22:34:05+08:00
draft: false
---

JWT (JSON Web Tokens) is a compact, URL safe artifacts for passing the claims between client and the server. Unlike cookies that make sense in only in the web application context, JWT can be easily used both in web applications and RESTful APIs. They make sense for browser clients as well as command line or thick clients.
 
However, there seems to be bit of confusion between JWT and other two related standards - JWS (JSON Web Signature) and JWE (JSON Web Encryption). JWT vs JWS vs JWE - is a common confusion among the developers. This article tries to explain away the differences and explain when to use which.

## JWT

> JWTs represent a set of claims as a JSON object that is encoded in a JWS and/or JWE structure.

> Source: RFC 7519 https://tools.ietf.org/html/rfc7519

In other words, a **JWT** claims (i.e., claims represented in JSON format) could be sent either as JWS structure or JWE structure (or both; more on that later) depending on what you are trying to achieve. So, when you are sending JWT to someone, it is essentially either a JWS payload or a JWE payload. JWS payload is more popular for reasons that will become obvious soon.

So what are claims? They are simple assertions about the user. As an example, for a user authenticated to payroll system, the claims could be as follows:

```
{    
	"userId": "johndoe",    
	"role": "employee",    
	"pay-frequency": "biweekly"
}
```

Now embedding this inside a JWS, the entire string would look like this:

```
{    
    "alg":"HS256"
} // End of JOSE Header
. 
{
    "userId": "johndoe",
    "role": "employee",
    "pay-frequency": "biweekly"
} // End of JWT Claims
.
{

} // End of Signature in case of JWS
```

## JWS

As the name suggests, JWS scheme digitally signs the content. The content to be carried here are JWT claims. The server signs the JWT and sends it to the client, say after successful user authentication. The server expects the client to send this JWS back to the server as part of the next request.
 
What if the client we're dealing with is rogue? That's where the signature comes in. Signature brings Integrity (remember the Confidentiality, Integrity, Availability triad?) and Authentication  into the equation. In other words, server can be sure that the JWT claims inside the JWS it just received were not tampered with either by the rogue client or by a man-in-the-middle.
 
Server achieves this by validating the signature of the message to ensure that the claims were not tampered with by the client. If the server detects any kind of tampering, it can take appropriate action (deny the request or block the client etc.).
 
The client can also validate the signature. To do so, the client either needs server's secret (if the JWT signature is HMAC) or needs server's public key (if the JWT was digitally signed).

> *Note: With JWS, the payload (claims portion) is not encrypted. So, don't send anything sensitive.*

> *注：JWS保证了数据的完整性，但不保证数据的保密性*

## JWE

The JWE scheme encrypts the content instead of signing it. The content being encrypted here are JWT claims. JWE, thus brings Confidentiality. The JWE can be signed and enclosed in a JWS. Now, you get both encryption and signature (thus getting Confidentiality, Integrity, Authentication).

## How to tell if you have a JWS or JWE?

Let's say you are the client and you receive a JWT. Now, how do you tell if it is a JWS or a JWE? From Section 9 of JWE RFC (https://www.rfc-editor.org/rfc/rfc7516.txt) :

* > If the object is using the JWS Compact Serialization or the JWE Compact Serialization, the number of base64url-encoded segments separated by period ('.') characters differs for JWSs and JWEs. JWSs have three segments separated by two period ('.') characters. JWEs have five segments separated by four period ('.') characters.

* > If the object is using the JWS JSON Serialization or the JWE JSON       Serialization, the members used will be different.  JWSs have a "payload" member and JWEs do not.  JWEs have a "ciphertext" member and JWSs do not.
      
* > The JOSE Header for a JWS can be distinguished from the JOSE Header for a JWE by examining the "alg" (algorithm) Header Parameter value. If the value represents a digital signature or MAC algorithm, or is the value "none", it is for a JWS; if it represents a Key Encryption, Key Wrapping, Direct Key Agreement, Key Agreement with Key Wrapping, or Direct Encryption algorithm, it is for a JWE. (Extracting the "alg" value to examine is straightforward when using the JWS Compact Serialization or the JWE Compact Serialization and may be more difficult when using the JWS JSON Serialization or the JWE JSON Serialization.)

* > The JOSE Header for a JWS can also be distinguished from the JOSE Header for a JWE by determining whether an "enc" (encryption algorithm) member exists. If the "enc" member exists, it is a JWE; otherwise, it is a JWS.

*原文： https://securedb.co/community/jwt-vs-jws-vs-jwe/*

