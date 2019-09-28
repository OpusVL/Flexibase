# Flexibase

## Synopsis

Service-oriented architecture is a scheme by which a system is built out of
separate, encapsulated components that communicate as services.

This mirrors the concept of microservices, except where microservices
communicate between processes, and often between hosts, SOA services can be
installed in-process.

Flexibase is a language-agnostic specification for a service-oriented
architecture, but expands upon the principle by demanding certain facilities be
met.

- A Flexibase implementation must be configurable such that an application
  developer can easily select services without having to write complex or
  tedious bootstrapping code.
- Components to a Flexibase application must communicate in very consistent
  ways. Flexibase (or an extension) defines service *types*, providing
  interfaces and protocols that an actual service must conform to.
- Components to a Flexibase system are not necessarily services, in the sense
  that they would not be called upon to perform actions by arbitrary parts of
  the system. Instead, the components may passively identify themselves as
  consuming a protocol whose actual controller may or may not be part of the
  system.
- Flexibase not only allows for the definition of behaviour but also of data
  types. Data types are defined along with the functional specification for
  service types, and there is a defined core service for registration and
  interrogation.

Often, when writing an application, the developer must first choose a paradigm
of user interface. There exist frameworks for web applications - both
browser-side and server side - for mobile devices, for desktop environments, for
terminal applications...

The problem with this approach is that the framework you select is the same
technology that defines how your modular code is actually put together. However
carefully you try to modularise your code you must always write some sort of
connector between your framework and your business logic, and any community code
you choose to install must be interfaced with in a specific way.

> For example, if you choose to write a web application, you must interface with
> your database in the way it prescribes. You probably will also install an
> authentication package and a session store for login, and then maybe try to
> integrate those with LDAP or OAuth2, or hack in 2FA later. Then, 2 years down
> the line, you realise that business logic has made its way into your web
> controllers and the only way to turn this into a cron job is to either
> refactor your entire application or create a script that fires mock requests
> at a copy of your app.

This makes as much sense as choosing a test framework first and then forcing all
your code to be written in terms of that.

Flexibase does not require you to decide what you are writing before you write
it. Instead, it encourages you to write services that can be employed by any
application, and defers the user interface concerns to whatever point you choose
to implement it.

> For example, you want to write a web application. You start a Flexibase
> application and select a module that provides an auth service, and a module
> that provides session storage. Then you write a module that provides the
> webapp service to Flexibase and now you have a UI. Later you realise that
> you've put business logic in your web controller, but because you wrote that
> code in terms of Flexibase you can trivially move it to a new service. Now
> your cron job can construct the same Flexibase application without the UI and
> just run the code with sensible data.

A Flexibase application is driven by putting Minds in a Hive.

## Terms

### Application

The application is the single executable that will comprise multiple Minds.
Each application will define its own Minds by configuring the Hive; an
application is therefore identified by this set of Minds.

Although each Mind can itself be configured, this configuration is not
considered pertinent to the identity of the application, but rather is drawn
from the environment in which the application runs.

### Component

Generic term for a module added to the system. A component contains a Mind and
some Hats, and may provide one or more services. The term also encompasses all
the other code in the component that is not relevant to the Hive or its
functionality.

### Hat

A Mind installed into the Hive wears one or many Hats. Each Hat is a unit of
behaviour provided by that Mind. Some Hats may be active; these implement
services. Passive Hats are used by Services to identify Minds that can provide
behaviour or information.

The term is named after the colloquial expression, "To wear one's [occupation]
hat". referring to the notion that an individual may be able to function in many
capacities but is at any one time only "wearing the hat" of one of them.

### Hive

The core object, properly known as the Hive, is the central system of Flexibase.
The Hive is constructed by the application developer defining a configuration
for it, and the configuration defines the Minds used and the uses to which they
are put.

### Mind

A Mind is any object that can be installed in a Hive. The Mind can define any
form of configuration necessary for its functionality and may require certain
Services to exist.

### Service

A service is a semantic name for a class of behaviour, whose usage is defined by
consensus. Some services are defined by Flexibase itself but any service can be
added to the Hive.

Conflicting services are unlikely: a second service would be designed to
implement the same interface as the first one in a different way. The interface
to services therefore needs to be strongly considered up front.

