# Code Conventions

## Code Structure

**Methods/Functions:**
- Avoid deep nesting of private methods
- Call private methods directly from public methods when possible
- Keep functions small and focused on one responsibility

## Dependency injection
- Use constructor injection for all dependencies
- Avoid field injection

## Package Structure

**For new projects:**
- Use flat package structure: `domain`, `application`, `infra`
- Skip Java convention prefixes (no `org.example.project`)
- Example: `teams.domain`, `teams.application`, `teams.infra`

**For existing projects:**
- Follow existing package conventions
- Maintain consistency with codebase patterns

## Concurrency
- Use optimistic locking for concurrent data access
- Let database handle concurrency conflicts

## Naming
- Follow established naming conventions in the codebase
- Never use `Impl` suffix for implementations** - Name classes by what they do, not their relationship to an interface
  - ❌ `TeamRepositoryImpl`
  - ✅ `PostgresTeamRepository`, `InMemoryTeamRepository`
  - ❌ `PaymentServiceImpl`
  - ✅ `StripePaymentGateway`, `PayPalPaymentGateway`

