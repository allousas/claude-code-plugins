# Examples

## Domain Exception hierarchy
```typescript
// domain/team.errors.ts
export abstract class DomainException extends Error {
    protected constructor(message: string) {
        super(message);
        this.name = new.target.name; // no need to set name in each subclass
    }
}

export class TeamNotFoundException extends DomainException {
    constructor(public readonly teamId: string) {
        super(`Team with id ${teamId} not found`);
    }
}

export type AddMemberErrorType = 'MEMBER_LIMIT_REACHED' | 'DUPLICATE_MEMBER';

export class AddMemberException extends DomainException {
    constructor(public readonly type: AddMemberErrorType, message: string) {
        super(message);
    }
}
```

## Domain entity throwing exceptions
```typescript
// domain/team.domain.ts
const MAX_MEMBERS = 10;

export class Team {
    constructor(
        public readonly id: string,
        public readonly name: string,
        public readonly members: readonly Member[],
    ) {}

    /** @throws AddMemberException */
    addMember(member: Member): Team {
        if (this.members.length >= MAX_MEMBERS) {
            throw new AddMemberException('MEMBER_LIMIT_REACHED', `Team ${this.id} has reached the maximum of ${MAX_MEMBERS} members`);
        }
        if (this.members.some((existing) => existing.personId === member.personId)) {
            throw new AddMemberException('DUPLICATE_MEMBER', `Person ${member.personId} is already a member of team ${this.id}`);
        }
        return new Team(this.id, this.name, [...this.members, member]);
    }
}
```

## Use case letting exceptions propagate
```typescript
// application/use-cases/add-team-member.use-case.ts
@Injectable()
export class AddTeamMemberUseCase {
    constructor(
        private readonly teamRepository: TeamRepository,
        private readonly domainEventPublisher: DomainEventPublisher,
        private readonly transactionProvider: TransactionProvider,
    ) {}

    /** @throws AddMemberException, TeamNotFoundException */
    async execute(teamId: string, personId: string): Promise<void> {
        await this.transactionProvider.run(async (trx) => {
            const team = await this.teamRepository.find(teamId);
            if (!team) throw new TeamNotFoundException(teamId);

            const updated = team.addMember(new Member(personId));

            await this.teamRepository.save(updated, trx);
            await this.domainEventPublisher.publish(new MemberAdded(teamId, personId), trx);
        });
    }
}
```

## Infrastructure exception (separate from domain)
```typescript
// infrastructure/infrastructure.errors.ts
export abstract class InfrastructureException extends Error {
    protected constructor(message: string, public readonly cause?: unknown) {
        super(message);
        this.name = new.target.name;
    }
}

export class HttpCallNonSucceededException extends InfrastructureException {
    constructor(public readonly httpClient: string, public readonly httpStatus: number, public readonly errorBody?: string) {
        super(`HTTP call failed in ${httpClient} with status ${httpStatus}: ${errorBody}`);
    }
}

export class OptimisticLockingException extends InfrastructureException {
    constructor(entity: string, id: string, cause?: unknown) {
        super(`Concurrent update of ${entity} ${id}`, cause);
    }
}
```
Only wrap where the catch adds meaning — otherwise let it propagate:
```typescript
try {
    await this.db.insertInto('teams').values(row).execute();
} catch (e) {
    if (isUniqueViolation(e)) throw new OptimisticLockingException('Team', row.id, e);
    throw e; // unknown driver error → let it fly to the boundary
}
```

