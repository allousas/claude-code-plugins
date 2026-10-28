# TypeScript Examples

All based on `@nestjs/axios` `HttpService`; with native `fetch` or plain axios keep the same structure.

## HTTP Client
```typescript
// src/infrastructure/outbound/http-client.people-finder.ts
@Injectable()
export class HttpClientPeopleFinder implements PeopleFinder {

    constructor(private readonly http: HttpService) {}

    async find(personId: string): Promise<Person | null> {
        const response = await firstValueFrom(
            this.http.get<PersonApiResponse>(`/people/${personId}`, {validateStatus: () => true}),
        );
        const apiResponse = this.extractBody(response);
        return apiResponse ? new Person(apiResponse.id, apiResponse.firstName, apiResponse.lastName) : null;
    }

    private extractBody(response: AxiosResponse<PersonApiResponse>): PersonApiResponse | null {
        if (response.status >= 200 && response.status < 300) return response.data;
        if (response.status === 404) return null;
        throw new HttpCallNonSucceededException(HttpClientPeopleFinder.name, response.status, JSON.stringify(response.data));
    }
}

interface PersonApiResponse {
    id: string;
    firstName: string;
    lastName: string;
}
```

```typescript
// src/infrastructure/infrastructure.errors.ts
export class HttpCallNonSucceededException extends InfrastructureException {
    constructor(httpClient: string, public readonly httpStatus: number, public readonly errorBody?: string) {
        super(`${httpClient} call failed with status ${httpStatus}`);
    }
}
```

Client configured in the module:
```typescript
imports: [
    HttpModule.register({baseURL: process.env.PEOPLE_SERVICE_URL, timeout: 2000}),
],
providers: [
    {provide: PeopleFinder, useClass: HttpClientPeopleFinder},
]
```

## TypeScript / NestJS specifics
- TypeScript has no checked exceptions; document thrown infrastructure exceptions by their type in `infrastructure.errors.ts` (extending `InfrastructureException`) instead of `@Throws`
- Disable axios's throw-on-non-2xx (`validateStatus: () => true`) and branch on the status explicitly, so 404 → `null` and other failures → `HttpCallNonSucceededException` are visible in one place instead of hidden in a `catch`
- `HttpService` returns Observables; convert with `firstValueFrom` so the adapter exposes a plain `Promise`
- Return `Promise<Entity | null>` for lookups that can miss; keep response `interface`s unexported below the class
- Configure base url and timeouts with `HttpModule.register` (one module/registration per external service) and bind the adapter to its port - see `cross-cutting:configuring-runtime-dependencies`
- Name files `<client>.<port>.ts` in kebab-case (`http-client.people-finder.ts`)
