# JULEP Logging

- **Title:** A unified logging interface
- **Author:** Chris Foster <chris42f@gmail.com>
- **Created:** February 2017
- **Status:** work in progress

## Abstract

*Logging* is a tool for understanding program execution by recording the order and
timing of a sequence of events.  A *logging library* provides tools to define
these events in the source code and capture the event stream when the program runs.
The information captured from each event makes its way through the system as a
*log record*. The ideal logging library should give developers and users insight
into the running of their software by provide tools to filter, save and
visualize these records.

Julia has included simple logging in `Base` since version 0.1, but the tools to
generate and capture events are still immature as of version 0.6. For example,
log messages are unstructured, there's no systematic capture of log metadata, no
debug logging, inflexible dispatch and filtering, and the role of the code at
the log site isn't completely clear.  Because of this, Julia 0.6 packages use
any of several incompatible logging libraries, and there's no systematic way to
generate and capture log messages.

This julep aims to improve the situation by proposing:

* A simple, unified interface to generate log events in `Base`
* Conventions for the structure and semantics of the resulting log records
* A minimum of dispatch machinery to capture, route and filter log records
* A default backend for displaying, filtering and interacting with the log stream.

A non-goal is to create a complete set of logging backends - these can be
supplied by packages.

## The design problem

There's two broad classes of users for a logging library - library authors and
application authors - each with rather different needs.

### The library author

Ideally logging should be a high value tool for library development, making
library authors lives easier, and giving users insight.

For the library author, the logging tools should make log events *easy to generate*:

* Logging should require a minimum of syntax - ideally just a logger verb and
  the message object in many cases.  Context information for log messages (file
  name, line number, module, stack trace, etc.) should be automatically gathered
  without a syntax burden.
* Log generation should be free from prescriptive log message formatting. Simple
  string interpolation, `@sprintf` and `fmt()`, etc should all be fine.  When
  log messages aren't strings, a sensible conversion should be applied by
  default.
* Flexible user definable structure for log records should make it easy to
  record snapshots of program state in the form of variable names and values.
  This would generalize `@show` using log records as a transport mechanism.

The default configuration for log message reporting should involve *zero
setup* and should produce *readable output*:

* No mention of log dispatch should be necessary at the message creation site.
* The default console log handler should integrate somehow with the display
  system, to show log records in a way which is highly readable.
* Basic filtering of log messages should be easy to configure.

The default configuration for log message reporting will generally define what
library authors see during development, so will end up defining the conventions
authors use when including logging in their library.  To this extent, it's
important to do a good job displaying metadata!

### The application author

Application authors bring together many disparate libraries into a larger
system; they need consistency and flexibility in collecting log records.

Log events are generally tagged with useful context information which is
available both lexically (eg, module, file name, line number) and dynamically
(eg, time, stack trace, thread id).  Log records should have *consistent,
flexible metadata* which represents and preserve this structured information in
a way that can be collected systematically.

* Each logging location should have a unique identifier, `id`, passed as part of
  the log record metadata.  This greatly simplifies tasks such limiting the rate
  of logging for a given line of code.
* Users should be able to add structured information to log records, to be
  preserved along with data extracted from the logging context. For example, a
  list of `key=value` pairs offers a decent combination of simplicity and power.
* Clear guidelines should be given about the meaning and appropriate use of
  standard log levels so libraries can be consistent.

Log *collection* should be unified:

* For all libraries using the standard logging API, it should be simple to
  intercept, and dispatch logs in a unified way which is under the control of
  the application author.  For example, to write json log records across the
  network to a log server.
* It should be possible to naturally control log dispatch from concurrent tasks.
  For example, if the application uses a library to handle simultaneous HTTP
  connections for both an important task and a noncritical background job, we
  may wish to handle the messages generated by these two `Task`s differently.

The design should allow for an *efficient implementation*, to encourage
the availability of logging in production systems; logs you don't see should be
almost free, and logs you do see should be cheap to produce. The runtime cost
comes in a few flavours:

* Cost in the logging frontend, to determine whether to filter a message.
* Cost in the logging frontend, in collecting context information.
* Cost in user code, to construct quantities which will only be used in a
  log message.
* Cost in the logging backend, in filtering and displaying messages.


## Proposed design

A prototype implementation is available at https://github.com/c42f/MicroLogging.jl

