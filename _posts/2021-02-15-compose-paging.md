---
layout: page
current: about
cover:  assets/images/book-pages.jpg
navigation: true
title: Pagination with Jetpack Compose
date: 2021-02-08 21:00:00
tags: [Jetpack Compose Android Pagination]
class: post-template
subclass: 'post'
author: ian
---

Declarative UIs are on the rise in Android with Jetpack Compose. We are going to explore how to use the paging library component.

During this walk through, we will be consuming the [Rick and Morty REST API](https://rickandmortyapi.com/). 

It provides us with pagination capabilities in the responses provided as follows:

```json
{
  "info": {
    "count": 671,
    "pages": 34,
    "next": "https://rickandmortyapi.com/api/character/?page=2",
    "prev": null
  },
  "results": [
    // ...
  ]
}
```

## Prerequisites

At time of writing I'm using Android Studio Canary 6 version. Jetpack Compose is still in alpha development phase and it is only available in canary builds. 

You can download the latest canary builds from [https://developer.android.com/studio/preview](https://developer.android.com/studio/preview)

Add the following Compose dependencies in your ```build.gralde.kts``` :

```kotlin
    implementation("androidx.compose.ui:ui:1.0.0-alpha12")
    implementation("androidx.compose.ui:ui-graphics:1.0.0-alpha12")
    implementation("androidx.compose.ui:ui-tooling:1.0.0-alpha12")
    implementation("androidx.compose.foundation:foundation:1.0.0-alpha12")
    implementation("androidx.compose.foundation:foundation-layout:1.0.0-alpha12")
    implementation("androidx.compose.material:material:1.0.0-alpha12")

    //Paging 
    implementation("androidx.paging:paging-compose:1.0.0-alpha07")
``` 

## Implement our Network Layer & Repository

Firstly, we will be fetching a list of characters from the [https://rickandmortyapi.com/api/character/](https://rickandmortyapi.com/api/character/) endpoint. We will be paging the endpoint by using the query parameter <span style="color:black">?page</span>

Example:
[https://rickandmortyapi.com/api/character/?page=20](https://rickandmortyapi.com/api/character/?page=20)

We will be using Square's [Retrofit](https://square.github.io/retrofit/) library to communicate with our API.

Implement our Retrofit Interface:
```kotlin
interface RickMortyService {

    @GET("/api/character/")
    suspend fun retrieveCharacters(@Query("page") page: Int): Characters
}
```

Then we can retrieve the data from inside our repository as follows:

```kotlin
class CharactersRepository @Inject constructor(private val service: RickMortyService) {

    suspend fun characters(page: Int) = service.retrieveCharacters(page)
}
```

## Implement our Paging Data Source

Since [Paging 3](https://developer.android.com/topic/libraries/architecture/paging/v3-overview) is purely written in Kotlin it allows us to make use of the Kotlin features such as [Kotlin Flows](https://developer.android.com/kotlin/flow). In the past it used to be tedious to write pagination support for our apps. We used to have to write a lot of boilerplate with the old paging APIs. That's not the case anymore. It's now simplified by just overriding the ```PagingSource``` abstract class functions.

Create a data source class called ``CharactersPagingSource``. We will implement the abstract functions, ``load`` and ``getRefreshKey``. In our ``load`` function, we can extract the existing page from the ``params`` parameter which accepts an Integer type of ``LoadParam``. Then we can use our page key to increment and decrement our pages by creating a ```LoadResult``` with our data and page keys.

Full data source code as follows:

```kotlin
class CharactersPagingSource @Inject constructor(private val repository: CharactersRepository): PagingSource<Int, Character>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Character> {
        return try {
            val page = params.key ?: 1

            val response = repository.characters(page)
            val characters = response.characters

            val prevKey = if (page > 0) page - 1 else null
            val nextKey = if (response.info.next != null) page + 1 else null

            LoadResult.Page(data = characters, prevKey = prevKey, nextKey = nextKey)
        } catch (exception: IOException) {
            LoadResult.Error(exception)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, Character>): Int? {
        return state.anchorPosition
    }
}
```   

## Using our Paging implementation with Compose

In order to observe and update our UIs with data in our Activity/Fragment we would use a ViewModel. In our view model, we will use ```Flow``` to emit values to our UI. 

Firstly, we wrap our ``PagingData`` with our ``Flow`` type. Then initialise a ``Pager`` object by passing a ``PagerConfig`` object to configure our initial page size to 20 items. Then we pass down our ``CharactersPagingSource`` instance. 

```kotlin
@HiltViewModel
class CharactersViewModel @Inject constructor(
    private val charactersPagingSource: CharactersPagingSource,
) : ViewModel() {

    val characters: Flow<PagingData<Character>> = Pager(PagingConfig(pageSize = 20)) {
        charactersPagingSource
    }.flow

}
```

Now we are wired up and ready to go! 

In our Activity, we can start creating our declarative UI by using Compose. We can set our theme composable and wrap it in a Scaffold composable with a top app bar.

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    private val charactersViewModel: CharactersViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            RickyMortyComposeTheme {
                CataloguesScreen(charactersViewModel)
            }
        }
    }
}
```

To collect our pagination data, we use the Flow extension ```collectAsLazyPagingItems```

```kotlin

@Composable
fun CataloguesScreen(charactersViewModel: CharactersViewModel) {
    val characters = charactersViewModel.characters.collectAsLazyPagingItems()

    Scaffold(
        topBar = {
            TopAppBar(title = { Text(text = "Rick & Morty Catalogue") })
        },
        bodyContent = {
            RowList(characters = characters, innerPadding = it)
        }
    )
}
```

In our Row Composable, we pass in the paging items and use the items extension from the ``LazyPagingItems`` composable. We can check for state by using ``loadState`` and checking for states such as ``LoadState.Error`` or ``LoadState.Loading``. 


```kotlin
@Composable
fun RowList(characters: LazyPagingItems<Character>, innerPadding: PaddingValues) {
    LazyColumn(contentPadding = innerPadding) {

        items(characters) { character ->
            character?.let {
                CardLayout(character = it)
            }
        }

        characters.apply {
            when {
                loadState.append is LoadState.Error -> {
                    item { ErrorContent() }
                }
                loadState.append is LoadState.Loading -> {
                    item { LoadingContent() }
                }
                loadState.refresh is LoadState.Loading -> {
                    item { LoadingView() }
                }
            }
        }
    }
}
```


## Conclusion
The abstraction provided by the new Paging 3 API makes it faster and easier to test our pagination solutions. The support with Jetpack Compose makes writing Recycler Views a thing of the past.  

Full code can be found on this github repository [https://github.com/IanArb/RickMortyComposeSample](https://github.com/IanArb/RickMortyComposeSample)

## References & Resources
Official Paging 3 Documentation
[https://developer.android.com/topic/libraries/architecture/paging/v3-overview](https://developer.android.com/topic/libraries/architecture/paging/v3-overview)
<br/>
<br/>
Inspiration on getting started with pagination with Compose 
[https://proandroiddev.com/infinite-lists-with-paging-3-in-jetpack-compose-b095533aefe6](https://proandroiddev.com/infinite-lists-with-paging-3-in-jetpack-compose-b095533aefe6)
<br/>
<br/>
An example of using Rick & Morty API with GraphQL in Kotlin Multiplatform Mobile shared code.
[https://github.com/joreilly/MortyComposeKMM](https://github.com/joreilly/MortyComposeKMM)


