# TypeScript Examples

All based on NestJS (Express platform), if the project uses any other framework, use the equivalent classes/decorators.

## HTTP Controller (Command - Create)
```typescript
// src/infrastructure/inbound/teams.http-controller.ts
@Controller('teams')
export class TeamsHttpController {

    constructor(private readonly createTeam: CreateTeamUseCase) {}

    @Post()
    async create(
        @Body() request: CreateTeamRequest,
        @Res({passthrough: true}) response: Response,
    ): Promise<CreateTeamResponse> {
        const teamId = await this.createTeam.execute(request.name);
        response.location(`/teams/${teamId}`);
        return {id: teamId};
    }
}

export interface CreateTeamRequest {
    name: string;
}

export interface CreateTeamResponse {
    id: string;
}
```

## HTTP Controller (Command - Operation on a resource)
```typescript
// src/infrastructure/inbound/teams.http-controller.ts
@Controller('teams')
export class TeamsHttpController {

    constructor(private readonly addTeamMember: AddTeamMemberUseCase) {}

    @HttpCode(204)
    @Post(':teamId/members')
    async addMember(@Param('teamId', ParseUUIDPipe) teamId: string, @Body() request: AddMemberRequest): Promise<void> {
        await this.addTeamMember.execute(teamId, request.personId);
    }
}

export interface AddMemberRequest {
    personId: string;
}
```

## Global Exception Filter
Error codes and statuses live in the filter, never on the exception (see `cross-cutting:handling-errors-with-exceptions`):
```typescript
// src/infrastructure/inbound/exception.filters.ts
@Injectable()
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {

    constructor(private readonly errorMonitor: ErrorMonitor) {}

    catch(exception: unknown, host: ArgumentsHost): void {
        const http = host.switchToHttp();
        const request = http.getRequest<Request>();
        const response = http.getResponse<Response>();
        const error = exception instanceof Error ? exception : new Error(`Unknown exception: ${String(exception)}`);

        this.errorMonitor.monitor(error, request);
        const {status, code, message} = responseFor(error);
        response.status(status).json(errorResponse(code, message));
    }
}

function responseFor(error: Error): {status: HttpStatus; code: string; message: string} {
    if (error instanceof TeamNotFoundException) return {status: HttpStatus.NOT_FOUND, code: 'TEAM_NOT_FOUND', message: error.message};
    if (error instanceof AddMemberException) return {status: HttpStatus.UNPROCESSABLE_ENTITY, code: error.type, message: error.message};
    if (error instanceof HttpException) return {status: error.getStatus(), code: 'HTTP_ERROR', message: error.message};
    return {status: HttpStatus.INTERNAL_SERVER_ERROR, code: 'INTERNAL_ERROR', message: 'Internal server error'};
}

const errorsCounter = new Counter({name: 'errors', help: 'Errors by type and use case', labelNames: ['type', 'useCase']});

@Injectable()
export class ErrorMonitor {
    private readonly logger = new Logger(ErrorMonitor.name);

    monitor(error: Error, request?: Request): void {
        const useCase = useCaseFrom(error);
        this.logger.warn(
            `Exception | useCase=${useCase} | exception=${error.name} | message=${error.message} | path=${request?.url} | method=${request?.method}`,
        );
        errorsCounter.inc({type: error.name, useCase});
    }
}

function useCaseFrom(error: Error): string {
    return error.stack?.match(/at (?:async )?(\w+(?:UseCase|QueryHandler))\./)?.[1] ?? 'UnknownUseCase';
}

function errorResponse(code: string, message: string): ErrorResponse {
    return {code, message};
}

export interface ErrorResponse {
    code: string;
    message: string;
}
```

Registered once in the module:
```typescript
providers: [
    ErrorMonitor,
    {provide: APP_FILTER, useClass: GlobalExceptionFilter},
]
```

## TypeScript specifics
- Default a dto to `interface`. Reach for a `class` only when runtime decorators need something to attach to - interfaces are erased at compile time, so a decorator-based `ValidationPipe` has no target and validation silently does nothing. Business validations still belong to the domain
- Because interfaces are hoisted, a dto declared *below* the controller can still be referenced by a method signature above it. That is what makes the "dtos below the main class" rule cost nothing
- Name files in kebab-case after the resource: `teams.http-controller.ts`, `exception.filters.ts`

## NestJS specifics
- Let pipes carry transport-level parsing in the signature (`@Param('teamId', ParseUUIDPipe) teamId: string`) so the handler body receives values already known-good, and the 400 comes from the framework rather than a hand-rolled check
- Use `@HttpCode()` when the default status is wrong (`POST` defaults to 201, everything else to 200); use `@Res({passthrough: true})` only to set headers such as `Location`, never to write the body by hand
- For global error handling, use an `ExceptionFilter` with `@Catch()`. Register it once via `APP_FILTER` in the module rather than also calling `app.useGlobalFilters()` - see `cross-cutting:configuring-runtime-dependencies`
- Map NestJS's own `HttpException`s (thrown by pipes, guards, unknown routes) by their status, so framework 400/404s are not turned into 500s