## Boundary handler: status, error code, log and metric
```typescript
// infrastructure/inbound/global-exception.filter.ts
import {ArgumentsHost, Catch, ExceptionFilter, HttpStatus, Injectable, Logger} from '@nestjs/common';
import {Counter} from 'prom-client';

const errors = new Counter({name: 'errors_total', help: 'Errors by type', labelNames: ['type', 'useCase']});

interface ErrorMapping {
    readonly status: HttpStatus;
    readonly code: string;
}

const RESPONSE_BY_ERROR = new Map<Function, (exception: DomainException) => ErrorMapping>([
    [TeamNotFoundException, () => ({status: HttpStatus.NOT_FOUND, code: 'TEAM_NOT_FOUND'})],
    [AddMemberException, (exception) => ({status: HttpStatus.UNPROCESSABLE_ENTITY, code: (exception as AddMemberException).type})],
]);

const UNMAPPED: ErrorMapping = {status: HttpStatus.INTERNAL_SERVER_ERROR, code: 'INTERNAL_ERROR'};

@Injectable()
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
    private readonly logger = new Logger(GlobalExceptionFilter.name);

    catch(exception: unknown, host: ArgumentsHost) {
        const http = host.switchToHttp();
        const request = http.getRequest();
        const useCase = `${request.method} ${request.route?.path ?? 'unknown'}`;
        const type = exception instanceof Error ? exception.name : 'Unknown';
        const toResponse = exception instanceof DomainException ? RESPONSE_BY_ERROR.get(exception.constructor) : undefined;
        const {status, code} = toResponse ? toResponse(exception as DomainException) : UNMAPPED;
        const message = toResponse ? (exception as DomainException).message : 'Internal server error';

        this.logger.warn(`Use case '${useCase}' failed with ${type}: ${exception instanceof Error ? exception.message : String(exception)}`);
        errors.inc({type, useCase});

        return http.getResponse().status(status).json({code, message});
    }
}
```

## Hot exception promoted to a per-operation sealed result
Only after monitoring and a benchmark show `AddMemberException` is hot:
```typescript
export type AddMemberResult =
    | {readonly kind: 'added'; readonly team: Team}
    | {readonly kind: 'memberLimitReached'}
    | {readonly kind: 'duplicateMember'};

addMember(member: Member): AddMemberResult {
    if (this.members.length >= MAX_MEMBERS) return {kind: 'memberLimitReached'};
    if (this.members.some((existing) => existing.personId === member.personId)) return {kind: 'duplicateMember'};
    return {kind: 'added', team: new Team(this.id, this.name, [...this.members, member])};
}
```
The caller switches on `kind`; add a `never` check in the `default` branch so a new variant fails compilation until handled.

## TypeScript / NestJS specifics
- TypeScript has no checked exceptions, so the thrown types are invisible to the compiler — document them with a `/** @throws ... */` JSDoc and rely on the single mapping table plus monitoring
- Set `this.name = new.target.name` in each base class so every subtype is identifiable without per-class boilerplate
- Use `instanceof` (or `constructor` identity), never string-name comparisons, to branch on error type
- Keep the original error as `cause` when wrapping into an `InfrastructureException`
- Group errors per file: domain errors in `domain/<feature>.errors.ts` (`team.errors.ts`), infrastructure errors in `infrastructure/infrastructure.errors.ts`
- Implement one global `@Catch()` filter and register it once via `APP_FILTER` (see `cross-cutting:configuring-runtime-dependencies`). Domain errors resolve through the mapping table; anything else (infrastructure exceptions, unknown errors) is logged and returns 500 without leaking internals. Mapping NestJS's own `HttpException`s is covered in `building-blocks:implementing-controllers`
- An `ExceptionFilter` only gets an `ArgumentsHost`, not the handler method — tag the metric's `useCase` with the HTTP method and route path instead
- In the filter, send exactly one response and `return` after sending
- `DomainException` is **not** a NestJS `HttpException` — never extend `HttpException` for domain errors, and never call `getStatus()` / `getResponse()` on them. The mapping table is the single source of status codes and error codes
- Let NestJS's built-in pipes (`ValidationPipe`, `ParseUUIDPipe`) produce 400s for transport-level input errors
- Cost of exceptions in V8: `new Error()` captures a stack trace at construction. Benchmark the hot operation (e.g. with `tinybench` or `mitata`) comparing throw-based vs. result-type variants before promoting