A Service is a Mind that implements the service; at runtime, the term refers to
the Mind configured to provide the service. Multiple Minds may implement the
same service but only one may provide it at runtime.

## Hive

Flexibase defines a core object whose implementation details are left vague to
accommodate the differences in languages.

We call the core object the Hive because it sounds cool. The concept of a hive
mind also helps us conceptualise the structure of the application.

### Availability

The Hive is essentialy defined as a singleton object; but this is in the sense
that each application needs only one. A developer may build a new one if they
choose to, but only in a way that does not infringe on the global one.

The Hive should be modified and controlled by means of a globally-available
interface. It is preferred that the global object is immutable; this is
especially important after initialisation.

The Hive and its Minds should work together to ensure the knowledge of their
structure is encapsulated within them and their communications. That is to say,
if someone constructs a new Hive for whatever reason, nothing done via that Hive
should ever interface with the global one.

### Construction

If the language permits, the Hive should be constructable by means of a
configuration object. This object will specify the Minds to use, define their
constructor parameters, and tell the Hive which Mind provides each service.

It is preferable to allow the Hive to construct the Minds using the data passed
in, than to pass in pre-constructed Minds. This ensures that we cannot have a
Hive in an unusable state. It also gives us an opportunity to initialise Minds
with the Hive they are installed in.

A compiled language may present a challenge constructing objects at
configuration time, since it may not be possible to load classes at runtime. The
precise mechanism by which a Hive is constructed is left up to the necessities
of the implementation, but it is a requirement that the Hive may not exist for
use by the application in an inconsistent state.

Irrespective of construction strategy, the configuration object must also
specify *which* Mind is used to provide each service. This applies even if only
one Mind is able to provide the service; we don't want to accidentally provide a
service we didn't intend to.

### Check

Check is the primary hook for ensuring Hive consistency. Every Mind may declare
dependencies on services or, rarely, other Minds. At check time, the
configuration is compared with the dependencies, and anything not provided
causes an error.

### Initialisation

The Hive is not viable until it has been initialised. This step iterates through
the constructed Minds and gives them the opportunity to perform further actions
now that they are instantiated and part of a Hive.

This is the first point at which the passive behaviour of Hats is valuable. The
initialisation step of certain Minds may involve asking the Hive for any other
Mind that implements a specific interface: in our world, this means wearing a
particular Hat.

Once every Mind has successfully run its initialisation then the Hive is
considered viable.

### Specification

> The Hive MUST be available at any point in the code.

> Any function called on a Mind by the Hive MUST receive the Hive.

> The Hive MUST provide itself to any function it calls on a Mind.

> The Mind MUST use the provided Hive and never call out to the global one.

> The Hive MUST be constructible.

> The Hive MUST be configurable. The Hive SHOULD be able to construct Minds
> based on this configuration. The Hive MAY accept pre-constructed Minds but is
> encouraged not to.

> The Hive MUST NOT provide a service unless it is explicitly configured to do
> so.

> The Hive MUST NOT be considered viable before declared constraints are
> satisfied.

> The Hive MUST cause a fatal error if declared constraints are not satisfied.

> The Hive MUST cause a fatal error if a service is requested that cannot be
> provided.

> The Hive MUST initalise each Mind before it is considered viable. This
> initialisation stage MUST pass the Hive to the initialisation functions. The
> Hive MUST have constructed all configured Minds and registered all configured
> services.

#### `new(Config config) -> Hive`

Constructs a new Hive from the given Config; the Config type is not specified
in this document to allow for interpretation.

#### `service(Str name) -> Hat`

Returns the Hat that provides the given service.

#### `hats(Str name) -> List[Hat]`

Returns a list of all Hats in the Hive by the given name.

#### `mind(Str name) -> Mind`

Returns the Mind called this. Calling this couples your Mind to another Mind and
is strongly discouraged.

## Minds

Minds are core objects in the component architecture. They provide an interface
into the behaviour your component provides, and exposes services and further
interfaces with which other Minds can access their functionality.

Each Mind is identified by a name. There is no restriction on the naming scheme
but it is recommended that it follows the identifier convention of the host
language. This name is intended to be set at the point the Mind is added to the
Hive, so that if for some reason two Minds have the same name, one can be
overridden.

