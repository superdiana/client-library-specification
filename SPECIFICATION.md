# Server Library Specification
The goal of this document is to make all Nexmo Server Libraries have a similar feel, while still adopting the standard practice and usage of each language. This is mostly a functional specification based on the user's interaction with the server library.

* This document is a work in progress and should be changed and updated as changes are made to the APIs or developer needs are discovered.
* Server Libraries attempt to make the API and the data it provides simple and easy to use. In some cases, this may involve implementing small amounts of extra functionality in the client.
* This specification is designed primarily for object-oriented languages that support inheritance. In other systems, this specification should be adapted to the conventions of the target language.

## General Principles
* Ideally, our server libraries should provide a simple, easy-to-use interface to the Nexmo APIs. In practice, the libraries should attempt to clean up some of the data required or returned by the older APIs. (ints returned as strings in JSON, for example, or more complex: parsed timestamps to native date objects). While the SDKs may provide a low-level interface to directly interact with the API, each SDK should provide an interface that abstracts out the fact the SDK is backed by an HTTP API.
* JSON data structures are not native to most programming languages, and so should not be expected or provided by the API's top-level methods. The data should be deserialized soon, serialized late, and held in objects which can provide richer functionality than simple dicts, lists, etc.
* Our libraries should be explicit. In some cases, it may be less work to have the library definition defined in a data file and dynamically loaded. In practice, this causes problems for IDEs and autocompletion.
* Parameters should be provided either directly to functions or as part of a 'request' object. They should not be provided as a single dictionary to the function when possible, as this sidesteps all of any IDEs auto-completion. Where groups of parameters go together it makes sense to put them in a value object, which can then be provided as a single parameter.
* Naming conflicts with language keywords or globals, (e.g: `to` is a keyword in Python) should be corrected using the language's accepted convention. (In Python it is to add an underscore suffix, e.g: `to_`). If you rename a value, document the change in this specification, and ensure it is renamed consistently across all endpoints.
* For the endpoints which return `420 Enhance your calm`, we will do additional sensible validation before sending the request to the server and additional validation on the response.
* For endpoints that return a `200 OK` but have an error in the body, we need to check for these on each response. This also includes partial OKs, like with the SMS API. These should be exposed as an error to the user.
* Although most of Nexmo's APIs are mostly RESTful, this is an implementation detail, and not something we should necessarily promote to the user, who should not care about the semantics of different HTTP verbs such as GET, POST or PUT.
* Errors should be reported using the language's error functionality. In languages like Java and Python, this would involve throwing an appropriate Exception object. In languages such as Go or Rust, this would involve returning an appropriate error from the function. This means any object returned from the API which represents an error should be used to populate an error type.
* We should keep third-party dependencies lightweight, but we should not shy away from bringing in third-party dependencies where they make sense. Where depending on third party libraries we should use libraries that are 'de-facto standard' wherever possible. No proprietary dependencies on the SDKs to minimize dependencies disappearing due to commercial interests.
  
# Callbacks
Support should be provided for parsing callback data and verifying the callback. Details to be hashed out.
Callback support may be provided in a separate module with extra dependencies to the core nexmo library.

# Behavior
## HTTP Transports
The libraries should all make use of each ecosystem's best-quality HTTP library, even if it means an added 3rd-party dependency. Each client object should maintain a pool of open HTTP connections, making subsequent calls less expensive.

### User Agents

Each library should set up a custom user agent for tracking. By default, the string must be in the following format and contain the library name, library version, language name, and language version:

* `LIBRARY-NAME/LIBRARY-VERSION LANGUAGE-NAME/LANGUAGE-VERSION`

The library must allow the user to append an application name and, if needed, an application version. If the user supplies this information, it will be appended to produce the following format:

* `LIBRARY-NAME/LIBRARY-VERSION LANGUAGE-NAME/LANGUAGE-VERSION APP-NAME[/APP-VERSION]`

## Logging
Each SDK should use the standardized logging system as determined by the language (a PSR-3 client like Monolog for PHP, Log4J for Java, etc). The SDKs should have DEBUG-level logging which can be turned on and off by the users. We should redact client secrets. 

