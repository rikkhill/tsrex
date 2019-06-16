# tsrex Design Document

In which we explain:
 - what tsrex is supposed to do
 - how it's supposed to work
 - how we're going to do it 
 
## What is tsrex supposed to do?
Search for patterns in sequential data by specifying strings similar to regexes.
By "sequential data", we mean things like log files, panel data, or time series.
By "patterns", we mean contiguous records in the series which conform to an ordering of interest to the user.
By "strings similar to regexes", we mean a string which encodes that pattern.

### An example
Imagine we have the following log file:

```
...
2019-06-14 17:29:59,019 INFO    FOO
2019-06-14 17:29:59,044 INFO    BAR
2019-06-14 17:29:59,102 INFO    FOO
2019-06-14 17:29:59,111 INFO    BAZ
2019-06-14 17:29:59,244 ERROR   ARGH!
2019-06-14 17:29:59,248 INFO    BAR
2019-06-14 17:30:46,788 INFO    SNOO
2019-06-14 17:30:46,811 INFO    BAZ
2019-06-14 17:30:46,908 ERROR   ARGH!
2019-06-14 17:30:46,913 INFO    FOO
...
```
Suppose you suspect the `BAR` events are some kind of precursor to the `ARGH!` error,
and want to inspect all the events that happen between `BAR` and the error. You *could* write a big dirty
regular expression to look through this log file, but you'd have to encode all the details of the log records
into the regex in order to do it.

What we want is to intelligently use the structure of the logfile to create a sequence of events with
properties, and then to specify a pattern, something along the lines of
`([@event_type="BAR"].+[@event_type="ARGH!"])`, and find all sequences of events that match that pattern.

This opens up other options. We could search a log file for some very specific sequences. Imagine a log of
user journeys through a website. It would be possible to interrogate these logs for very specific journeys
through the site. Alternatively, consider a record of a customer's interactions on a CMS or someone's spending
history. Any sequential tabular data suddenly becomes much easier to interrogate. From here on in, we will
refer to all such data as 'time series data'.

## How it's supposed to work
Going through a sequence of entities and matching them against a specified pattern is a solved problem.
We've even spoken about that solution already: regexes. The "vanilla" method for matching regexes is to
parse the regex pattern and use it to construct a finite state machine. The sequence is then fed through the
finite state machine to see if it reaches the end. If it does, it's a match. Contemporary regex
implementations are a little more complicated than this, as they have additional features such as lookaheads,
backreferences, etc.

The proposal here is to convert our time series data into a sequence of events, specify a pattern to
match to that sequence, compile that sequence to a finite state machine and then feed the sequence into
that machine to find relevant matches.

### Complications and additional features
There are obviously further complications when dealing with sequences of events instead of sequences of
characters. A string regex just has to check for equality of one character. In the case of time series
regexes, we have a container of properties to check. While this makes the process of matching an event
more complicated, it also opens up some new feature options. We could, for example, specify comparator
values for these properties. Imagine a time series with multiple fields of numeric data: we could specifiy
to only match with events where `@NumericFieldA<@NumericFieldB`.

A feature which may not be implementable using just a standard FSM is time-conditional matching. Suppose
we want to specify "match any event that occurred within *s* seconds of the last match". This would be an
extraordinarily useful feature, but requires the matching engine to keep a reference to the timestamp of the
last matched event.

## How we're going to do it

We're going to build a core library in C++ which does all the heavy lifting, and then we're going to make
Python bindings, so it's in a language people actually use for time series analysis. We should keep these
as modular as possible so we can extend support to other languages such as R in future.

### Components

#### Sequencer
A bunch of data structures for representing a sequence of events. Probably the most straightforward part
of the whole thing.

#### Parser
Similar to a standard regex parser. Needs to handle braces and infix operators, and return a sequence of
operands and operators that can be used 

Ancillary work on this is syntax specification. It is probably a good idea to repurpose familiar regex
syntax where possible, for two reasons: 1) people are already familiar with regex syntax, and 2) the
standard regex operators (`|*?+`, etc.) are the same operators required to construct FSMs.

#### FSM Generator
A component that takes the output of the parser and constructs a finite state machine.

#### Matching Engine
Handles the logic of searching through the sequence for the specified pattern. Most string regex tools offer
some equivalent of `match` (which returns true if the match exists in the sequence), `find`, (which returns
the first case of the pattern in the sequence), and `findall`, which returns all matches in the sequence.
We will want to honour how these operations work.

We may also want to offer something along the lines of `findsequential`. Users will want to specify
whether they do or do not want overlapping patterns.

#### Bindings
We will write some Python bindings to allow use of the tool in Python, as well as some convenience logic
to allow easy conversion of relevant datatypes such as Pandas dataframes.

#### CLI entrypoint
This is probably best written as an extension to the Python components in Click or a similar CLI
framework. It can offer basic support for matching CSV files, log files or JSON-formatted time series.
We will probably end up building something like this as a debugging tool while in development.