---
title: "关于JWT的一些概念"
date: 2018-01-15T22:34:05+08:00
draft: false
---

##JSON Web Token (JWT) 

### Introduction

1. JWT is a compact, URL-safe means of representing claims to be transferred between two parties.  The claims in a JWT are encoded as a JSON object that is used as the payload of a JSON Web Signature (JWS) structure or as the plaintext of a JSON Web Encryption (JWE) structure, enabling the claims to be digitally signed or integrity protected with a Message Authentication Code (MAC) and/or encrypted.

2. A string representing a set of claims as a JSON object that is encoded in a JWS or JWE, enabling the claims to be digitally signed or MACed and/or encrypted.

### JWT Claims Set
 
 A JSON object that contains the claims conveyed by the JWT.

### Claim

A piece of information asserted about a subject.  A claim is represented as a name/value pair consisting of a Claim Name and a Claim Value.

### Claim Name

The name portion of a claim representation. A Claim Name is always a string.

### Claim Value

The value portion of a claim representation. A Claim Value can be any JSON value.

### Unsecured JWT
     
A JWT whose claims are not integrity protected or encrypted.


## JSON Web Signature (JWS)

1. JSON Web Signature (JWS) represents content secured with digital signatures or Message Authentication Codes (MACs) using JSON-based data structures.  Cryptographic algorithms and identifiers for use with this specification are described in the separate JSON Web Algorithms (JWA) specification and an IANA registry defined by that specification.  Related encryption capabilities are described in the separate JSON Web Encryption (JWE) specification.
	
2. A data structure representing a digitally signed or MACed message.

3. JWS represents digitally signed or MACed content using JSON data structures and base64url encoding. 
	
### JOSE Header

JSON object containing the parameters describing the cryptographic operations and parameters employed.  The JOSE (JSON Object Signing and Encryption) Header is comprised of a set of Header Parameters.

###  JWS Payload

The sequence of octets(八位字节数组) to be secured -- a.k.a. the message. The payload can contain an arbitrary sequence of octets.

### JWS Signature

Digital signature or MAC over the JWS Protected Header and the JWS Payload.

### Header Parameter
      
A name/value pair that is member of the JOSE Header.

### JWS Protected Header
      	
JSON object that contains the Header Parameters that are integrity protected by the JWS Signature digital signature or MAC operation. For the JWS Compact Serialization, this comprises the entire JOSE Header.  For the JWS JSON Serialization, this is one component of the JOSE Header.

### JWS Unprotected Header
      
JSON object that contains the Header Parameters that are not integrity protected.  This can only be present when using the JWS JSON Serialization.

### JWS Compact Serialization
     
A representation of the JWS as a compact, URL-safe string.

### JWS JSON Serialization

A representation of the JWS as a JSON object.  Unlike the JWS Compact Serialization, the JWS JSON Serialization enables multiple digital signatures and/or MACs to be applied to the same content. This representation is neither optimized for compactness nor URL-safe.

### Unsecured JWS
	      
A JWS that provides no integrity protection. Unsecured JWSs use the "alg" value "none".

## JSON Web Encryption (JWE)

1. JSON Web Encryption (JWE) represents encrypted content using JSON-based data structures. 

2. A data structure representing an encrypted and integrity-protected message.


