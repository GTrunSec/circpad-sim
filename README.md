# circpad-sim
A minimal simulator for padding machines in Tor's circuit padding framework.

## Setup
- `git clone https://gitweb.torproject.org/tor.git/`
- `cp add-test_circuitpadding_sim.patch tor/`
- `git apply add-test_circuitpadding_sim.patch`
- `cp test_circuitpadding_sim.c tor/src/test/`

## Sketchpad during development
Advice from Mike: https://trac.torproject.org/projects/tor/ticket/31788 

TODO:
+ compile tor, run their unit-tests: `./src/test/test circuitpadding/..`
+ tor/doc/HACKING/WritingTests.md
+ add a new test-file for the sim logic, minimal changes to rest of tor,
  dedicated function for adding machines
- write a simple test that adds and runs a machine at a client and middle relay

### important design considerations
- the intended workflow is "modify machine -> sim many (order 1k-10k+) traces ->
  evaluate -> repeat"
  - embarrassingly parallel, but using many cores in C is not nice -> we run the
    sim on one trace, let user (python script) deal with calling many times
  - the workflow will cause us to recompile the test often -> we can hardcode
    parameters to the test if we want
- timers for sending padding cells are unreliable, 0-10 ms extra delay
  (https://trac.torproject.org/projects/tor/ticket/31653) -> we ignore detailed
  time for now, only want to get cell ordering right 

### How to format our input and output
From the framework, transitions are causes by padding and non-padding
sent/received, infinity bin, bins empty, and length count used up. All of these
events can be generated by just getting as input non-padding cells and their
respective timestamps at the client (this is a must). A comprehensive sim
framework should have input also from the relay (and guard according to Mike,
see https://trac.torproject.org/projects/tor/ticket/31788). This is because we
also need to accurately simulate the events at the relay-side padding machine,
and correctly simulate the delay between relay-client. We choose to be lazy
here, since our time is short and timers are unreliable as-is (see design
considerations above). 

A circuit looks like this:

client - guard - middle - exit - destination

Our goal is to, from a trace of events with timestamps at the client, infer
events at the middle relay. We do this by estimating the RTT from the timestamps
of cells sent and then received, then adding some significant randomness to this
value to encourage machines to be conservative in their timers. About 1/4 of the
RRT is the time it takes for a (non-)padding cell to traverse to/from the client
to/from the relay.

### input collection
in torrc, set ``Log [circ]info notice stdout''.- TODO, write glue script. Make
separate patch for default logging to tor log in TB for collection. Collect one
short trace here to use for sim development, don't spend more time on collection
before sim half-works.

### premature optimization
- now we fork then run the setup. If we want to use the c binary to fork bomb
  ourselves, then this is not so good. Probably shouldn't have as input the
  trace directly, but rather some other cli arg? Seems not to fit the framework
  right now, maybe turn into a main? Ask for help? Ideally we want: 
  
  ./sim <input> <output> <optional params>