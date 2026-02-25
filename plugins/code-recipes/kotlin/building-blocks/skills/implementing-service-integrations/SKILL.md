---
name: implementing-service-integrations
description: Apply when creating, modifying, or reviewing any file that is an outbound HTTP client, REST client adapter, that integrates with an external service via sync http calls. 
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
7. return the result to the application service layer if needed

### Guidelines

**DO:**
- File naming: `<ExternalSystem><EntityClientOrInterfaceImplemented>`, e.g., `HttpClientPeopleFinder`
- Throw infrastructure exceptions for non-successful responses (except 404, it will return null)
- Map external API DTOs to domain entities using private functions
- Use Mapper classes for complex mappings
- Declare DTOs in the same file where they are used
- If Retrofit is used, define Retrofit client interfaces in the same file below for cohesion

**DON'T:**
- Leak http request/response dtos outside of the component, use only domain entities or primitives IN and OUT
- Throw domain exceptions or return domain errors - those belong in domain/application layers
- Include any business logic

## Examples

Please use always these examples as reference: [examples.md](examples.md)