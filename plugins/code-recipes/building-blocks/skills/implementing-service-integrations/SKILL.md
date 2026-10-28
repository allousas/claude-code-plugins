---
name: implementing-service-integrations
description: Invoke when writing or reviewing code that calls another service over HTTP (including plans), or when the user asks about integrating an external API. Covers HTTP client adapters, timeouts, retries, error handling and mapping external responses.
---

## Purpose
External service integrations translate domain calls into external API calls.

## Typical Flow
1. Receive domain parameters from the application service layer
2. Map domain parameters to external API DTO if needed
3. Call external REST API using an http client 
4. Map answer to http response DTO
5. Handle happy case-scenario: map back to a domain entity if needed
6. Handle non-happy case-scenarios
7. Return the result to the application service layer if needed

## Guidelines

**DO:**
- File naming: `<ExternalSystem><EntityClientOrInterfaceImplemented>`, e.g., `HttpClientPeopleFinder`
- Implement a domain/outbound port (e.g. `PeopleFinder`), so application services never depend on the http client
- Throw infrastructure exceptions for non-successful responses (except 404, it will return null)
- Map external API DTOs to domain entities using private functions
- Use Mapper classes for complex mappings
- Declare DTOs in the same file where they are used
- Declare declarative http client interfaces (if the client library uses them) in the same file below for cohesion
- Configure base url, timeouts and retries in the client configuration, not in the adapter

**DON'T:**
- Leak http request/response dtos outside of the component, use only domain entities or primitives IN and OUT
- Throw domain exceptions or return domain errors - those belong in domain/application layers
- Include any business logic

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