### Quickstart Example

#### Frontend
```julia
# using Base.Log

# Logging macros
@debug "A message for debugging (filtered out by default)"
@info "Information about normal program operation"
@warn "A potentially problem was detected"
@error "Something definitely went wrong, but we recovered enough to continue"
@logmsg Logging.Info "Explicitly defined info log level"

# Free form message formatting
x = 10.50
@info "$x"
@info @sprintf("%.3f", x)
@info begin
    A = ones(4,4)
    "sum(A) = $(sum(A))"
end

# Progress reporting
for i=1:10
    @info "Some algorithm" progress=i/10
end

# User defined key value pairs
foo_val = 10.0
@info "test" foo=foo_val bar=42
```

#### Backend

### What is a log record?

Logging statements are used to understand algorithm flow - the order and timing
in which logging events happen - and the program state at each event.  Each
logging event is preserved in a *log record*.  The information in a record
needs to be gathered efficiently, but should be rich enough to give insight into
program execution.

A log record includes information explicitly given at the call site, and any
relevant metadata which can be harvested from the lexical and dynamic
environment.  Most logging libraries allow for two key pieces of information
to be supplied explicitly:

* The *log message* - a user-defined string containing key pieces of program
  state, chosen by the developer.
* The *log level* - a category for the message, usually ordered from verbose
  to severe.  The log level is generally used as an initial filter to remove
  verbose messages.

Some logging libraries (for example
[glib](https://developer.gnome.org/glib/stable/glib-Message-Logging.html)
structured logging) allow users to supply extra log record information in the
form of key value pairs.  Others like
[log4j2](https://logging.apache.org/log4j/2.x/manual/messages.html) require extra information to be
explicitly wrapped in a log record type.  In julia, supporting key value pairs
in logging statements gives a good mixture of usability and flexibility:
Information can be communicated to the logging backend as simple keyword
function arguments, and the keywords provide syntactic hints for early filtering
in the logging macro frontend.

In addition to the explicitly provided information, some useful metadata can be
automatically extracted and stored with each log record.  Some of this is
extracted from the lexical environment or generated by the logging frontend
macro, including code location (module, file, line number) and a unique message
identifier.  The rest is dynamic state which can be generated on demand by the
backend, including system time, stack trace, current task id.

### The logging frontend

TODO

### Logging middle layer

TODO

### Early filtering

TODO

### Default backend

TODO

## Concrete use cases

### Base

In Base, there are three somewhat disparate mechanisms for controlling logging.
An improved logging interface should unify these in a way which is convenient
both in the code and for user control.

* The 0.6 logging system's `logging()` function with redirection based on module
  and function.
* The `DEBUG_LOADING` mechanism in loading.jl and `JULIA_DEBUG_LOADING`
  environment variable.
* The depwarn system, and `--depwarn` command line flag


## Inspiration

This Julep draws inspiration from many previous logging frameworks, and helpful
discussions with many people online and at JuliaCon 2017.

The Java logging framework [log4j2](https://logging.apache.org/log4j/2.x/) was a
great source of use cases, as it contains the lessons from at least twenty years
of large production systems.  While containing a fairly large amount of
complexity, the design is generally very well motivated in the documentation,
giving a rich set of use cases.  The julia logging libraries - Base in julia 0.6,
Logging.jl, MiniLogging.jl, LumberJack.jl, and particularly
[Memento.jl](https://github.com/invenia/Memento.jl) - provided helpful
context for the needs of the julia community.

Structured logging as available in
[glib](https://developer.gnome.org/glib/stable/glib-Message-Logging.html)
and [RFC5424](https://datatracker.ietf.org/doc/rfc5424/?include_text=1) (The
Syslog protocol) provide context for the usefulness of log records as key value
pairs.

For the most part, existing julia libraries seem to follow the design tradition
of the standard [python logging library](https://docs.python.org/3/library/logging.html),
which has a lineage further described in [PEP-282](https://www.python.org/dev/peps/pep-0282/).
The python logging system provided a starting point for this Julep, though the
design eventually diverged from the typical hierarchical setup.

TODO: Re-survey the following?
* a-cl-logger (Common lisp) - https://github.com/AccelerationNet/a-cl-logger
* Lager (Erlang) - https://github.com/erlang-lager/lager