The Mind must wear at least one Hat. Hats are either passive or they implement a
service. A Hat that implements a service will be named the same as the service,
in context of the Mind. More on that later.

Minds can declare reliance on services. This is normally done to ensure other
Minds are initialised first. It also allows for an early sanity check if you
know that your Mind cannot function without another service being available.
For example, you might declare a dependency on a database connection, or you
might be writing a component whose sole purpose is to provide an extension to
another one.

When a Mind is installed into a Hive its dependencies are checked. Following
this, the Mind is initialised with the completed Hive. At this stage, it is
guaranteed that all other Minds are available. It is also guaranteed that
dependencies are initialised.

Minds cannot leave their Hive context for behaviour! In some cases, the
application is using a singleton or global strategy for its Hive, but it is also
perfectly acceptable that the application is using a Hive stored in a local
scope. It is also perfectly acceptable for the system to arbitrarily create
another Hive at any point for whatever reason.

This means that you can only ever be sure of the existence of the Hive you are
installed in, or of a Hive you construct yourself for a specific purpose.
Remember that the Hive you are installed in has already been sanity-checked
against your dependency specification, so it's the only Hive you can be sure of.

### Specification

> A Mind MUST wear at least one Hat.

> A Mind MUST return an instantiated Hat when provided with its name.

> The name of a Mind MUST be configurable, and MUST have a default value.

> A Mind MAY provide one or more services.

> A Mind MUST throw an exception if initialisation fails for any reason.

> A Mind MUST be able to determine the Hive in which it is installed.

> The Mind MUST NOT use a Hive that it does not own, or the Hive in which it is
> installed.

#### `Str name = 'defaultname'`

Holds the name of the Mind. Must be defined.

#### `hat(Str name) -> Hat`

Returns the Hat by this name. Service name is the same as Hat name so there is
no equivalent `service` defined.

#### `hats() -> List[Str]`

Returns a list of names of Hats that this Mind wears.

#### `services() -> List[Str]`

Returns a list of service names available on this Mind.

#### `initialise(Hive)`

Runs any necessary initialisation. Default behaviour is no-op. No return value
is expected; a failure to initialise should be fatal.

#### `!hive() -> Hive`

Private method. Return the Hive in which this Mind is installed. Minds should
*never* use any other Hive than this; except that they may construct a *new*
Hive for private use and call methods on that.

#### `!other-mind(Str name) --> Mind`

Private method. Returns the other mind from *the same Hive* by the given name.
Additionally, this should cause a fatal error if a Mind is requested that this
Mind does not declare a dependency on. This could easily be written just once in
a base class for Minds.

As with `mind` on the Hive (which this should use), use of this method is
discouraged, because it couples your Mind to another Mind instead of an abstract
service. Prefer to use `self.hive.service(Str)` instead.

## Hats

Hats are how everything ultimately communicates, so obviously they're down here
at the end of the document instead of somewhere near the beginning.

A Hat provides a unit of behaviour and comes in two types: active and passive.
There is essentially no difference between a passive and an active Hat, except
how it is used.

### Defining Hat types

A Hat's type is determined by how it is used, which really means how it is
intended to be used. An important consideration is that a Hat is only useful if
something is expecting it to exist.

#### Services

The first type of Hat implements a service. When a service is requested from the
Hive, the Hive looks up the configured Mind to provide that service. It then
requests the corresponding Hat from that Mind. Only one of these Hats will be in
active duty at any one time.

There is a problem in defining an interface to a service, which is that a
service is something that can be replaced by an alternative implementation. That
means that any component that implements a service should be implementing an
interface that is defined somewhere more generic than that component; otherwise,
the first component gains the monopoly on the interface and interoperability is
sacrificed.

In practice, this is fine; with sufficient care applied to the design of an
interface, anything expecting that interface will continue to work if the
implementation is replaced.

Some services will be defined by the core of Flexibase, and these will be the
sorts of things considered common in many applications. (Not all services need
be provided in any given application!)

#### Passive Hats