## Filtering
Many endpoints allow the users to Search, or Filter, the result sets that are returned. This is a distinctly different operation than a Find operation, which is intended to return a single result. The SDKs should provide an interface for helping developers create and pass along the filters to make it easier to subset information. 

For example, at a base level a `KeyValueFilter` object should be supplied that accepts a set of key-values that can be turned into a filter. Each SDK should implement additional filters as they make sense to the request types. For example, a `CursorFilter` may take a request and a direction (next/previous) to generate a filter to grab the next cursor-based page.

Possible example filters are:

* `EmptyFilter` - A filter that returns all results
* `KeyValueFilter` - Takes a generic set of keys and values to search by

## Pagination
By default, pagination should be hidden from the user and exposed as a cursor or iterable for high-level functions. Users may, if they elect, drop down to a closer API level client to do manual paging if the endpoint supports it. For example, a library will expose a higher-level function to retrieve objects that will do proper hydration and dependency injection, while allowing the user to use a direct API object to get back raw values.

The higher level iterable/cursors should be smart enough to reduce the number of HTTP requests that they make with sane defaults. For example, a paging size of 1 or 100 can be suboptimal, where a paging size of 5 or 10 may be more suitable. This value may change based on the product/endpoint.

Cursor-based endpoints that cannot accept a page may accept the next or previous cursor page link as a substitute.

Examples:
* `$client->conversations()->searchConversations($filter)` will return an iterable that returns Conversation objects, and will handle paging or cursor navigation internally
* `$client->conversations()->getApiClient()->get($filter)`, with `$filter` being given a `page=3` option, will return page 3 of the given filter results
* `$client->conversations()->getApiClient()->get($filter)`, with `$filter` being given a `cursor=abcd` option,  will return the next page of the cursor of the filter results
  
## Testing
At a minimum, each server library should strive for 100% unit test coverage, which can be automated through Continuous Integration tools. While it is acknowledged that unit testing coverage alone is not indicative of whether a library is fully tested, having the ability to write tests as bugs are found is very important.

Unit tests should test both the success as well as failure routes in code. It is not enough to have a test that shows a method works, there must be corresponding tests to show how a method behaves under failure conditions as well. We should make sure that all the failure and success responses documented in the API spec are covered by tests. 

Testing suites should be implemented in the language's standard tooling, such as PHPUnit for PHP or the pytest module for Python. The Server Library maintainer can make the final decision, but the unit test suite must be runnable and reportable from the command line and Continuous Integration tools.

Additional testing may be placed on top of standard unit tests. Server Library maintainers may add additional testing like lint testing, style checking, and static analysis when they see fit.

# Structure
The top-level entity is a class/struct called NexmoClient. In languages where the library would be imported and used under the 'nexmo' namespace, it may be called Client. E.g `nexmo.Client`. 

## Further Namespacing
It is possible to put all the required functionality directly into the NexmoClient object, but this creates a large single class, with the potential for naming conflicts. For example, createConversation could exist in the voice and conversation APIs. To avoid this, in languages where this is considered relatively standard, the NexmoClient object should contain fields for each of Nexmo's APIs:

* Account
* Audit
* Application
* Conversation
* Dispatch
* Insight
* Pricing
* Redact
* Messages
* Messaging
* NumberInsight
* Numbers
* Redact
* Reports
* Verify
* Voice

These namespaces should be accessed directly through the client via an accessor. 
In most languages, these should be accessed directly through the field name, e.g.: `client.application`. In Java, it should be accessed via an accessor, e.g.: `client.getApplicationClient()`

# Naming Principles
Each ecosystem's conventions for naming and case override anything below, especially whether entities are named using snake_case, CamelCase or some other convention.

Also, consistency trumps everything else. If the library has been written following a different convention, then continue to follow that convention. If the library requires a major overhaul, consider moving to the conventions described below.

## Direct API Call Methods
When a method is just a wrapper for a specific API function, the method name should take the form of the operationID in the open API spec. In the case of legacy SDK naming, those should be eventually deprecated and moved to the new naming convention.

## Global Verb Definitions for Convenience Methods
One of the main goals of our Server SDKs is to provide not only access to our APIs but also convenient access to working with our APIs. Many times this means wrapping bare API calls in objects like Services or Facades to make it easier for the developer to do complex operations. When possible, the SDKs should adhere to specific naming conventions for the actions being taken. 

