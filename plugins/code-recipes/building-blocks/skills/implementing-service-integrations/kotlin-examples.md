# Kotlin Examples

## HTTP Client
```kotlin
@Component
class HttpClientPeopleFinder(private val peopleServiceApi: PeopleServiceApi) : PeopleFinder {

    @Throws(HttpCallNonSucceededException::class)
    override fun find(personId: UUID): Person? {
        val wrappedApiResponse = peopleServiceApi.find(personId).execute()
        val apiResponse = extractBody(wrappedApiResponse)
        return apiResponse?.let { Person(it.id, it.firstName, it.lastName) }
    }

    private fun extractBody(response: Response<PersonApiResponse>): PersonApiResponse? =
        when {
            response.isSuccessful -> response.body()!!
            response.code() == 404 -> null
            else -> throw HttpCallNonSucceededException(
                httpClient = this::class.simpleName!!,
                errorBody = response.errorBody()?.charStream()?.readText()?.trimIndent(),
                httpStatus = response.code()
            )
        }
}

interface PeopleServiceApi {

    @GET("/people/{id}")
    fun find(@Path("id") personId: UUID): Call<PersonApiResponse>
}

data class PersonApiResponse(val id: UUID, val firstName: String, val lastName: String)
```

## Kotlin / Spring specifics
- Annotate adapter methods with `@Throws(...)` to document the infrastructure exceptions they can raise
- If Retrofit is used, define Retrofit client interfaces in the same file below the adapter for cohesion, and build the Retrofit instance (base url, timeouts, converters) in a configuration class
- Use `@Component` annotation to declare the adapter bean
