# Examples

## Promise / array chains
```typescript
// Don't
const register = (request: RegisterRequest) =>
    validate(request).then((t) => audit(t).then(() => repository.save(t)));

// Do
async function register(request: RegisterRequest): Promise<Team> {
    const team = await validate(request);
    await audit(team);
    return repository.save(team);
}
```

## Exhaustive switch instead of if-else chains
```typescript
function statusFor(error: AddMemberErrorType): HttpStatus {
    switch (error) {
        case 'MEMBER_LIMIT_REACHED':
            return HttpStatus.UNPROCESSABLE_ENTITY;
        case 'DUPLICATE_MEMBER':
            return HttpStatus.CONFLICT;
    }
}
```

## Parameter object for same-typed parameters
```typescript
transfer({from: sourceAccountId, to: targetAccountId, amount});
```

## TypeScript specifics
- Use `switch` over string-literal unions / enums with `strict` on, so a missing case is a compile error (or add a `never` check in `default`)
- Model value objects and DTOs with `readonly` properties (classes or `interface`s); create new instances instead of mutating
- Prefer `const` over `let`, `readonly T[]` / `ReadonlyArray<T>` for collections that must not change
- Use a destructured parameter object when a function takes several parameters of the same type (TypeScript has no named arguments)
- Prefer `async`/`await` over nested `.then()` chains; don't chain more than one level of `map`/`filter`/`reduce` when it hides the flow
- Don't type values as optional (`?`, `| undefined`, `| null`) when they are always present; avoid non-null assertions (`!`) and `any`
- Don't create classes with only `static` methods - export module-level functions instead
- Name multi-line arrow-function parameters descriptively, not `x` / `t`
- Files in kebab-case named by role (`add-team-member.use-case.ts`, `postgres.team-repository.ts`); group closely related types in one file (`team.errors.ts`)
