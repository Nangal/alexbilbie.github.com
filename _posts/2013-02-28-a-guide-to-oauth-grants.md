---
layout: default
published: true
title: "A guide to OAuth 2.0 grants"
permalink: /2013/02/a-guide-to-oauth-2-grants/
date: "2013-02-28 16:15:00"
---

OAuth 2.0 by it’s nature is a very flexible standard and can adapted to work in many different scenarios. The [core specification](http://tools.ietf.org/html/rfc6749) describes four authorisation grants:

* Authorisation code grant
* Implicit grant
* Resource owner credentials grant
* Client credentials grant

The specification also details another grant called the _refresh token grant_.

Furthermore there are a number of other grants that have gone through the IETF ratification process (none of which at the time of writing have been formally standardised):

* Message authentication code (MAC) tokens
* SAML 2.0 Bearer Assertion Profiles
* JSON web token grant

The end goal of each of these grants (except the refresh token grant) is for the client application to have an access token (which represents a user’s permission for the client to access their data) which it can use to authenticate a request to an API endpoint.

In this post I’m going to describe each of the above grants and their appropriate use cases.

As a refresher here is a quick glossary of OAuth terms (taken from the core spec):

* **Resource owner (a.k.a. the User)** - An entity capable of granting access to a protected resource. When the resource owner is a person, it is referred to as an end-user.
* **Resource server (a.k.a. the API server)** - The server hosting the protected resources, capable of accepting and responding to protected resource requests using access tokens.
* **Client** - An application making protected resource requests on behalf of the resource owner and with its authorisation. The term client does not imply any particular implementation characteristics (e.g. whether the application executes on a server, a desktop, or other devices).
* **Authorisation server** - The server issuing access tokens to the client after successfully authenticating the resource owner and obtaining authorisation.

## Authorisation code grant ([section 4.1](http://tools.ietf.org/html/rfc6749#section-4.1))

The authorisation code grant is the grant that most people think of when OAuth is described.

If you’ve ever signed into a website or application with your Twitter/Facebook/Google/(insert major Internet company here) account then you’ll have experienced using this grant.

Essentially a user will click on a “sign in with Facebook” (or other <attr title=“Identity Provider”>IdP</attr>) and then be redirected from the application/website (the “client”) to the IdP authorisation server. The user will then sign in to the IdP with their credentials, and then - if they haven’t already - authorise the client to allow it to use the user’s data (such as their name, email address, etc). If they authorise the request the user will be redirected back to the client with a token (called the authorisation code) in the query string (e.g. `http://client.com/redirect?code=XYZ123`) which the client will capture and exchange for an access token in the background.

This grant is suitable where the resource owner is a user and they are using a client which is allows a user to interact with a website in a browser. An obvious example is the client being another website, but desktop applications such as Spotify or Reeder use embedded browsers.

Some mobile applications use this flow and again use an embedded browser (or redirect the user to the native browser and then are redirected back to the app using a custom protocol).

In this grant the access token is kept private from the resource owner.

If you have a mobile application that is for your own service (such as the official Spotify or Facebook apps on iOS) it isn’t appropriate to use this grant as the app itself should already be trusted by your authorisation server and so the _resource owner credentials grant would be more appropriate.

## Implicit grant ([section 4.2](http://tools.ietf.org/html/rfc6749#section-4.2))

The implicit grant is similar to the authentication code grant described above. The user will be redirected in a browser to the IdP authorisation server, sign in, authorise the request but instead of being returned to the client with an authentication code they are redirected with an access token straight away.

The purpose of the implicit grant is for use by clients which are not capable of keeping the client’s own credentials secret; for example a JavaScript only application.

**If you decide to implement this grant then you must be aware that the access token should be treated as “public knowledge” (like a public RSA key)** and therefore it must have a very limited permissions when interacting with the API server. For example an access token that was granted using the authentication code grant could have permission to be used to delete resources owned by the user, however an access token granted through the implicit flow should only be able to “read” resources and never perform any destructive operations.

## Resource owner credentials grant ([section 4.3](http://tools.ietf.org/html/rfc6749#section-4.3))

When this grant is implemented the client itself will ask the user for their username and password (as opposed to being redirected to an IdP authorisation server to authenticate) and then send these to the authorisation server along with the client’s own credentials.  If the authentication is successful then the client will be issued with an access token.

This grant is suitable for trusted clients such as a service’s own mobile client (for example Spotify’s iOS app). You could also use this in software where it’s not easy to implement the authorisation code - for example we bolted this authorisation grant into [OwnCloud](http://owncloud.org/) so we could retrieve details about a user that we couldn’t access over LDAP from the university’s Active Directory server.

## Client credentials grant  ([section 4.4](http://tools.ietf.org/html/rfc6749#section-4.4))

This grant is similar to the resource owner credentials grant except only the client’s credentials are used to authenticate a request for an access token. Again this grant should only be allowed to be used by trusted clients.

This grant is suitable for machine-to-machine authentication, for example for use in a cron job which is performing maintenance tasks over an API. Another example would be a client making requests to an API that don’t require user’s permission.

When someone visits a member of staff’s page on the [University of Lincoln staff directory](http://staff.lincoln.ac.uk/) the website uses it’s own access token (that was generated using this grant) to authenticate a request to the API server to get the data about the member of staff that is used to build the page. When a member of staff signs in to update their profile however their own access token is used to retrieve and update their data. Therefore there is a good separation of concerns and we can easily restrict permissions that each type of access token has.

## Refresh token grant ([section 1.5](http://tools.ietf.org/html/rfc6749#section-1.5))

The OAuth 2.0 specification details a fifth grant which can be used to “refresh” (i.e. renew) an access token which has expired.

Authorisation servers which support this grant will also issue a “refresh token” when it returns an access token to a client. When the access token expires instead of sending the user back through the authorisation code grant the client can use to the refresh token to retrieve a new access token with the same permissions as the old one.

My problem with the grant is that it means the client has to maintain state of each token and then either on a cron job keep access tokens up to date or when it tries to make a request and it fails then go and update the access token and repeat the request.

I personally prefer to issue access tokens that last longer than the user’s session cookie so that when they next sign in they’ll be issued a new token anyway.

---

# The extension grants

The following grants are currently going through the standardisation process.

_N.B. My explanation from here on is how ** *I* ** (as a developer) am interpreting the specifications - all three of the following grants specs need further work in my opinion to better explain their purpose and how they compare to the core grants._

## MAC token grant ([draft 3 at time of writing](http://tools.ietf.org/html/draft-ietf-oauth-v2-http-mac))

The MAC token grant can be used alongside another grant (the specification describes using it alongside the authentication code grant) to further improve the security of requests to the API server by including the addition of a MAC (message authentication code) signature along with the access token in order to both authenticate the request and prove the identity of the client making the request (because it prevents tampering and forgery). In some ways it is similar to the OAuth 1.0 request process.

When the authorisation server returns an access token to a client it also includes a key which the client uses to generate the MAC signature. The MAC signature is generated by combining the parameters of the request and then hashing them against the key.

This grant could be used when additional security measures are needed to ensure that the client querying an API is definitely who it is identifying itself as. It can also prevent access tokens being used by unauthorised clients (if an access token has been compromised by a man in the middle attack) because a valid signature (which can only be generated by the client which knows the MAC key associated with the access token) is required for each request.

## SAML 2.0 Bearer Assertion Profiles ([draft 15 at time of writing](http://tools.ietf.org/html/draft-ietf-oauth-saml2-bearer))

A SAML assertion is an XML payload which contains a security token. They are issued by identity providers and consumed by a service provider who relies on its content to identify the assertion’s subject for security-related purposes. It is quite literally an assertion as to who someone (or what something) is.

This grant allows a client to exchange an existing SAML assertion for an access token, meaning the user doesn’t have to authenticate again with an authorisation server. In some ways it resembles the concept of the refresh token grant.

An example use case of this grant would be a publishers website receiving an assertion from the UK Federation (after a user from a UK education institution has authenticated with their institution’s own resource server), converting the assertion into an access token and then querying the users institution’s API server to request additional details about the user.

## JSON web token grant ([draft 6 at time of writing](http://tools.ietf.org/html/draft-ietf-oauth-json-web-token))

The JSON web token grant is similar to the SAML assertion grant but uses JSON as the serialisation format instead of XML. This means that a JWT (JSON web token) can comfortable fit inside a HTTP `authorization` header.

A potential use case could be a web based client application that also has a mobile version, the web version could somehow share the existing access token with the mobile application which in turn requests its own access token. Possibly? Maybe? I’ll admit I can’t think of a better example.