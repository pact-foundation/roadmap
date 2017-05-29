# Dependency graph

To minimise the amount of code maintenance, many pact implementations depend on some shared libraries. When the shared libraries are updated, it is important to update the packages that use them.

This graph shows the dependency relationships to assist in updating the libraries.


* [pact-support gem][pact-support-gem]
    * [pact-mock_service gem][pact-mock-service-gem]
        * [pact-mock_service standalone packages][pact-mock-service-standalone]
            * [pact-mock-service-npm][pact-mock-service-npm]
                * [pact-node][pact-node]
                    * [pact-js][pact-js]
    * [pact gem][pact-gem]
        * [pact-provider-verifier gem][pact-provider-verifier-gem]
                * [pact-provider-verifier standalone packages][pact-provider-verifier-standalone]
                    * [pact-node][pact-node]
                * [pact-node][pact-node]
                    *  [pact-js][pact-js]
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
[pact-broker-gem]: https://github.com/pact-foundation/pact_broker/blob/master/RELEASING.md
[pact_broker-docker]: https://github.com/DiUS/pact_broker-docker/blob/master/RELEASING.md
