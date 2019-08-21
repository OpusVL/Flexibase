# Flexibase Core Service Specification

This document specifies core services recommended for a Flexibase system to be
considered complete. None of these services is strictly required, but it is
intended as a guideline for base functionality.

## Terms and document layout

Each second-level heading (except this one) defines a service using its human
name. Each service's definition has an English prologue explaining the purpose
and design of the service.

This is followed by a specification with several paragraphs, each defining a
feature of the service. These use the familiar MUST, MUST NOT, MAY, MAY NOT,
SHOULD, and SHOULD NOT qualifiers from RFCs.

The first of these will specify the name of the service within the Hive. The
name of the Mind itself is irrelevant; an implementation detail. However,
unless there is some reason not to, this might as well be the same as the
service name.

It is not recommended to implement more than one of these services on the same
Mind. This is to prevent unnecessarily large amounts of constructor parameters
(and to separate those concerns).

After these paragraphs the interface is defined. In this section the term `type`
is used to define a data type used within the rest of the interface definition.
This is not to be confused with the term Types used later on.

Following the lists of data types is a list of interface functions. Each
function's parameters and return type is formally defined here, plus an English
paragraph defining the purpose and behaviour of the function.

The interface specification may be followed by a Types section, defining types
contained within the service that are actually exposed to the Hive by means of
the type registry. Usually these types will be tied to one of the types used in
the interface definition but this is not necessarily always the case.

### Function specifications

Each interface function is specified thus:

    function-name[(parameter-list)] -> return-type

Where `return-type` is `TypeName`; `parameter-list` is:

    parameter[, parameter ...]

And `parameter` is:

    TypeName parameter-name[?][= default-value]

A `?` denotes an optional parameter with no default. `= default-value` denotes
an optional parameter *with* a default. The presence of neither denotes a
required parameter.

`TypeName` will either be some hopefully recognisable basic data type (Str,
Num, Bool, etc), or one of the data types defined above the function
specification in `type` headings.

To allow for differences in language capabilities and conventions, these are
usually kept intentionally vague.

It is also not prescribed whether the implementation uses keyword arguments or
positional arguments. Implementers are free to select whichever language feature
makes most sense.

### A note on data types

Some data types are mentioned without fanfare. I will list them here.

#### type `OpenAPI`

Any data structure that contains an OpenAPI v3 schema. With OpenAPI schemata
being JSON objects, this would represent whatever a "deserialised JSON object"
turns out to be in the host language.

### A note on semantic Types

Each **Types** section defines a set of *semantic* types that the service should
register with Flexibase's type registry. In each, we use `::` to separate
namespaces. This is a convention found in several languages but SHOULD NOT be
translated into the convention for the implementation's language.

The reason for this is that if we ensure that types are named the same across
languages, we can theoretically have Flexibase applications communicate through
a connector interface that farms off any given service to another Flexibase.

As a result, we can expect the name of these types to leave one Flexibase system
and arrive in another, potentially written in a different language.  This means
that the naming of the types follows Flexibase conventions, not local language
conventions.

## Authentication

Authentication is simply the requirement that a set of credentials be verified.
We prescribe no structure to these credentials.

Upon a verification attempt the service must respond with a structured response.
We do prescribe this structure.

The `auth` service therefore

* Accepts arbitrary data
* Returns a data structure defining a response

We want to support an arbitrary method of authentication. This includes
various sources of user/password authentication, password-free authentication,
and multi-factor authentication.

Therefore, we define various alternative responses for the authentication
attempt, all of which can be encoded in the object returned.

We also define a pre-auth step, allowing the auth system to seed the login UI
with data, for example the current value of an OTP.

In each auth step (including preauth) we communicate to the caller two pieces of
information: a `schema` for data that must be produced for the next step, and a
`payload` for data that must be *returned* to the next step verbatim.

Every auth service is expected to respond to `preauth` with a schema for the
data structure this auth type requires. It is unlikely that an auth system would
work without accepting auth data!

Every auth service is expected to also respond to `auth`, with an a structured
response object. This object may return another schema - rinse and repeat!

Each auth service should concern itself with a single type of authentication. A
Flexibase application can string together authentication forms by defining its
own authentication service that forwards `auth` calls to one service, produces a
token, and returns the structure required by the second service.

Here's a pseudocode example:

```
pre-auth()
    -> [custom auth service]
        -> password-auth.pre-auth()
        <- schema{user, password}
    <- schema{user, password}
auth(preauth-data)
    -> [custom auth service]
        -> password-auth.auth(preauth-data)
        <- success
        -> 2fa.pre-auth()
        <- schema{code}, payload{challenge}
        generate session token
        store 2fa challenge in session
    <- schema{code}, payload{session-token}
auth(code, session-token)
    -> [custom auth service]
        recover 2fa challenge from session
        -> 2fa.auth(code, challenge)
        <- success
    <- success
logged in!
```

### Specification

> The service in the Hive MUST be `auth`.

> An auth service MUST implement a pre-auth method, providing an OpenAPI schema
> for the initial request.

> An auth service MUST implement an auth method, accepting data conforming to
> the schema returned by pre-auth.

> The auth method MAY return another schema instead of a success or failure.

> The pre-auth method and auth methods MAY return payload data.

> Calling code MUST allow the 'next step' response to be returned.

> Calling code MUST send the payload data back in the next auth request.

> The auth method MUST know what payload to expect and fail if it is not
> provided.

> The auth system SHOULD use exceptions to indicate that the caller made an
> error; returning from the auth method SHOULD only indicate success, bad
> credentials, or next step. (This is a SHOULD because not all languages
> implement an exception system.)

