# Examples

## HTTP Client
```kotlin
@Component
class HttpClientPeopleFinder(private val peopleServiceApi: PeopleServiceApi) : PeopleFinder {

    @Throws(HttpCallNonSucceededException::class)
    override fun find(personId: UUID): Person? {
        val wrappedApiResponse = peopleServiceApi.find(personId.value).execute()
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
    fun find(@Path("id") accountId: UUID): Call<PersonApiResponse>
}

data class PersonApiResponse(val id: UUID, val firstName: String, val lastName: String)
```
