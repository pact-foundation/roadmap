---
name: From Ruby to Rust - Improving the developer experience of Pact
started: (fill me in with today's date, YYYY-MM-DD)
pr: pact-foundation/roadmap#0000
tracking_issue: pact-foundation/roadmap#0000
---
<!--
The following is a template to help you write your RFC. You don't have to strictly follow this template, but it should help you to consider the various aspects of your proposal. If you have any questions, feel free to ask in the PR or in the Pact Slack community.
-->
## Summary

Simplify the Pact developer experience, by consolidating our cli toolings into a single binary built with rust, to reduce mental model complexity by migrating pact-ruby to the rust core.

## Motivation

The Pact ecosystem has grown over the 12 years since it’s inception. From a ruby library for ruby consumers, for non ruby users via a command line bundle that packaged the ruby runtime and the pact ruby core and pact broker client, a standalone implementation in Java, and a complete rewrite of the Ruby core, in Rust.

Pact itself prides itself on being a tool to aid in managing api sprawl, however our own Pact ecosystem itself has seen its fair share of sprawl over the years.

The Ruby and Rust core have different feature support, Ruby supporting up to Pact V2 specification, and Rust supporting up to Pact V4, and consuming libraries have different levels of adoption.

Over the past three years, we have seen many of our client libraries upgraded to use the rust core, via the distribution mechanism of a c shared library ( compiled from rust), over the previous cli based interactions. This brought a raft of benefits, but is not the main subject of this thread, so I will park that for now.

The rust ecosystem is split up into several crates, some of which are distributed as cli applications, for standalone use.

There hasn’t been a consolidated package of all of our rust tools in a single delivery mechanism ( either standalone or via docker ) until recently where the were added into pact-standalone and pact-docker-cli respectively.

### Current Challenges

- rust executable naming is inconsistent, making building a mental model difficult
- rust executables are not composable missing benefits to create a single entry point with shared deps, and smaller total footprint that indiv binaries
- each rust tool is not a direct counterpart for its ruby sibling, and the migration path isn’t well documented. for the most part, this is less of an issue, client libraries use the ffi now, meaning less user rely on the cli tools.

## Guide-level explanation

Explain the proposal as if it were already a part of Pact. That generally means:

- Explaining the feature largely in terms of examples.

## 💁 Example

Composed cli: https://github.com/YOU54F/pact-cli/blob/main/src/cli.rs#L11-L29

```rust
pub fn build_cli() -> Command {
    let app = Command::new("pact_cli")
        .about("A pact cli tool")
        .subcommand(
            pact_broker_cli::cli::pact_broker_client::add_pact_broker_client_command()
                .name("pact-broker")
                .subcommand(add_standalone_broker_subcommand())
                .subcommand(add_docker_broker_subcommand()),
        )
        .args(pact_broker_cli::cli::add_logging_arguments())
        .subcommand(
            pact_broker_cli::cli::pactflow_client::add_pactflow_client_command().name("pactflow"),
        )
        .subcommand(add_completions_subcommand())
        .subcommand(pact_plugin_cli::Cli::command().name("plugin"))
        .subcommand(pact_mock_server_cli::setup_args().name("mock"))
        .subcommand(pact_verifier_cli::args::setup_app().name("verifier"))
        .subcommand(pact_stub_server_cli::build_args().name("stub"));
    app
}
```

Cargo toml: https://github.com/YOU54F/pact-cli/blob/main/Cargo.toml#L43-L47

```toml
pact-stub-server = { version = "*", git = "https://github.com/YOU54F/pact-stub-server.git", branch = "feat/cli_as_lib"}
pact_mock_server_cli = { version = "2.0.0-beta.1", git = "https://github.com/YOU54F/pact-core-mock-server.git", branch = "feat/cli_as_lib"}
pact-plugin-cli = { version = "*", git = "https://github.com/pact-foundation/pact-plugins", branch = "feat/cli_as_lib"}
pact_verifier_cli = { version = "*",  git = "https://github.com/YOU54F/pact-reference", branch = "feat/cli_as_lib"}
pact-broker-cli = { version = "0.2.0" }
```

## 🔨 How To Test

```sh
git clone git@github.com:YOU54F/pact-cli.git
cd pact-cli
RUSTFLAGS=-Awarnings cargo run -q --
```

output

```console
A pact cli tool

Usage: pact_cli [OPTIONS] [COMMAND]

Commands:
  pact-broker  
  pactflow     
  completions  Generates completion scripts for your shell
  plugin       CLI utility for Pact plugins
  mock         Standalone Pact mock server
  verifier     Standalone pact verifier for provider pact verification
  stub         Pact Stub Server 0.6.3
  help         Print this message or the help of the given subcommand(s)

Options:
      --log-level <LEVEL>  Set the log level (none, off, error, warn, info, debug, trace) [default: off] [possible values: off, none, error, warn, info, debug, trace]
  -h, --help               Print help
```

usage

```sh
➜  pact_cli git:(multi-cli) RUSTFLAGS=-Awarnings cargo run -q -- stub -f tests/pacts/GettingStartedOrderWeb-GettingStartedOrderApi.json 
2025-10-08T21:45:53.218070Z  INFO main pact_stub_server_cli: Loaded 1 pacts (1 total interactions)
2025-10-08T21:45:53.218530Z  INFO tokio-runtime-worker pact_stub_server_cli::server: Server started on port 52196
2025-10-08T21:46:30.133081Z  INFO tokio-runtime-worker pact_stub_server_cli::server: ===> Received HTTP Request ( method: GET, path: /orders, query: None, headers: Some({"host": ["localhost:52196"], "accept": ["*/*"], "user-agent": ["curl/8.7.1"]}), body: Empty )
2025-10-08T21:46:30.135928Z  INFO tokio-runtime-worker pact_stub_server_cli::pact_support: <=== Sending HTTP Response ( status: 200, headers: Some({"Content-Type": ["application/json; charset=utf-8"]}), body: Present(63 bytes) )
```

```sh
➜  pact_cli git:(multi-cli) curl http://localhost:52196/orders
[{"id":1,"items":[{"name":"burger","quantity":2,"value":100}]}]%   
```

- Explaining how Pact users should *think* about the feature, and how it should impact the way they use Pact. It should explain the impact as concretely as possible.
	- If I need to use Pact, I need the `pact` cli, it should be top of mind in the docs and a one liner to install
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
	- `PACT_CLI_LEGACY` flag in `pact-standalone` / `pact-docker-cli` to allow reverting back to original ruby implementation
- If applicable, describe the differences between teaching this to existing Pact users and new Pact users.
	- We just need to tell users to install a single cli mechanism
	- Simple copy/paste install script on website
	- Just need to explain each cli purpose and function
	- tell users that some cli func is available in native language form
	- prefer language based dsl for mock-server / verifier
	- use pact cli to start up a local pact broker either with docker/ruby
	- use pact cli to 
		- interact with broker
		- use pact stub server func to use generated as a stub for integration tests
		- use pact plugin cli to install custom plugin
- Discuss how this impacts existing Pacts, and the Pact ecosystem in general.
	- `pact-broker-cli` is build as a 1-2-1 replacement
	- `pact_mock_server_cli`  / `pact_verifier_cli` / `pact-stub-server` are not direct drop in replacements
		- most users do not rely on the `pact_mock-service` / `pact-provider-verifier` unless you are using pact-ruby.
			- pact-ruby will be upgraded to support the rust core, as part of this motion
		- pact-python v2 is now using pact-rust core, and has ruby fallback. pact-python v3 bring rust support
	- we may break some users esoteric flows, these will at least be uncovered, before projects are archived

### Proposed Benefits

a single standalone executable containing all of pact’s latest cli functionality 

- smallest total size
- easier to newbies
- easier to install/use
- easier to work with security team requirements ( no need to explain why ruby is running )
- easier to document

### Proposed actions 


### rewrite pact-broker-cli in rust

1. create `pact-broker-cli` crate in rust
2. should be fully backwards compatible with existing pact_broker-client
3. should target at least all existing supported pact-standalone targets

### make rust cli naming conventions align

1. Provide guidelines on rust crate / cli / library naming conventions
2. Align binary names
3. Avoid changing existing crate names, but binary renames are allowed
#### Conventions

- binary names separated by hyphens
- convention 
	- `pact-<tool name>` for crates which are libs / clis
	- `pact-<tool name>-cli` for crates which are clis only
	- `pact_<tool name>` for crates which are libs only
	- `pact_<tool name>` for libraries imports
	- `pact_<tool name>_cli` for cli libraries imports
	- `pact-<tool name>` for cli binaries
- drops cli suffix from all tools
- crates names may be separated by hyphens or underscores, existing crates unaffected
- library imports are always used as underscores, with cli based libraries allows either or mixed underscore/hyphen

#### Impact

- some binaries will be renamed
	- pact_mock_server_cli to pact-mock-server
	- pact-plugin-cli to pact-plugin
	- pact_verifier_cli to pact-verifier
	- pact-broker-cli to pact-broker
- End result after renaming
	- pact
	- pact-broker
	- pact-plugin
	- pact-mock-server
	- pact-stub-server
- CLI layout in single cli
	- pact
		- broker
		- plugin
		- mock-server
		- stub-server

### make rust cli’s composable

1. Seperate logic from binary only, to shared `lib.rs`
2. Allow binaries to be imported and composed in a meta cli via `<crate_name>_cli`
	1. Library imports are down-cased by default


### create pact-cli in rust, composed of all pact rust cli’s

1.  create `pact` crate in rust
	1. owned namespace provided from now renamed `codas` project
2. Agree on nomenclature for subcommands
3. Release to
	1. Cargo as `pact`
	2. Docker as `pact-foundation/pact` (ghcr) / `pactfoundation/pact` (docker.io)
	3. Homebrew as `pact-foundation/pact` via `homebrew-pact` repo


### add new cli's to existing distribution mechanism

1. Existing mechanisms are 
	1. pact-standalone
	2. homebrew-pact-standalone
	3. pact-docker-cli
	4. pactflow/actions
2. add pact-cli to packages
	- expose new `pact-cli` executable
	- divert existing commands to rust executables
		- without executing ruby runtime (via wrapper or bat scripts)
	- add env based fallback to ruby tools `PACT_CLI_LEGACY`
	- pubish as `minor` version bump
3. Archival & Deprecation
	1. Remove ruby runtime in major version
	2. A tool nomeclature in major version
	3. Publish archival notices
	4. Archive

### create new delivery mechanisms

	- docker
		- `pact-foundation/pact` (ghcr)
		- `pactfoundation/pact` (docker.io)
	- cargo
		- `pact`
		- `pact-broker-cli`
	- install scripts
		- unix shell (posix, non bash specific)
		- windows powershell
	- github action
		- `- uses: you54f/pact-cli@main`
	- distribution mechanisms
		- npm
			- `@pact-foundation/pact-cli`
		- pypi
			- `pip install pact-python-cli`
		- homebrew
			- `brew install pact-foundation/pact`
		- choco
			- `choco install pact`
		- scoop
			- `scoop install pact`
		- alpine pkgs
			- is this neccessary if we have 1 liner install scripts?
		- rpm
			- is this neccessary if we have 1 liner install scripts?
		- deb
			- is this neccessary if we have 1 liner install scripts?

### migrate pact ruby to pact rust core

1. transfer `you54f/pact-ruby-ffi` to `pact-foundation`
2. merge pact-ruby v2 changes
3. release as `minor`
	1. no breaking changes to existing pact-ruby v1 code
	2. some new dependencies introduced to support pact-ruby v2 code
4. in release v2.x
	1. remove pact v1 code
	2. migrate `v2` namespaced code to main lib

### archival of ruby code

	- post pact-ruby v2.x release
		- pact-message
		- pact-provider-verifier
		- pact-stub-service
		- pact_mock-service
	- post pact-cli release with new distrubition mechanisms
		- pact_broker client
	- post release of both of above
		- pact-docker-cli
		- pact-standalone
		- homebrew-pact-standalone
	- pact-support
		- migrate all pact_broker required code, into pact_broker
		- or migrate all unrequired shared code from pact-support
	


## Reference-level explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.




## End State CLI Tooling

| OS      | Architecture | Supported |
| ------- | ------------ | --------- |
| MacOS   | x86_64       | ✅         |
| MacOS   | arm64        | ✅         |
| Linux   | x86_64       | ✅         |
| Linux   | arm64        | ✅         |
| Windows | x86_64       | ✅         |
| Windows | arm64        | ✅         |
- End result after renaming
	- pact
	- pact-broker
	- pact-plugin
	- pact-mock-server
	- pact-stub-server
- CLI layout in single cli
	- pact
		- broker
		- plugin
		- mock-server
		- stub-server


Each available via

- Docker
	- DockerHub
	- GHCR
- HomeBrew
- Install Scripts
	- Unix
	- Windows



- ✅ Supported
- 🗑 Retired

| Crate/Gem Name            | Binary Name          | Status   | Pact Spec                     | Repo                                                | Release                                             |
| ------------------------- | -------------------- | -------- | ----------------------------- | --------------------------------------------------- | --------------------------------------------------- |
| pact-cli                  | pact                 | ✅        | n/a                           | [GitHub][pact-cli]                                  | [pact-cli-releases][pact-cli-releases]              |
| pact_core_mock_server_cli | pact-mock-server     | ✅        | v1.1 -> v4                    | [GitHub][mock-cli]                                  | [pact_mock_server-cli-releases][mock-cli-releases]  |
| pact_verifier_cli         | pact-verifier        | ✅        | v1.1 -> v4                    | [GitHub][verifier-cli]                              | [pact_verifier-cli-releases][verifier-cli-releases] |
| pact-stub-server          | pact-stub-server     | ✅        | v4                            | [GitHub][stub-cli]                                  | [pact-stub-server-cli-releases][stub-cli-releases]  |
| pact-plugin-cli           | pact-plugin          | ✅        | v4                            | [GitHub][plugin-cli]                                | [plugin-cli-releases][plugin-cli-releases]          |
| pact-broker-cli           | pact-broker          | ✅        | n/a                           | [GitHub][broker-cli]                                | [pact-broker-cli releases][pact-broker-cli-release] |
| pact-broker-cli           | pact-broker pactflow | ✅        | n/a                           | [GitHub][broker-cli]                                | [pact-broker-cli release][pact-broker-cli-releases] |
| pact-broker (client)      | 🗑                   | n/a      | [GitHub][broker-client-cli]   | [pact-standalone releases][pact-standalone-release] |                                                     |
| pactflow                  | 🗑                   | n/a      | [GitHub][pactflow-client-cli] | [pact-standalone releases][pact-standalone-release] |                                                     |
| pact                      | 🗑                   | n/a      | [GitHub][pact-cli]            | [pact-standalone releases][pact-standalone-release] |                                                     |
| pact-message              | 🗑                   | v3       | [GitHub][message-cli-legacy]  | [pact-standalone releases][pact-standalone-release] |                                                     |
| pact-mock-service         | 🗑                   | v1 -> v2 | [GitHub][mock-cli-legacy]     | [pact-standalone releases][pact-standalone-release] |                                                     |
| pact-provider-verifier    | 🗑                   | v1 -> v2 | [GitHub][verifier-cli-legacy] | [pact-standalone releases][pact-standalone-release] |                                                     |
| pact-stub-service         | 🗑                   | v2       | [GitHub][stub-cli-legacy]     | [pact-standalone releases][pact-standalone-release] |                                                     |

[verifier-cli]: https://github.com/pact-foundation/pact-reference/tree/master/rust/pact_verifier_cli
[stub-cli]: https://github.com/pact-foundation/pact-stub-server
[mock-cli]: https://github.com/pact-foundation/pact-core-mock-server/tree/main/pact_mock_server_cli
[plugin-cli]: https://github.com/pact-foundation/pact-plugins/tree/main/cli
[verifier-cli-releases]: https://github.com/pact-foundation/pact-reference/releases
[stub-cli-releases]: https://github.com/pact-foundation/pact-stub-server/releases
[mock-cli-releases]: https://github.com/pact-foundation/pact-core-mock-server/releases
[pact-broker-cli-releases]: https://github.com/pact-foundation/pact-broker-cli/releases
[pact-cli-releases]: https://github.com/you54f/pact-cli/releases
[plugin-cli-releases]: https://github.com/pact-foundation/pact-plugins/releases
[verifier-cli-legacy]: https://github.com/pact-foundation/pact-provider-verifier
[stub-cli-legacy]: https://github.com/pact-foundation/pact-stub-service
[message-cli-legacy]: https://github.com/pact-foundation/pact-message-ruby
[mock-cli-legacy]: https://github.com/pact-foundation/pact-mock_service
[broker-client-cli]: https://github.com/pact-foundation/pact_broker-client
[broker-cli]: https://github.com/pact-foundation/pact_broker-cli
[pactflow-client-cli]: https://github.com/pact-foundation/pact_broker-client?tab=readme-ov-file#provider-contracts-pactflow-only
[pact-cli]: https://github.com/pact-foundation/pact-ruby/tree/master/lib/pact/cli
[wrapper]: /wrapper_implementations
[pact-standalone-release]: https://github.com/pact-foundation/pact-standalone/releases

## Docker

- ✅ Supported
- 🗑 In retirement phase

| Name                   | Status | DockerHub                        | GitHub Container Registry               | Repo                                        |
| ---------------------- | ------ | -------------------------------- | --------------------------------------- | ------------------------------------------- |
| pact-broker            | ✅      | [DockerHub][pact-broker-docker]  | [GHCR][pact-broker-docker-github]       | [pact-ruby-cli][pact-broker-docker-repo]    |
| pact-broker-chart      | ✅      |                                  | [GHCR][pact-broker-chart-docker-github] | [pact-broker-chart][pact-broker-chart-repo] |
| pact                   | ✅      | [DockerHub][pact]                | [GHCR][pact-docker-github]              | [pact-docker-cli][pact-docker-repo]         |
| pact-mock-server       | ✅      | [DockerHub][mock-cli-docker]     |                                         | [pact-reference][mock-cli-docker-repo]      |
| pact-verifier          | ✅      | [DockerHub][verifier-cli-docker] |                                         | [pact-reference][verifier-cli-docker-repo]  |
| pact-stub-server       | ✅      | [DockerHub][stub-cli-docker]     |                                         | [pact-stub-server][stub-cli-docker-repo]    |
| pact (top level entry) | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact-broker (client)   | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pactflow               | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact_mock_server_cli   | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact_verifier_cli      | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact-stub-server       | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact-plugin-cli        | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pactflow-ai            | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact-message           | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact-mock-service      | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact-provider-verifier | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |
| pact-stub-service      | 🗑     | [DockerHub][pact-cli-docker]     | [GHCR][pact-cli-docker-github]          | [pact-docker-cli][pact-cli-docker-repo]     |

[verifier-cli-docker]: https://hub.docker.com/r/pactfoundation/pact-ref-verifier
[stub-cli-docker]: https://hub.docker.com/r/pactfoundation/pact-stub-server
[mock-cli-docker]: https://hub.docker.com/r/pactfoundation/pact-ref-mock-server
[pact-docker]: https://hub.docker.com/r/you54f/pact
[pact-docker-github]: https://github.com/you54f/pact-cli/pkgs/container/pact
[pact-cli-docker]: https://hub.docker.com/r/pactfoundation/pact-cli
[pact-cli-docker-github]: https://github.com/pact-foundation/pact-ruby-cli/pkgs/container/pact-cli
[pact-broker-docker]: https://hub.docker.com/r/pactfoundation/pact-broker
[pact-broker-docker-github]: https://github.com/pact-foundation/pact-broker-docker/pkgs/container/pact-broker
[pact-broker-chart-docker-github]: https://github.com/pact-foundation/pact-broker-chart/pkgs/container/pact-broker-chart%2Fpact-broker
[verifier-cli-docker-repo]: https://github.com/pact-foundation/pact-reference/blob/master/rust/pact_verifier_cli/Dockerfile
[stub-cli-docker-repo]: https://github.com/pact-foundation/pact-stub-server/tree/master/docker
[mock-cli-docker-repo]: https://github.com/pact-foundation/pact-reference/blob/master/rust/pact_mock_server_cli/Dockerfile
[pact-cli-repo]: https://github.com/you54f/pact-cli
[pact-cli-docker-repo]: https://github.com/pact-foundation/pact-ruby-cli
[pact-broker-docker-repo]: https://github.com/pact-foundation/pact-broker-docker
[pact-broker-chart-repo]: https://github.com/pact-foundation/pact-broker-chart

## Homebrew

| Name                   | Status | Repo                  |
| ---------------------- | ------ | --------------------- |
| pact                   | ✅      | [homebrew-pact]       |
| pactflow               | 🗑     | [homebrew-standalone] |
| pact_mock_server_cli   | 🗑     | [homebrew-standalone] |
| pact_verifier_cli      | 🗑     | [homebrew-standalone] |
| pact-stub-server       | 🗑     | [homebrew-standalone] |
| pact-plugin-cli        | 🗑     | [homebrew-standalone] |
| pact                   | 🗑     | [homebrew-standalone] |
| pact-message           | 🗑     | [homebrew-standalone] |
| pact-mock-service      | 🗑     | [homebrew-standalone] |
| pact-provider-verifier | 🗑     | [homebrew-standalone] |
| pact-stub-service      | 🗑     | [homebrew-standalone] |

[homebrew-standalone]: https://github.com/pact-foundation/homebrew-pact-standalone
[homebrew-pact]: https://github.com/pact-foundation/homebrew-pact
###  Homebrew Pact Supported Platforms

| OS    | Architecture | Supported |
| ----- | ------------ | --------- |
| OSX   | x86_64       | ✅         |
| OSX   | arm64        | ✅         |
| Linux | x86_64       | ✅         |
| Linux | arm64        | ✅         |


## Drawbacks

Why should we *not* do this?

We have previously held off on doing this due to other priorities but I strongly feel now is the time to greatly improve our Pact developer experience

## Rationale and alternatives

- Why is this design the best in the space of possible designs?

	- Migrates users away from pact-ruby core
	- negates need to continue to build esoteric packaging system (traveling-ruby)
	- Gives users the latest features
	- Gives users the latest features, in the simplest mechanisms
	- A single binary composed of all our Pact projects, which shares dependencies, takes us the most minimal disk space

- What other designs have been considered and what is the rationale for not choosing them?
	- other languages, but not preferred due to inability to compose rust applications and share deps

- What is the impact of not doing this?
	- prolonged pain of a fragmented ecosystem

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
	- naming conventions
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
	- issues in `pact-broker-cli` implementation, when introduced as the default, through existing distribution mechanisms
		- fallback provided to allow users to revert to existing pact-ruby ecosystem, to iron our any wrinkles
	- system compatibility
		- should be improved by single binary, and our building mechanisms
			- old versions of glibc
			- static musl versions
			- macos deployment target set to `11.0`
			- windows performance, as the ruby distrib mechanism could not provide native gems

## Future possibilities

make the cli dx experience killer

- `pact doctor` - check for common project setup issues
	- could be scoped to lang `pact doctor <lang>`
- `pact broker docker info` - run pact broker locally with docker
- `pact broker ruby info` - run pact broker locally with system /userinstalled ruby
- `pact init <lang>` - setup pact for a particular language
- `pact docs` - links to docs etc
- `pact docs search` - search pact docs algolia api

give me more juice!
