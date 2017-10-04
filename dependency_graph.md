# The Pact Dependency Graph of Doom

To minimise the amount of code maintenance, many pact implementations depend on some shared libraries. When the shared libraries are updated, it is important to update the packages that use them.

This graph shows the dependency relationships to assist in updating the libraries.

The `standalone packages` provide the Ruby Pact code bundled along with the Ruby interpreter so that it can be installed and run on a machine without Ruby. There is a package built for each operating system. The `pact ruby standalone package` is a (newer) amalgamation of `pact-mock_service standalone package` and `pact-provider-verifier standalone package`, and should be used in preference to the separate ones as it shares the Ruby interpreter code and hence is a smaller total file size.

* [pact-support gem][pact-support-gem]
    * [pact-mock_service gem][pact-mock-service-gem]
        * [pact ruby standalone package][pact-ruby-standalone]
            * [pact-python][pact-python]
            * [pact-net][pact-net]            
        * [pact-mock-service-docker][pact-mock-service-docker]
        * [pact-ruby-standalone][pact-ruby-standalone]
            * [pact-standalone-npm][pact-standalone-npm]
                * [pact-node][pact-node]
                    * [pact-js][pact-js]         
            * [pact-go][pact-go]
    * [pact gem][pact-gem]
        * [pact ruby standalone package][pact-ruby-standalone]
            * [pact-python][pact-python]
        * [pact-provider-verifier gem][pact-provider-verifier-gem]
            * [pact-provider-verifier standalone package][pact-provider-verifier-standalone]
                * [pact-node][pact-node]
                    *  [pact-js][pact-js]
            * [pact-python][pact-python]                    
            * [pact-provider-verifier-docker][pact-provider-verifier-docker]
            * [pact-go][pact-go]            
        * [pact_broker gem][pact-broker-gem]
            * [pact_broker-docker][pact_broker-docker]


[pact-support-gem]: https://github.com/pact-foundation/pact-support/blob/master/RELEASING.md
[pact-mock-service-gem]: https://github.com/pact-foundation/pact-mock_service/blob/master/RELEASING.md
[pact-mock-service-standalone]: https://github.com/pact-foundation/pact-mock_service/blob/master/packaging/README.md
[pact-gem]: https://github.com/realestate-com-au/pact/blob/master/RELEASING.md
[pact-mock-service-npm]: https://github.com/pact-foundation/pact-mock-service-npm/blob/master/RELEASING.md
[pact-node]: https://github.com/pact-foundation/pact-node/blob/master/RELEASING.md
[pact-js]: https://github.com/pact-foundation/pact-js/blob/master/RELEASING.md
[pact-provider-verifier-gem]: https://github.com/pact-foundation/pact-provider-verifier/blob/master/RELEASING.md
[pact-provider-verifier-standalone]: https://github.com/pact-foundation/pact-provider-verifier/blob/master/RELEASING.md
[pact-provider-verifier-docker]: https://github.com/DiUS/pact-provider-verifier-docker/blob/master/RELEASING.md
[pact-mock-service-docker]: https://github.com/pact-foundation/pact-mock-service-docker/blob/master/RELEASING.md
[pact-broker-gem]: https://github.com/pact-foundation/pact_broker/blob/master/RELEASING.md
[pact_broker-docker]: https://github.com/DiUS/pact_broker-docker/blob/master/RELEASING.md
[pact-python]: https://github.com/pact-foundation/pact-python/blob/master/RELEASING.md
[pact-ruby-standalone]: https://github.com/pact-foundation/pact-ruby-standalone/blob/master/RELEASING.md
[pact-go]: https://github.com/pact-foundation/pact-go/blob/master/RELEASING.md
[pact-net]: https://github.com/SEEK-Jobs/pact-net
