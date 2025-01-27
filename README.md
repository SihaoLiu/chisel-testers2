# ChiselTest
*formerly known as testers2*

This is **alpha software** that is currently under development, and interfaces may be subject to change (see [stability](#stability) for details).
However, it is very much in a usable state, so if you're fine living on the bleeding edge, give it a try. 

## Overview
ChiselTest is a test harness for [Chisel](https://github.com/chipsalliance/chisel3)-based RTL designs, currently supporting directed testing (all test stimulus manually specified - no constrained random and coverage-driven flows).
ChiselTest emphasizes tests that are lightweight (minimizes boilerplate code), easy to read and write (understandability), and compose (for better test code reuse).

The core primitives are similar to nonsynthesizable Verilog: input pin assignment (`poke`), pin value assertion (`expect`), and time advance (`step`). Threading concurrency is also supported with the use of `fork` and `join`, and concurrent accesses to wires are checked to prevent race conditions.

### Migrating from chisel-testers / iotesters
The core abstractions (`poke`, `expect`, `step`) are similar to [chisel-testers](https://github.com/freechipsproject/chisel-testers), but the syntax is inverted: instead of doing `tester.poke(wire, value)` with a Scala number value, in ChiselTest you would write `wire.poke(value)` with a Chisel literal value.
Furthermore, as no reference to the tester context is needed, test helper functions can be defined outside a test class and written as libraries.

Currently, this should support all the functionality that was in chisel-testers, and provides additional features.
This project is meant to supersede chisel-testers, and eventually may become a default part of chisel3.

Test cases written in chisel-testers cannot be directly used in ChiselTest, as the syntax is significantly different.


## Getting Started

### Installation 
To use chisel-testers as a managed dependency, add this in your build.sbt:
```scala
libraryDependencies += "edu.berkeley.cs" %% "chiseltest" % "0.2.1"
```

Note, `chiseltest` snapshots generally track chisel3 snapshots, and requires the use of a chisel3 snapshot.
We may introduce versioned snapshots and releases in the future, tied to a particular release of chisel3.

You can also build ChiselTest locally with `publishLocal`.

### Writing a Test
ChiselTest integrates with the [ScalaTest](http://scalatest.org) framework, which provides a framework for detection and execution of unit tests.

Assuming a typical Chisel project with `MyModule` defined in `src/main/scala/MyModule.scala`:

```scala
class MyModule extend Module {
    val io = IO(new Bundle {
        val in = Input(UInt(16.W))
        val out = Output(UInt(16.W))
    })

    io.out := RegNext(io.in)
}
```

Create a new file in `src/test/scala/`, for example, `BasicTest.scala`.


In this file:
1.  Add the necessary imports:
    ```scala
    import chisel3._
    import chiseltest._
    import org.scalatest.flatspec.AnyFlatSpec
    ```
2.  Create a test class:
    ```scala
    class BasicTest extends AnyFlatSpec with ChiselScalatestTester {
      behavior of "MyModule"
      // test class body here
    }
    ```
    - `AnyFlatSpec` is the [default and recommended ScalaTest style for unit testing](http://www.scalatest.org/user_guide/selecting_a_style).
    - `ChiselScalatestTester` provides testdriver functionality and integration (like signal value assertions) within the context of a ScalaTest environment.
    - For those interested in additional ScalaTest assertion expressibility, `Matchers` provides additional [assertion syntax options](http://www.scalatest.org/user_guide/using_matchers). `Matchers` is optional as it's mainly for Scala-land assertions and does not inter-operate with circuit operations.
3.  In the test class, define a test case:
    ```scala
    it should "do something" in {
      // test case body here
    }
    ```
    There can be multiple test cases per test class, and we recommend one test class per Module being tested, and one test case per individual test. 
4.  In the test case, define the module being tested:
    ```scala
    test(new MyModule) { c =>
      // test body here
    }
    ```
    `test` automatically runs the default simulator (which is [treadle](https://github.com/freechipsproject/treadle)), and runs the test stimulus in the block.
    The argument to the test stimulus block (`c` in this case) is a handle to the module under test.
5.  In the test body, use `poke`, `step`, and `expect` operations to write the test:
    ```scala
    c.io.in.poke(0.U)
    c.clock.step()
    c.io.out.expect(0.U)
    c.io.in.poke(42.U)
    c.clock.step()
    c.io.out.expect(42.U)
    println("Last output value :" + c.io.out.peek().litValue)
    ```
6.  With your test case complete, you can run all the test cases in your project by invoking ScalaTest.
    If you're using [sbt](http://scala-sbt.org), you can either run `sbt test` from the command line, or `test` from the sbt console.
    `testOnly` can also be used to run specific tests.


## Usage References
See the test cases for examples:
- [BasicTest](src/test/scala/chiseltest/tests/BasicTest.scala) shows basic `peek`, `poke`, and `step` functionality
- [QueueTest](src/test/scala/chiseltest/tests/QueueTest.scala) shows example uses of the DecoupledDriver library, providing functions like `enqueueNow`, `expectDequeueNow`, their sequence variants, `expectPeek`, and `expectInvalid`.
  Also, check out the [DecoupledDriver](src/main/scala/chiseltest/DecoupledDriver.scala) implementation, and note that it is not a special case, but code that any user can write. 
- [BundleLiteralsSpec](src/test/scala/chiseltest/tests/BundleLiteralsSpec.scala) shows examples of using bundle literals to poke and expect bundle wires.
  - Note: Bundle literals are still an experimental chisel3 feature and need to be explicitly imported:
    ```scala
    import chisel3.experimental.BundleLiterals._
    ```
- [AlutTest](src/test/scala/chiseltest/tests/AluTest.scala) shows an example of re-using the same test for different data
- [ShiftRegisterTest](src/test/scala/chiseltest/tests/ShiftRegisterTest.scala) shows an example of using fork/join to define a test helper function, where multiple invocations of it are pipelined using `fork`.
- [VerilatorBasicTests](src/test/scala/chiseltest/experimental/tests/VerilatorBasicTests.scala) shows an example using Verilator as the simulator.
  - Note: the simulator is selected by passing an annotation into the `test` function, which requires experimental imports:
    ```scala
    import chiseltest._
    ```
    ```scala
    test(new MyModule).withAnnotations(Seq(VerilatorBackendAnnotation)) { c =>
      // test body here
    }
    ``` 

## New Constructs
- `fork` to spawn threads, and `join` to block (wait) on a thread.
  Pokes and peeks/expects to wires from threads are checked during runtime to ensure no collisions or unexpected behavior.
  - `fork`ed threads provide a concurrency abstraction for writing testbenches only, without real parallelism.
    The test infrastructure schedules threads one at a time, with threads running once per simulation cycle.
  - Thread order is deterministic, and attempts to follow lexical order (as it would appear from the code text): `fork`ed (child) threads run immediately, then return to the spawning (parent) thread.
    On future cycles, child threads run before their parent, in the order they were spawned.
  - Only cross-thread operations that round-trip through the simulator (eg, peek-after-poke) are checked.
    You can do cross-thread operations in Scala (eg, using shared variables) that aren't checked, but it is up to you to make sure they are correct and intuitive.
    This is not recommended.
    In the future, we may provide checked mechanisms for communicating between test threads.
- Regions can be associated with a thread, with `fork.withRegion(...)`, which act as a synchronization barrier within simulator time steps.
  This can be used to create monitors that run after other main testdriver threads have been run, and can read wires those threads have poked.
- `timescope` allows pokes to be scoped - that is, pokes inside the timescope block "disappear" and the wire reverts to its previous value at the end of the block.
  This fits well with the pattern of assigning a default pull-up/down to a wire, and temporarily overriding that value, for example a Decoupled `valid` signal defaulting low but driven high during an enqueue transaction.
  See [TimescopeTest](src/test/scala/chiseltest/tests/TimescopeTest.scala) for examples.


## Quick References
To dump VCDs (into the test_run_dir subfolder) using sbt:
```
testOnly chiseltest.tests.BasicTest -- -DwriteVcd=1
```

## Verilator Backend

We currently support the following versions of the [verilator](https://www.veripool.org/wiki/verilator) simulator:
- `v4.028`: [Ubuntu 20.04](https://packages.ubuntu.com/focal/verilator), [Fedora 32](https://src.fedoraproject.org/rpms/verilator)
- `v4.032`: [Fedora 33](https://src.fedoraproject.org/rpms/verilator)
- `v4.034`: [Chipyard](https://chipyard.readthedocs.io/en/latest/Chipyard-Basics/Initial-Repo-Setup.html#requirements)
- `v4.038`: [Ubuntu 20.10](https://packages.ubuntu.com/groovy/verilator)
- `v4.108`: [Fedora 34](https://src.fedoraproject.org/rpms/verilator)
- `v4.202`

## Stability
These APIs may be considered stable and are unlikely to change significantly:
- Test invocation `test`
- Basic operations: `poke`, `peek`, `expect`, `step`, including on Bundles
- Basic concurrency operations: `fork`, `join`
- Decoupled library: `enqueueNow`, `expectDequeueNow`, their sequence variants, `expectPeek`, `expectInvalid` - though the names may be refactored
- `timescope` - though the name may be refactored

These are subject to change:
- Multiclock behavior, which is currently not well defined, including `poke` on clocks and `step` when there are multiple clocks.


## Roadmap
These features are on our roadmap, but are not implemented yet.
No timeframe is currently available, but feel free to let us know if some feature is critical to your use case in the relevant issue thread, and we may adjust our development priorities accordingly.
- [#14](https://github.com/ucb-bar/chisel-testers2/issues/14) support for multi-clock designs, and in particular, supporting clock poke to clock step inter-thread dependencies.
- [#28](https://github.com/ucb-bar/chisel-testers2/issues/28) faster Verilator / VCS testing using mechanisms that avoid interprocess communication, like JNI
- [#58](https://github.com/ucb-bar/chisel-testers2/issues/58) faster threading (note, unclear if there are good solutions here, especially ones that are fully API compatible - Scala generally lacks good coroutine support)
- [#60](https://github.com/ucb-bar/chisel-testers2/issues/60), [#34](https://github.com/ucb-bar/chisel-testers2/issues/34), [#2](https://github.com/ucb-bar/chisel-testers2/issues/2) support for testing non-Chisel modules, such as post-syn Verilog, or generally a separation of DUT interface and implementation
