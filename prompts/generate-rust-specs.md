Deeply analyze this codebase for the Avalanche blockchain and capture the code-agnostic essence of the Go code and generate specs-rust/*.md which contain specifications
of the port of avalanchego to Rust. The specifications should be very detailed:
- Should contain decisions on which external libraries to use
- Should use [reth](https://github.com/paradigmxyz/reth) in place of libevm. reth should already have plenty of hooks / customization support which you can build off of.
If any particular customization points are missing, highlight what wrapper code will be built around reth primitives to allow for the necessary customization.
- Should make sure to use [Firewood](https://github.com/ava-labs/firewood) as a direct Rust dependency.
- As a general rule, high-performance should be maintained and if improvements are noticed while analyzing the Go codebase, make sure to note them as part of the spec.

In terms of development setup (which should be mentioned in the spec):
- It should use Nix flakes to setup all development tool dependencies.
- It should use the latest Bazel and use Bazel Module system properly and use Rust rules along with gazelle extension for Rust if it exists.
- It should define an AGENTS.md and CLAUDE.md necessary for effective long term development.
- It should follow Rust best practices and use clean, modular, generic abstractions when they are appropriate.
- The code should be easy to read and follow while at the same time abstracting when necessary.

In terms of testing strategy (which should be mentioned in the spec):
- Unit-tests + property-based tests are a must for every part of the code base.
- Red/Green TDD should be followed as a developmental methodology.
- All relevant tests from this Avalanchego repo should be replicated in the Rust port.

Use as many subagents as necessary to perform the research by researching each individual point mentioned above and consolidating everything into multiple specification files 
under specs-rust/*.md. It should be complete and standalone for coding agents to fully derive a Rust implementation.