#### type `AuthData`

An arbitrary data structure. Used to represent data collected from the user as
part of a login attempt.

#### type `AuthResponse`

A key/value type with the following properties:

`AuthResult result`: Any identifiable data type (e.g. an enumeration) to
determine whether the result was one of "success", "failure", or "next step".

"Next step" tells the system that to authenticate, another attempt must be made
with further data. The `schema` and probably `payload` will be used by the login
system to perform the next step.

`OpenAPI schema`: An OpenAPI schema defining the data that this step requires
from the user.

`struct payload`: Data that must be sent verbatim back to the system when auth
is requested.

> The use of properties in this specification is an implementation detail. Some
> languages may prefer to use subtypes of AuthResponse to differentiate between
> the three auth result options.

#### type `AuthResult`

Anything that is meaningful in the host language to identify the result of the
authentication attempt. The simplest example is just a 3-element subset of Str.
A more robust example would be three separate data types: `AuthResponseSuccess
isa AuthResponse`. As usual, practicality in context of the host language should
influence this decision.

#### `preauth -> AuthResponse`

The entrypoint into an auth system is to call preauth. This returns the initial
OpenAPI schema, and possibly payload, to begin a login attempt.

#### `auth(AuthData data, struct payload?) -> AuthResponse`

Perform authentication. `payload` contains any data provided to the login system
in the `AuthResponse` from before.

This will loop as long as `AuthResponse` indicates "next step"!

### Types

* `auth::user` - The user object that represents the logged-in user (or user
  attempting to log in).

> Fundamentally, a user has no information that we can define at this point, so
> this user object does not "exist" in so many words. However, simply defining
> the name of a type is sufficient for other systems to make use of it, so we
> do.

## Session

The session service simply supplies serialised assurance of state. This is done
by means of a token and some form of storage.

The service can either generate a token (for example, on a success response from
the `auth` service), or validate that a token is legitimate. It can also be
requested to terminate a token.

Tokens are active within realms. A realm is simply any arbitrary identifier for
a, well, realm of behaviour. This ensures that a correct session token from one
realm is not considered correct in another. A default realm is supplied if not
specified.

Two realms in the same system, for example, may be `auth` and `cookie`; one to
remain logged in, and another to store cookie data. Different realms may have
different rules on token expiry.

The service will associate a supplied payload with the token it generates, and
allow access to this payload via the token. Users of the service should take
care to only store as little data as possible, to avoid the sorts of problems
that can come with serialising oversized payloads.

### Specification

> The service MUST be called `session` within the Hive.

> The session service MUST provide an interface to generate a token.

> The generator function MUST require a payload parameter.

> The generator function MUST accept an optional realm parameter.

> The service MUST store the provided payload, and return a token to identify
> it.

> The token itself MUST be cryptographically secure.

> The session service MUST provide an interface to retrieve stored data.

> The retrieval function MUST accept a token of the same type as the generator
> function returns.

> The retrieval function MUST accept an optional realm parameter.

> The retrieval function MUST return functionally identical data to those
> provided by the user of the generator function.

> Hopefully obviously, the returned data MUST be that data that was provided
> when the generataor function returned the supplied token.

> The retrieval function MUST NOT return any data if the token is considered
> expired.

> The retrieval function MUST NOT return any data if the token exists in storage
> but against a different realm.

> The retrieval function SHOULD use an exception to indicate that the token is
> invalid.

> The service MUST implement a function to test the validity of a token.

> If no realm is supplied, a default one is provided; the function MUST NOT
> provide a means to search across realms.

> The session service SHOULD implement a timeout feature. This means that if a
> session token is used after a certain amount of real time has passed, even if
> it is valid it will be considered invalid. Requirements marked * only apply to
> session services that implement timeouts.

> \* The generator function MUST accept an optional timeout parameter. This will
>   default to the timeout set by the realm provided in the realm parameter, or
>   of course the default realm if this is also not provided. The returned token
>   will not be available after this much time has passed.

> \* Functions MUST allow for sessions not to time out at all. Common values to
>   indicate this are `0`, `-1`, or some variation on the theme of `Nil`.

> \* The retrieval function MUST reset the timer for the timeout function on
>   retrieval. (The timeout function is based on access time, not creation
>   time.)

> \* The retrieval function MUST return the same response for a timed-out token
>   as for an invalid one.

> \* The service configuration SHOULD allow each realm to define a default
>   timeout.

#### type `SessionToken`

Common representations of crypto-secure data include hex and base64, both of
which are subsets of the String type. Implementations may wish to use something
consistent, e.g. 32-character hex strings.

> Implementations should bear in mind that some uses of these tokens will leave
> the system, like session cookies. They should be short enough to be used in
> most common situations without causing issues.

#### type `SessionData`

An arbitrary, serialisable data structure.

#### `store(SessionData data, Number timeout?, Str realm = 'default') -> SessionToken`

Store this data and return a session token. `timeout` may be interpreted as
relevant to the language in question; some languages use integers and assume
milliseconds; some languages assume seconds but allow decimals; and everything
in between.

`realm` is described above and is used to partition data to avoid
cross-pollination of separate concerns. While the argument is optional, a
default realm must exist.

#### `retrieve(SessionToken token, Str realm = 'default') -> SessionData`

Retrieve the data associated with this token. if the token is invalid (either
does not exist, or has timed out), returns no value. May throw an exception to
indicate invalidity of token.

The timeout of a token is based on its last access time so on a successful
access this function must reset the access time.

#### `checkToken(SessionToken token, Str realm = 'default') -> Bool`

Test the validity of this token in the given realm. Return `True` if the token
is valid, or else `False`.
