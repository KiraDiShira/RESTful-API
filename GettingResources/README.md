- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Getting Resources

- [Structuring Our Outer Facing Contract](#structuring-our-outer-facing-contract)
- [Working with routing](#working-with-routing)
- [Interatcting with resources through HTTP methods](#interatcting-with-resources-through-http-methods)
- [Outer Facing Model vs. Entity Model](#outer-facing-model-vs-entity-model)
- [The Importance of Status Codes](#the-importance-of-status-codes)
- [Handling faults](#handling-faults)
- [Example: Putting all together](#example-putting-all-together)
- [Working with Parent/Child Relationships](#working-with-parentchild-relationships)
- [Formatters and Content Negotiation](#formatters-and-content-negotiation)

## Structuring Our Outer Facing Contract

The outer facing contract consists of three big concepts a consumer of an API uses to interact with that API.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr1.PNG" />

Method Definitions: https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html

When creating a resource, the HTTP response will contain a resource representation in its body. The format of those representations is what **media types** are used for, like application JSON. 

The uniform interface constraint does cover the fact that resources are identified by URIs. Each resource has its own URI, but as far as naming of resources is concerned, there isn't a standard that describes that, or at least not unless you want to dive into OData. There are, however, best practices for this.

A resource name in a URI should always be a noun. In other words, a RESTful URI should refer to a resource that is a thing, instead of referring to an action. So we shouldn't create a *getauthors* resource, that's an action. We should create an *authors* resource, that's a thing conveyed by a noun, and use the *GET* method to get it. To get one specific author then, we'd append it with a forward slash and the authorId. Using these nouns conveys meaning, it describes the resource, so we shouldn't call a resource orders, when it's in fact about authors. That principle should be followed throughout the API for predictability. It helps a consumer understand the API structure. 

If we have another non-hierarchical resource, say employees, we shouldn't name it api/something/something/employees, we should name it api/employees. A single employee then shouldn't be named id/employees, it should be named employees/{employeeId}. This helps keep the API contract predictable and consistent. 

There's quite a bit of a debate going on on whether or not we should pluralize these nouns. I prefer to pluralize them as it helps to convey meaning. When I see an authors resource, that tells us it's a collection of authors and not one author, but good APIs that don't pluralize nouns exist as well. If you prefer that you can, but do make sure to stay consistent. Either all resources should be pluralized nouns, or singular nouns, and not a mix. 

Another important thing you'd want to represent in an API contract is the hierarchy. Our data or models have structure. For example, an author has books that should be represented in the API contract. So if you want to define an author's books, where the books in the model hierarchy are children of an author, we should represent them as  `api/authors/{authorId}/books `. A single book should then be followed by the bookId. 

APIs often expose additional capabilities. Later on we'll learn about filtering and ordering resources, those parameters should be passed through the credit string, they aren't resources in their own right. So we shouldn't write something along the lines of  `api/authors/orderby/name `. There's a few contract smells in that URI. A plural noun should be followed by an Id, and not by another word, and orderby isn't a noun, and a URI like this would mean we'd have defined three different resources - authors, authors/orderby, and authors/orderby/name. So  `api/authors?orderby=name` is a better fit. 

And with that we've already covered a lot, but there is an exception. There always has to be an exception, life would be too easy without them. Sometimes there's these remote procedure calls, style-like calls, like calculate total, that don't easily map to resources. Most RPC-style like calls do map to resources, as we've just proven, but what if we need to calculate, say, the total amount of pages an author wrote? It's not that easy to create a resource from that using pluralized nouns. You'd end up with something like `api/authors/{authorId}/pagetotals`, and then what? Because we'd expect this to return a collection and not a number. You could go for something else, like `api/authorpagetotals/{Id}`, where the backend would then have to map that Id to an authorId, or it can even be the same Id, and that would work, and it does fit the URI design guidelines, but it does feel a bit out of place so this is one of these examples, or one of these exceptional cases, where I'd suggest to take a bit of a pragmatic approach, `api/authors/{authorId}/totalamountofpages`. It isn't according to these best practices, but as long as it's an exceptional case, it doesn't mean you've suddenly got a bad API. Remember there isn't standards for following for these naming guidelines, these are just guidelines.

So by following these simple rules we'll end up with a good resource URI design. But there's one more thing we need to talk about, Ids. Should these be integers? Typically auto-numbered from the database, or should they be GUIDs? In fact, REST stops at that outer facing contract. The layers underneath, including the data store, are of no importance to REST, so getting an author might mean that you're actually fetching data from three different data stores, including some fields from Active Directory, to compose that author resource representation. So it's of no importance, our resource isn't the same as what is in the backend store. These are two different concepts. But from that follows the question, what should we use as identifiers? REST is unrelated to the backend data store, yet often you'll see APIs that actually use the auto-numbered primary key Ids from the database. If the backend doesn't matter, what happens to the resource URIs if you change the backend? The resource URI should remain the same, but if resources are identified by their database auto-numbered fields, and we switch out our current SQL Server to a backend that uses another type of auto-number sequence like MongoDB, all of a sudden all our resource Ids can change. We can, of course, work around that on migration, but still, it's a good idea to keep this in mind when designing resource URIs. And there's a solution for this. GUIDs, unique and unguessable values you can use as primary key in every database. From that we can then switch out datastore technologies and our resource URIs will stay the same. We're also no longer potentially exposing implementation details, as those GUIDs don't give anything away about the underlying technology. They work with all of them, so that's why we'll use GUIDs in this course. This advantage, in my book, is readability. As a developer it's not that convenient to type over a GUID to test an API call, but that's what testing tools are for, and for the end product it shouldn't matter. It's not users like you and me who tend to talk to an API, say for during development, its other pieces of code. And for those it really doesn't matter if the URI contains a hard-to-type GUID. And with that it's time to start implementing this contract.

## Working with routing

Routing matches request URI to an action on a controller. There are two ways: convention-based and attribute-based.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr2.PNG" />

Use attribute at controller and action level: [Route], [HttpGet] ...

## Interatcting with resources through HTTP methods

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr3.PNG" />

**HEAD** is identical to GET with the notable difference that the API shouldn't return a response body, so no response payload. It can be used to obtain information on the resource like testing it for validity, for example, to test if a resource exists. 

**OPTIONS** represents a request for information about the communication options available on that URI. So in other words, OPTIONS will tell us whether or not we can GET the resource, POST it, DELETE it and so on. These OPTIONS are typically in the response headers and not in the body, so no response payload.

## Outer Facing Model vs. Entity Model

The outer facing model does only represents the resources that are sent over the wire in a specific format

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr4.PNG" />

An author is stored in our database with a DateOfBirth, but that DateOfBirth, well that might not be what we want to offer up to the consumers of the API. They might be better off with the age. Another example might be concatenation. Concatenating the FirstName and LastName from an entity into one name field in the resource representation, and sometimes data might come from different places. An author could have a field, say, Royalties, that comes from another API our API must interact with. That alone leads to issues when using entity classes for the outer facing contract, as they don't contain that field. 

Keeping these models separate leads to more robust, reliably evolvable code. Imagine having to change a database table, that would lead to a change of the Entity class. If we're using that same Entity class to directly expose data via the API, our clients might run into problems because they're not expecting an additional renamed or removed field. So when developing, it's fairly important to keep these separate.

## The Importance of Status Codes

Status codes and when to use them are part of the HTTP standard, and it's really important to get these right because these status codes are the only thing a consumer of the API can inspect to know if a request worked out as expected, and if something went wrong, whether it's the fault of the consumer or of the API itself. 

There's **five levels of status codes**. 

**Level 100** status codes are informational and weren't part of the HTTP 1 standard. These are currently not used by APIs. 

The **level 200** status codes mean the request went well. The most common ones are **200 - Ok** for a successful GET request, **201 - Created** for a successful request that resulted in the creation of a new resource, and **204 - NoContent** for the successful request that shouldn't return anything. 

**Level 300** status codes are used for redirection, for example, to tell a search engine a page has permanently moved. Most APIs don't have a need for these. 

Then there's the status codes that tell the consumer he did something wrong, **level 400**, client mistakes. **400 - BadRequest** means a Bad request, the request you, as a consumer of the API, sent to the server is wrong. For example, the JSON the consumer provided can't be parsed. **401 - Unauthorized** means that no,or invalid authentication details, were provided, and **403 - Forbidden** means that authentication succeeded, but the authenticated user doesn't have access to the requested resource. **404 - NotFound** means that the requested resource doesn't exist. Then we have **405 - method not allowed**. This happens when we try to send a request to a resource with an HTTP method that isn't allowed. For example we try to send a POST request to `api/authors`, when only GET is implemented on that resource. **406 - Not acceptable**, and now we're diving into our `presentation media types`. For example a consumer might request the `application/xml` media type, while the API only supports `application/json` and it doesn't provide that as a default representation. **409 - Conflict** then means that there's a conflicted request versus the current state of the resource the request is sent to. This is often used when we're editing a version of a resource that has been renewed since we started editing it, but it's also used when creating a resource, or trying to create a resource that already exists. Two more of importance here, **415 - Unsupported media type**, and that's the other way around from the 406 status codes. Sometimes we have to provide data to our API in the request body, when creating a resource for example, and that data also has a specific media type. If the API doesn't support this, a 415 status code is returned. And lastly, **422 - Unprocessable entity**. It's part of the WebDAV HTTP Extension Standard. This one is typically used for semantic mistakes, and semantic mistakes, well, that's what we get when working with validation. If a validation rule fails, 422 is what's returned. 

And lastly there's **level 500**, server mistakes. Often only **500 - Internal server error** is supported. This means the server made a mistake and the client can't do anything about it, other than trying again later. 

That's a lot of status codes, no reason to learn them by heart, we'll encounter all of them during this course. Some mistakes do happen, and these mistakes are then categorized into two categories, **faults and errors**. Errors are defined as a consumer of the API, like a web app, passing invalid data to the API, and the API correctly rejecting that data. Examples include invalid credentials or incorrect parameters, in other words, these are level 400 status codes and are the result of a client passing incorrect or invalid data. Errors do not contribute to overall API availability. Faults are defined as the API failing to correctly return a response to a valid request by a consumer. In other words, the API made a mistake, so these are level 500 status codes, and these faults, they do contribute to the overall API availability.

## Handling faults

```c#

app.UseExceptionHandler(appBuilder =>
{
    appBuilder.Run(async context =>
    {
        context.Response.StatusCode = 500;
        await context.Response.WriteAsync("An expected fault happened. Try again later.");
    });
});

```

## Example: Putting all together

```c#

[Route("api/authors")]
public class AuthorsController : Controller
{
    private readonly ILibraryRepository _libraryRepository;

    public AuthorsController(ILibraryRepository libraryRepository)
    {
        _libraryRepository = libraryRepository;
    }

    [HttpGet()]
    public IActionResult GetAuthors()
    {
        var authorsFromRepo = _libraryRepository.GetAuthors();

        var authors = Mapper.Map<IEnumerable<AuthorDto>>(authorsFromRepo);

        return Ok(authors);
    }

    [HttpGet("{id}")]
    public IActionResult GetAuthor(Guid id)
    {
        var authorFromRepo = _libraryRepository.GetAuthor(id);
        if (authorFromRepo == null)
        {
            return NotFound();
        }

        var author = Mapper.Map<AuthorDto>(authorFromRepo);

        return Ok(author);
    }
}

```

## Working with Parent/Child Relationships

```c#

[Route("api/authors/{authorId}/books")]
public class BooksController : Controller
{
    private readonly ILibraryRepository _libraryRepository;

    public BooksController(ILibraryRepository libraryRepository)
    {
        _libraryRepository = libraryRepository;
    }

    [HttpGet()]
    public IActionResult GetBooksForAuthor(Guid authorId)
    {
        if (!_libraryRepository.AuthorExists(authorId))
        {
            return NotFound();
        }

        var booksForAuthoreFromRepo = _libraryRepository.GetBooksForAuthor(authorId);

        var booksForAuthor = Mapper.Map<IEnumerable<BookDto>>(booksForAuthoreFromRepo);

        return Ok(booksForAuthor);
    }

    [HttpGet("{id}")]
    public IActionResult GetBookForAuthor(Guid authorId, Guid id)
    {
        if (!_libraryRepository.AuthorExists(authorId))
        {
            return NotFound();
        }

        var bookForAuthoreFromRepo = _libraryRepository.GetBookForAuthor(authorId, id);
        if (bookForAuthoreFromRepo == null)
        {
            return NotFound();
        }
        var bookForAuthor = Mapper.Map<BookDto>(bookForAuthoreFromRepo);

        return Ok(bookForAuthor);
    }
}

```

But let's set this off against one of our constraints, **manipulation of resources through representations**. That's the constraint that's stated that when a client altered a presentation of a resource, including any possible metadata, it must have enough information to modify or delete a resource on the server, provided it has permission to do so, and in the next module we'll allow creating and deleting an author. So if the consumer of the API gets the response we see now, does he have enough information to modify or delete the author? Well, not really, what should be in the response to allow for that, at a minimum, is the resource URI. We already include an Id, and often that's considered enough. From the Id a consumer can create a URI, but if you think about this a bit further, it isn't completely correct. An Id alone isn't what identifies the resource, it's the URI that identifies the resource, and the resource URI is part of the request, but it's not part of the response. So to adhere to this constraint we should include the URI in each representation if update or delete is allowed. We could do that now already, it's just a matter of adding an extra field and filling it up with the URI, but it's also not completely correct, and we're just getting started. There's a much better way of handling this than including the URI, and that's through **HATEOAS**. So we're going to leave this as is currently, and we'll solve this issue once we get to HATEOAS.

## Formatters and Content Negotiation

When we think about a RESTful API these days, we often think of JSON, but as we learned in the previous module, JSON has nothing to do with REST per se, it just happens to be the format of the resource representation, and that implies that a RESTful API can also work with other representation formats, like XML, or a proprietary format. 

So that also means that we have to be able to let our API know what format we want to get back, and that brings us to one of the key concepts when working with HTTP requests and responses, and that's **content negotiation**. That's the process of selecting the best representation for a given response when there are multiple representations available, and it's important. 

If you're only building an API for a specific client, it might be okay to always get back representations in JSON format, but if we're building an API for consumption by multiple clients, some of which we have no control over, chances are that not all of these clients can easily consume JSON representations as a response. Some might be better off with XML or another format, so how is this done? 

Well that's what the **Accept header** of the HTTP request message is for. An API consumer should pass in the `requested media type` like `application/JSON` or `application/xml` through this header. For example, if an Accept header has a value of `application/json`, the consumer states that if our API supports the requested format, it should return that format. If it has a value of `application/xml`, it should return an XML representation, and so on. 

We'll learn later on in the module on HATEOAS and media types that media types like these have a more important role in REST than what we're covering now, but let's not get ahead of ourselves. If no Accept header is available or if it doesn't support the requested format, a lot of APIs default to a default format, like JSON. But I'd suggest to avoid that, especially in the latter case. We could argue that we'd allow our API to serve up responses for requests without an Accept header with the default representation, but if the client requests a specific format, that means he'd also expect to get something back that he can parse. If the API returns its default format, if the requested format is missing, we'd likely end up with a client that cannot parse the response anyway, so in those cases a **406 Not acceptable** status code is warranted. After all, that's what this code was made for. Needless to say that it's also a best practice to always include an Accept header. 

Okay, so now how do we do that in ASP.NET Core? 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr5.PNG" />

By default only the JSON input and output formatters are used. 

If we never passed in an Accept header, we do get back JSON. That's because our API defaults to that. 

If we add application/json as value for the Accept header, which we always should, we also should get back JSON.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr6.PNG" />

So let's send this again, and that is indeed the case. Now let's change this, let's try and send a request with an Accept header of `application/xml`. We still get back a JSON representation. That's already a bit worse, so we actually have two things we want to fix here. First, if the requested media type isn't supported by the API, we shouldn't default to the default type, as the consumer probably won't be able to parse it anyway. So sending a request with an Accept header of `application/xml` and returning JSON, that's not okay. And the second thing we do want to fix is that we want to ensure we do support XML. 

```c#

services.AddMvc(setupAction =>
{
    setupAction.ReturnHttpNotAcceptable = true; 
    setupAction.OutputFormatters.Add(new XmlDataContractSerializerOutputFormatter());
});

```

On the setupAction we can find a list of **OutputFormatters**. To this list we can add a new one. By the way, if you're wondering how ASP.NET Core chooses its default formatter if no Accept header is added to the request, it's always the first one in this list. So by manipulating this list, for example by adding a formatter at the end, inserting one in the beginning, or removing a formatter, we can define what types of output formatter our API supports, and what the default output format is. 