The second type of Hat provides passive behaviour. For this type, *all* Hats of
the same name will be discovered in the Hive. Often, this type of Hat will
provide an extension to another service. Thus, the service, when it performs its
duty, will want to find all such extensions.

The interface to this Hat type is defined by the service to which it is
relevant. As with the service definitions, the interface to its passive
behaviour should be defined independently of the implementation that consumes
them. This way, a replacement implementation can interface with the other Minds
already in the Hive, already wearing those Hats.

### Specification

> A Hat MUST implement an interface. This interface MUST exist somewhere, but
> the location of this interface is specific to the type.

> The Hive MUST return a Hat when asked for a service. The caller MUST NOT use
> any methods not defined by the interface.

#### `Mind mind`

A private handle on the Mind wearing this hat.

## Semantic Types

Flexibase defines a system by which components can register their internal types
in an externally semantic way. This means that the internal details of a
particular implementation of a service (the stuff in a component that is not a
Hat or a Mind) can be hidden while still allowing other services to interact
with the objects inside it.

The semantic nature of these semantic types refers to the principle that any
service can deal with any object by its semantic type name, knowing that,
irrespective of the implementation detail, it will always deal with "the thing
that it thought it was going to be". Which is to say, if you ever asked for,
e.g., an object representing a user, you will always get an object representing
a user, irrespective of which Mind it is that responds to the request.

The second advantage of this semantic typing system is that Minds can expose
behaviour associated with types, without having to know whether anything is
actually providing that type. As with behaviour, the specification for a type
exists independently of the components that implement them, meaning any Mind can
wear a Hat that passively makes use of any semantic type names from any
specification.

### Example

Alice is writing a normal web app, with password authentication. She wants each
user to be able to edit their own profile, and she wants administrators to be
able to associate various other data with the users as well.

She writes a Mind that wears the `webapp` Hat, allowing the generic `webrunner`
component to find her application and expose it to her httpd.

She installs the `auth` service by selecting the Password Auth component, and
hooks it up to her login page by converting the OpenAPI schema into an HTML
form. She sends the posted data back to the service and handles the response
accordingly.

She also installs the `objectparams` service, which initially has no effect. To
enable it, she puts the `objectparams::extender` Hat on her Mind.

> The `::` in this name is a convention brought in from various languages. While
> not strictly necessary to adhere to the naming convention, it is preferred.

This new Hat is used to expose a list of *semantic type names*, and associate
them with OpenAPI schemata. The `objectparams` service will discover this Hat
and register these schemata against the types.

Finally, she installs the User administration UI component, which is not a Hive
component, but a component for her web app framework that understands how to use
the Hive. This component is designed to administrate any object that calls
itself `auth::user`. When the User page is displayed, the component asks the
`objectparams` service for all parameters for the `auth::user` semantic type;
and this responds with the OpenAPI schema that Alice programmed onto her
`objectparams::extender` Hat. It also responds with any other schemata that
other Minds might have added to the `auth::user` type to support their own
behaviour.

The component converts these schemata into HTML form components, adding their
fields to the form it was already going to draw for the core component. When the
form is saved, the posted data are despached back to the components that
initially defined them.

### Specification

> Flexibase implementations MUST support a semantic type registry system.

> The Hive SHOULD use a `type` method to retrieve information about a type by
> name. An alternative method name MAY be used, with justification.

> The Hive MUST check all Hats for their exposed types.

> Service specifications MAY define one or more semantic types.

> Service specifications MUST define on each semantic type at least sufficient
> attributes to comprise a unique key.

> Service specifications SHOULD use OpenAPI schemata in the documentation to
> define the types, even if it is just an example schema. They MAY use plain
> English to define the attributes and leave the specific schema up to the
> implementation.

> Type names MUST have at least one level of namespace, identifying the scope to
> which the type belongs.

> Namespaces SHOULD be relevant to the service name where possible.

> Implementations of a service MUST expose ALL of the types defined by the
> specification.

> Implementations MUST NOT expose any types defined by a different service.
> (This is avoided by sensible use of namespaces.)

> Implementations MUST use the OpenAPI schema format to define the fields.

> Implementations MAY expose types not documented by the service specification.

> Implementations MUST use, for these extra types, a namespace relevant to the
> specific implementation.
