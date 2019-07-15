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

### Platform vs. Runtime Layer

The OpenCL API is segmented into two parts: the platform and the runtime layers.

The platform layer is responsible for loading the available implementations (runtimes) at run-time and setting up all the underlying machinery to present them to the programmer in a pre-processed, easy to use manner.

The runtime layer is the bulk of the API. This part defines what can be done with OpenCL and how.

## Using the API

Correct usage of the C API is extremely verbose compared to when they are wrapped and used in other languages. This verbosity however is neccessary to allow runtimes to operate optimally. OpenCL is at a much lower level of abstraction than graphics APIs such as OpenGL or DirectX 11 and under for instance. Runtimes of these APIs have to make a lot of implicit assumptions which may not hold or they have to make a lot of expensive run-time checks which impacts performance.