Each function should take the form of verbNoun, with suggestions for verbs from the list below. Nouns may be plural or singular depending on what makes sense but avoid having different functions with the same verb on plural and singular noun eg getMessage & getMessages (which semantically should be called searchMessages).

* **Correct:** `redact.redactTransaction(...)` - this follows the pattern {namespace}.{verb}{noun}
* **Incorrect:** `redact.transaction(...)` - this mistakes the namespace for a verb, and thus just has the name of the entity (`transaction`) as a function. Function names should almost always contain a verb because functions do things and the verb describes what they do.
* **Incorrect:** `sms.send(...)` - this mistakes the namespace for the entity. In the future, we may add extra entities to the sms namespace, and then this method will no longer make sense - send what?

### Verbs
* `create` - Create an object of this type
* `delete` - Delete an object by ID of this type
* `get` - Retrieve a specific object by ID
* `list` - Wrappers for searches that have no search criteria
* `search` - Return a list of these objects by the search criteria
* `send` - For operations that do something but do not create a resource, like sending an SMS
* `update` - Update an existing object with a new copy/data

# Errors
In most cases, assuming a system supporting inheritance, there should be a top-level NexmoException, extended by NexmoClientException, for reporting errors with the way the API is being used by the client and [NexmoServerException](https://tools.ietf.org/html/rfc7807) for server errors, [NexmoValidationException]( https://nexmo.github.io/api-standards/http-code/400/) or problems with the network or Nexmo APIs. These exceptions will be extended further for more specific cases where more data should be stored in the exception. (Given that each language has a convention for Exception vs Error, these should be adapted to NexmoError, NexmoClientError, as necessary).

## Exception Handling
The language's native exception/error reporting system should be used for all errors handled by the server library. Endpoints that return 200 or 420 responses on error will be parsed in the server library and converted to appropriate errors where possible.

Partially successful responses (not all parts sent successfully, for example) will be handled as errors, but the error should also provide access to information about exactly what failed and what succeeded. Details to be hashed out.

# Client Construction
## Authentication
Either a constructor or initializer should allow the creation of a NexmoClient object with appropriate credential parameters;

This will be one or more of:
* API KEY & SECRET
* API KEY, SIGNATURE SECRET, AND SIGNATURE HASH FUNCTION/OPTION
* API KEY, SECRET, & SIGNATURE SECRET
* APP ID and PRIVATE KEY
* APP ID and PRIVATE KEY PATH

If a method supports multiple forms of authentication, it should choose a preferred one in the following order:
* JWT
* Signature Secret
* Key / Secret

Different functions will only support a subset of these, therefore if a function is called that is not supported by the client's provided authentication data an error should be reported. The error message should inform the user of the full set of credentials that would be suitable.

## Endpoint Base URLs
A client object should allow the base URL to be overridden when it is created.

## Recommended Environment Variable Names
This is intended for framework-specific extensions where we magically bootstrap a Nexmo Client in a service provider etc.

* `NEXMO_API_KEY`
* `NEXMO_API_SECRET`
* `NEXMO_SIGNATURE_SECRET`
* `NEXMO_SIGNATURE_METHOD`
* `NEXMO_PRIVATE_KEY`
* `NEXMO_PRIVATE_KEY_PATH`
* `NEXMO_APPLICATION_ID`

## JWT Generation
The client should provide a function for generating a JWT to be used by custom-built requests. Add a wrapper for the JWT generation for the things we know about, but still provide the ability to add custom data. This allows future expandability for upcoming features.

## User Agent Reporting

To better understand the usage of needs of developers building on Nexmo, libraries:

* MUST identify requests as originating from the library.
* MUST report internal client library version in each request.
* MUST set a user-agent with the following format: `LIBRARY-NAME/LIBRARY-VERSION LANGUAGE-NAME/LANGUAGE-VERSION`
    Example: `nexmo-php/1.0.0 php/7.0.8`
* SHOULD report language version in each request, if not possible MUST report version as `-`
    Example: `nexmo-php/1.0.0 php/-`
* MUST allow an application name and version to be appended with the following format: `APP-NAME/APP-VERSION`
    Example: `nexmo-php/1.0.0 php/7.0.8 demo/2.0`


