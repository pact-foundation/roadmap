# The Pact Dependency Graph of Doom

To minimise the amount of code maintenance, many pact implementations depend on some shared libraries. When the shared libraries are updated, it is important to update the packages that use them.

This graph shows the dependency relationships to assist in updating the libraries.

The `pact ruby standalone` provides the Ruby Pact code bundled along with the Ruby interpreter so that it can be installed and run on a machine without Ruby. There is a package built for each operating system. 

* [pact-support gem][pact-support-gem]
    * [pact-mock_service gem][pact-mock-service-gem]
        * [pact-ruby-standalone][pact-ruby-standalone]
    * [pact gem][pact-gem]
        * [pact-ruby-standalone][pact-ruby-standalone]
        * [pact-provider-verifier gem][pact-provider-verifier-gem]
        * [pact-cli-docker][pact-cli-docker]

* [pact ruby standalone package][pact-ruby-standalone]
    * [pact-python][pact-python]
    * [pact-net][pact-net]
    * [pact-php][pact-php]            
    * [pact-js-core][pact-js-core]
        * [pact-js][pact-js]         
    * [pact-go][pact-go]
    * [pact-consumer-swift][pact-consumer-swift]

[pact-support-gem]: https://github.com/pact-foundation/pact-support/blob/master/RELEASING.md
[pact-mock-service-gem]: https://github.com/pact-foundation/pact-mock_service/blob/master/RELEASING.md
[pact-mock-service-standalone]: https://github.com/pact-foundation/pact-mock_service/blob/master/packaging/README.md
[pact-gem]: https://github.com/pact-foundation/pact-support/blob/master/RELEASING.md
[pact-mock-service-npm]: https://github.com/pact-foundation/pact-mock-service-npm/blob/master/RELEASING.md
[pact-js-core]: https://github.com/pact-foundation/pact-js-core/blob/master/RELEASING.md
[pact-js]: https://github.com/pact-foundation/pact-js/blob/master/RELEASING.md
[pact-provider-verifier-gem]: https://github.com/pact-foundation/pact-provider-verifier/blob/master/RELEASING.md
[pact-provider-verifier-standalone]: https://github.com/pact-foundation/pact-provider-verifier/blob/master/RELEASING.md
[pact-provider-verifier-docker]: https://github.com/DiUS/pact-provider-verifier-docker/blob/master/RELEASING.md
[pact-broker-gem]: https://github.com/pact-foundation/pact_broker/blob/master/RELEASING.md
[pact-python]: https://github.com/pact-foundation/pact-python/blob/master/RELEASING.md
[pact-ruby-standalone]: https://github.com/pact-foundation/pact-ruby-standalone/blob/master/RELEASING.md
[pact-standalone-npm]: https://github.com/pact-foundation/pact-standalone-npm/blob/master/RELEASING.md
[pact-go]: https://github.com/pact-foundation/pact-go/blob/master/RELEASING.md
[pact-net]: https://github.com/pact-foundation/pact-net
[pact-cli-docker]: https://github.com/pact-foundation/pact-ruby-cli
[pact-consumer-swift]: https://github.com/DiUS/pact-consumer-swift
