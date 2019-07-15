# Anatomy of a vanilla OpenCL application

OpenCL, like many other portable APIs is defined as a C API. This is because C with it's simple, stable and mature ABI has become the _lingua franca_ of portable languages. Many portable technologies are defined as a C library which are then wrapped into other languges through their C interop capabilities and dressed up with all the goodness the target language provides.

## The C API

OpenCL can natively be programmed in C using the API that is defined with each version of the standard. In order to get going, the bare minimum one's going to need is a set of C headers that define all the API functions available, and an OpenCL library to link to.

This OpenCL library that we link to can be one of two things:

- An actual library holding the implementations to the API functions
- A relay library which will delegate all invocations to yet another library

The prior is fairly straightforward and does not overcomplicate the situation. This scenario is usually employed by runtime developers (not end-users) because it's simple (easier to debug) but not flxible enough for real life scenarios. The latter in OpenCL-land is called an Installable Client Driver, aka. ICD.

### ICD

#### Why do we need one?

Take OpenGL for eg. It is a much older API which is backwards compatible to it's very first incarnations, not just API-wise, but also regarding it's ABI. Many things that were fine at the time are less sexy today. One such decision is that canonically, OpenGL implementations reside in a shared library with the name `OpenGL`. This makes it difficult for multiple OpenGL implementations to reside side-by-side, because one can only link an executable to a single library of a given name. Again, this was a natural choice in the 90's, but today having integrated and dedicated graphics from mixed vendors is a commonplace. This choice of ABI is the primary source of aggravation for switchable graphics on *nix OS flavors.

As OpenGL evolved, many extensions were added, many of which require new host-side functions. These functions were provided through functions pointers of well-defined signature, but with a `NULL` value at program launch. These extensions to the API were set by projects external to OpenGL itself, such as the lightweight OpenGL Extension Wrangler (GLEW) library, or more comprehensive and integrated solutions, such as the QOpenGL module of Qt.

#### What does it do?

OpenCL solves this issue by also naming it's library `OpenCL`, but integrating a GLEW-like layer into the API itself, aptly named the platform modell. This layer of the API will be responsible for loading implementations at runtime as well as storing and setting implementation function pointers into these libraries which are not exposed to the user.

OpenCL applications are blind to being linked to an ICD or an actual implementation, they (trivially) cannot distinguish between the two modes of operation. In reality, they really don't have two, typical usage of OpenCL is done through an ICD.

#### How does it work?

A typical ICD implementation queries some record for a list of libraries to load dynamically (usually via `dlopen()`) and probe for the pointer to the API functions inside. For a detailed description of how the ICD works, refer to the [ICD chapter](ICD.md).

### Platform vs. Runtime Layer

The OpenCL API is segmented into two parts: the platform and the runtime layers.

The platform layer is responsible for loading the available implementations (runtimes) at run-time and setting up all the underlying machinery to present them to the programmer in a pre-processed, easy to use manner.

The runtime layer is the bulk of the API. This part defines what can be done with OpenCL and how.

## Using the API

Correct usage of the C API is extremely verbose compared to when they are wrapped and used in other languages. This verbosity however is neccessary to allow runtimes to operate optimally. OpenCL is at a much lower level of abstraction than graphics APIs such as OpenGL or DirectX 11 and under for instance. Runtimes of these APIs have to make a lot of implicit assumptions which may not hold or they have to make a lot of expensive run-time checks which impacts performance.

The API follows a few concepts which naturally manifest in the C API (and are sometimes hidden by wrappers). It is instructive to be familiar with these concepts in general, even if you are not going to be using the C API when developing your applications. Knowing that under the hood, these calls are going to be made helps:

1. appreciate the wrapper
2. understand performance implications
3. debug faulty code

### Pervasive use of error codes

The return value of most API functions is an error code. The excpetion to this rule are the `clCreateXYZ` functions, which create a single new API object. These functions don't return the error code, but take a pointer to an error code as their last argument.

> It is highly advised to not discard error codes! They are of paramount importance to detect erronous API usage.

### clRetain & clRelease

API objects are: reference counted / follow reference semantics / should be treated as handles. Copying them around never duplicates the underlying resource, may it be a buffer, a command queue or an event (synchronisation primitive). _In the rare cases where it makes sense, there is a dedicated API function for deep-copying of objects._ It is the responsibility of the programmer to free/release API objects when they go out of scope, which signals the underlying runtime that the resource can be disposed of.

As a rule of thumb:

- Whenever the user retrieves new API entities, they are implicitly retained (their ref count will equal to 1).
- When the user creates a copy of an API object, **it should be retained**.
- When an object goes out of scope, **it should be released**.
- Do not try to be smart and omit these retains/releases, because sooner or later, it will blow up in your face. Retain/release is much cheaper than tracking down memory leaks.

_Most wrappers hide this part of the API, but they also allow for these low-level handles to be obtained. Doing so, one usually circumvents the wrappers capability to handle object lifetimes automatically. "With great power comes great responsibility." - Uncle Ben_

### Platform layer first, runtime layer next

A typical OpenCL application does the following things in roughly this order:

1. Query the ICD for the available implementations
2. Perform device selection based on some heuristic (or user input)
3. Create context(s) for the selected device(s)
4. Create command queue(s) for the selected device(s)
5. Create buffers that will take care of host-device and device-device data movement
6. Load and compile device code (kernels, see later)
7. Execute code on host and device as needed

_Note: Some parts of the API may be let go of as the application is running. The device compiler may potentially be a heavy-weight runtime object, and as such has a dedicated function which unloads it from memory. Querying for platforms however triggers the ICD to dynamically load available implementations into memory. It is not possible to let unload them, even if we know we won't be using some of them in the foreseeable future._

### Run-time kernel compilation