# Dependency graph

To minimise the amount of code maintenance, many pact implementations depend on some shared libraries. When the shared libraries are updated, it is important to update the packages that use them.

This graph shows the dependency relationships to assist in updating the libraries.

```
[pact-support gem][pact-support-gem]
  <- [pact-mock_service gem][pact-mock-service-gem]
    <- [pact-mock_service standalone packages][pact-mock-service-standalone]
    <- [pact gem][pact-gem]
    <- [pact-mock-service-npm][pact-mock-service-npm]
      <- [pact-node][pact-node]
        <- pact-js

  <- pact gem
    <- pact-provider-verifier-docker
    <- pact-provider-verifier gem
    <- pact-provider-verifier standalone packages
      <- pact-node
        <- pact-js

    <- pact_broker gem
      <- pact_broker-docker
      <- pact-broker-docker-private
```

[pact-support-gem]: https://github.com/pact-foundation/pact-support/blob/master/RELEASING.md
[pact-mock-service-gem]: https://github.com/pact-foundation/pact-mock_service/blob/master/RELEASING.md
[pact-mock-service-standalone]: https://github.com/pact-foundation/pact-mock_service/blob/master/packaging/README.md
[pact-gem]: https://github.com/realestate-com-au/pact/blob/master/RELEASING.md
[pact-mock-service-npm]: https://github.com/pact-foundation/pact-mock-service-npm/blob/master/RELEASING.md
[pact-node]: https://github.com/pact-foundation/pact-node
