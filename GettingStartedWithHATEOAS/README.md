- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Getting Started with HATEOAS 

- [Hypermedia as the Engine of Application State](#hypermedia-as-the-engine-of-application-state)
- [Supporting HATOEAS](#supporting-hatoeas)
- [Statically typed approach](#statically-typed-approach)
- [Dynamically typed approach](#dynamically-typed-approach)

## Hypermedia as the Engine of Application State

Currently, our APIs already quite good but it's not exactly evolvable nor is it RESTful. HATEOAS will help with this. It makes APIs evolvable and even self-describing.

Hypermedia like links drives how to consume and use the API. It tells the consumer how it can interact with the API. Can I delete this resource? Can I update it? How can I create it? And how can I access the next page of data? And so on. 

But, what problem does it solve? Well, imagine we want to allow the consumer of the API to update or delete a book. Currently it does not know how to do that by looking at the response body. It has to have some intrinsic knowledge of how our API contract is structured. From contractor documentation, the consumer can learn that he'd have to send a book or patch request to `API/authors/{author ID}/books/{book ID}`, to update a book with a specific payload. And delete request to delete it. But it cannot get this information from the response itself. The consumer has to know this in advance. 

Now imagine another case. Let's say we have an additional field, NumberOfCopiesInLibrary on our book. If there are still copies in our library, somebody can reserve a copy. This is obviously an optimistic example, set in the hopefully near future, as "The Winds of Winter" hasn't been released yet. Anyway, in this case, the consumer of the API will have to know it has to inspect that field before it can send a post request to, say, api/bookReservations'. And that's again intrinsic knowledge of how the API works. But say this changes. Say an additional rule is implemented. There have to be copies left, but if the user wants to make a reservation for a book that's marked with content mature, he must be older than 16. This rule will effectively break the consumers of the API. They have to implement that on their side as well. In short, if an API doesn't support HATEOAS the consumers of the API have to know too much. And if the API evolves, this might break the consuming applications because the assumptions made by those applications can become invalid. And that's the issue that HATEOAS solves. A lot of these assumptions can be thrown overboard by including links in the response that tell the consumer which action on the API is possible. And those links, well, that's hypermedia. So HATEOAS allows the API to truly evolve without breaking consuming applications. And that in the end results in self-discoverable APIs. 

So in our example, we could add an additional property, links to the response. And the client would just have to inspect these links. 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh1.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh2.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh3.PNG" />

And if a reservation link appears, a reservation can be made for this book. So it's up to the server to decide whether or not to show this link. The consumer needs no knowledge about that business rule. And moreover, the rule can change without having to redo anything at the client level. If all of a sudden we expand the rule, stating that there must be copies left and that the user must be over 16 to reserve a book marked with content mature, this is no longer something that has to be checked by the client. The server would simply not include that link to make the reservation and the client only has to inspect the link to say, show a button to make the reservation. 

Let's have a look at another Roy Fielding quote, the guy who invented REST. 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh4.PNG" />

If we match this to building applications, we're almost talking about self-generating client user interfaces. Most apps don't go that far. But, as the example we used teaches us, for things like rules that change, this is pretty great. And, additional pieces of functionality can be added. For example, marking a book as one of your favorite books. In that case, an additional link will be provided and this will not break existing client applications. But, from that moment on, they can implement this functionality starting from that link. We'll get back to this sample later on. 

If you think about all of this, this is actually nothing more than how we should work with the HTTP protocol. As we know, RESTy folks, any much of how a good web app should work. Well, on the app, it doesn't really matter if a link changes, we start at De Tijd, where I have a newspaper site for example. And from that we navigate to an article by clicking a link or submitting a comment to a forum. And that's two examples of hypermedia driving application state. And, if the server decides that the link should change, well, to request that we turn to route page, we'll contain that new link. The browser, which is our application in this context, does not break because that link changed. Instead, it's the hypermedia in the response that's used by the browser to show us what we can do next. 

Okay, now what do these links look like? JSON or XML don't have a notion on how to represent links. But HTML does, the anchor element. In HTML, href contains the URI, rel describes how the link relates to the resource, and type, which is optional, describes the media type. 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh5.PNG" />

For supporting HATEOAS, this format is often expanded upon. Let's have a look at one of the links from our previous example again. They all follow the same principles. The method property defines the method to use. Rel defines a type of action. This is what clients look for in the links list. Href contains the URI to be invoked to execute this action. 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh6.PNG" />

The client simply uses this link, he doesn't have to create it anymore. Mind you, HATEOAS does not completely lift a client of having to have some knowledge of what to expect. It still needs to know about the link types that can come back. In other words, the rel values and whether or not it wants to use them. But if we're no longer hard coding URIs and assumptions in the client, a URI or assumption change no longer breaks the client. 

Now, if this were a collection resource like our authors resource, this is also where we'd include the pagination links. So they're no longer in the pagination header, they're now in the links area on our response.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh7.PNG" />

For collection resources, we will need some sort of envelope, an object to hold the value, which has the list of authors and the links. Were we to just add an array of authors and a links property, which is an array of links, we'd have invalid JSON.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh8.PNG" />

And this might be the moment when you're thinking, Wait a minute, this does not make sense. When we learned about paging, we learned that we couldn't use an object like this because that would break the self-descriptive message constraint. After all, when requesting the authors with media type application JSON, the representation should be an area of authors. This isn't truly RESTful, but that can be fixed. We're covering that next. For now, let's see how we can implement this in the next few clips.

## Supporting HATOEAS

In the upcoming demos, we'll change our API so it adheres to the hypermedia as the engine of application state constraint. But, how are we going to implement this? The logic for creating the links can't just be automated, as it will depend on the business rules. For example, a book might contain links to update or delete it but also a link that's supposed to book reservations. Which only appears when some rules are met. We can't just guess what these links might be for each resource. We'll have to code this ourselves. That said, what remains a fact is that we have to add the links to the output. So, there should be a links property on each representation we're sending back to the consumer of the API. Essentially, there's two approaches I often see used for this. And, coincidence or not, we've got a case for both. 

The first one, a statically typed approach, involves working with base and wrapper classes. If you think about our books controller, the get book for alter action returns a list of BookDto. We can ensure that that BookDto class inherits a class that contains links. So they can be serialized. And get books for author returns a list of books, so that's an action for which we'll have to wrap the results in a containing class so we can include the links. Second approach is a dynamically typed approach. It involves working with anonymous types and dynamics. For example, an ExpandoObject. If you think back about what we did with the get author action on our authors controller, we remember that it no longer works with the author Dto. It works with the ExpandoObject because we shaped the data before we return it. And that's a dynamically typed class. As it's an ExpandoObject, we can add links to it. For collection resources, we can wrap that in anonymous type. So, let's have a look at both approaches.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh9.PNG" />

## Statically typed approach

First we'll work towards supporting HATEOAS using the base and wrapper class approach.

```c#

public class LinkDto
{
    public string Href { get; }
    public string Rel { get; }
    public string Method { get; }

    public LinkDto(string href, string rel, string method)
    {
        Href = href;
        Rel = rel;
        Method = method;
    }
}

public abstract class LinkedResourceBaseDto
{
    public List<LinkDto> Links { get; set; }
        = new List<LinkDto>();
}

public class BookDto : LinkedResourceBaseDto
{
    ...
}

private BookDto CreateLinksForBook(BookDto book)
{
    book.Links.Add(new LinkDto(_urlHelper.Link("GetBookForAuthor",
            new { id = book.Id }),
        "self",
        "GET"));

    book.Links.Add(
        new LinkDto(_urlHelper.Link("DeleteBookForAuthor",
                new { id = book.Id }),
            "delete_book",
            "DELETE"));

    book.Links.Add(
        new LinkDto(_urlHelper.Link("UpdateBookForAuthor",
                new { id = book.Id }),
            "update_book",
            "PUT"));

    book.Links.Add(
        new LinkDto(_urlHelper.Link("PartiallyUpdateBookForAuthor",
                new { id = book.Id }),
            "partially_update_book",
            "PATCH"));

    return book;
}

[HttpGet("{id}", Name = "GetBookForAuthor")]
public IActionResult GetBookForAuthor(Guid authorId, Guid id)
{
   ...

    //return Ok(bookForAuthor);
    return Ok(CreateLinksForBook(bookForAuthor));
}

[HttpPost(Name = "CreateBookForAuthor")]
public IActionResult CreateBookForAuthor(Guid authorId, [FromBody] BookForCreationDto book)
{
    ...

    //return CreatedAtRoute("GetBookForAuthor", 
    //    new { authorId = authorId, id = bookToReturn.Id }, bookToReturn);

    return CreatedAtRoute("GetBookForAuthor",
        new { authorId = authorId, id = bookToReturn.Id },
        CreateLinksForBook(bookToReturn));
}

[HttpGet(Name = "GetBooksForAuthor")]
public IActionResult GetBooksForAuthor(Guid authorId)
{
   ...

    var booksForAuthor = Mapper.Map<IEnumerable<BookDto>>(booksForAuthoreFromRepo);

    booksForAuthor = booksForAuthor.Select(book =>
    {
        book = CreateLinksForBook(book);
        return book;
    });

    return Ok(booksForAuthor);
}

```
```
GET ---> http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books
```

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingStartedWithHATEOAS/Images/gsh10.PNG" />

We also see that for each book, the links are included. So that's great. But what we're not getting back are the actions on the books resource itself. We only have actions available on each separate book. We can't just add a links property to this JSON because it would result in an invalid JSON. So we're going to need a wrapper class. Let's add a new class and name it LinkedCollectionResourceWrapperDto.

```c#
public class LinkedCollectionResourceWrapperDto<T> : LinkedResourceBaseDto
    where T : LinkedResourceBaseDto
{
    public IEnumerable<T> Value { get; set; }

    public LinkedCollectionResourceWrapperDto(IEnumerable<T> value)
    {
        Value = value;
    }
}

private LinkedCollectionResourceWrapperDto<BookDto> CreateLinksForBooks(
    LinkedCollectionResourceWrapperDto<BookDto> booksWrapper)
{
    // link to self
    booksWrapper.Links.Add(
        new LinkDto(_urlHelper.Link("GetBooksForAuthor", new { }),
        "self",
        "GET"));

    return booksWrapper;
}

[HttpGet(Name = "GetBooksForAuthor")]
public IActionResult GetBooksForAuthor(Guid authorId)
{
    ...

    var booksForAuthor = Mapper.Map<IEnumerable<BookDto>>(booksForAuthorFromRepo);

    booksForAuthor = booksForAuthor.Select(book =>
    {
        book = CreateLinksForBook(book);
        return book;
    });

    var wrapper = new LinkedCollectionResourceWrapperDto<BookDto>(booksForAuthor);

    return Ok(CreateLinksForBooks(wrapper));
}

```

```
GET ---> http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books

{
    "value": [
        {
            "id": "09af5a52-9421-44e8-a2bb-a6b9ccbc8239",
            "title": "A Dance with Dragons",
            "description": "A Dance with Dragons is the fifth of seven planned novels in the epic fantasy series A Song of Ice and Fire by American author George R. R. Martin.",
            "authorId": "76053df4-6687-4353-8937-b45556748abe",
            "links": [
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/09af5a52-9421-44e8-a2bb-a6b9ccbc8239",
                    "rel": "self",
                    "method": "GET"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/09af5a52-9421-44e8-a2bb-a6b9ccbc8239",
                    "rel": "delete_book",
                    "method": "DELETE"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/09af5a52-9421-44e8-a2bb-a6b9ccbc8239",
                    "rel": "update_book",
                    "method": "PUT"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/09af5a52-9421-44e8-a2bb-a6b9ccbc8239",
                    "rel": "partially_update_book",
                    "method": "PATCH"
                }
            ]
        },
        {
            "id": "447eb762-95e9-4c31-95e1-b20053fbe215",
            "title": "A Game of Thrones",
            "description": "A Game of Thrones is the first novel in A Song of Ice and Fire, a series of fantasy novels by American author George R. R. Martin. It was first published on August 1, 1996.",
            "authorId": "76053df4-6687-4353-8937-b45556748abe",
            "links": [
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/447eb762-95e9-4c31-95e1-b20053fbe215",
                    "rel": "self",
                    "method": "GET"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/447eb762-95e9-4c31-95e1-b20053fbe215",
                    "rel": "delete_book",
                    "method": "DELETE"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/447eb762-95e9-4c31-95e1-b20053fbe215",
                    "rel": "update_book",
                    "method": "PUT"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/447eb762-95e9-4c31-95e1-b20053fbe215",
                    "rel": "partially_update_book",
                    "method": "PATCH"
                }
            ]
        },
        {
            "id": "bc4c35c3-3857-4250-9449-155fcf5109ec",
            "title": "The Winds of Winter",
            "description": "Forthcoming 6th novel in A Song of Ice and Fire.",
            "authorId": "76053df4-6687-4353-8937-b45556748abe",
            "links": [
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/bc4c35c3-3857-4250-9449-155fcf5109ec",
                    "rel": "self",
                    "method": "GET"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/bc4c35c3-3857-4250-9449-155fcf5109ec",
                    "rel": "delete_book",
                    "method": "DELETE"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/bc4c35c3-3857-4250-9449-155fcf5109ec",
                    "rel": "update_book",
                    "method": "PUT"
                },
                {
                    "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books/bc4c35c3-3857-4250-9449-155fcf5109ec",
                    "rel": "partially_update_book",
                    "method": "PATCH"
                }
            ]
        }
    ],
    "links": [
        {
            "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books",
            "rel": "self",
            "method": "GET"
        }
    ]
}
```
## Dynamically typed approach

```c#
private IEnumerable<LinkDto> CreateLinksForAuthor(Guid id, string fields)
{
    var links = new List<LinkDto>();

    if (string.IsNullOrWhiteSpace(fields))
    {
        links.Add(
            new LinkDto(_urlHelper.Link("GetAuthor", new { id = id }),
                "self",
                "GET"));
    }
    else
    {
        links.Add(
            new LinkDto(_urlHelper.Link("GetAuthor", new { id = id, fields = fields }),
                "self",
                "GET"));
    }

    links.Add(
        new LinkDto(_urlHelper.Link("DeleteAuthor", new { id = id }),
            "delete_author",
            "DELETE"));

    links.Add(
        new LinkDto(_urlHelper.Link("CreateBookForAuthor", new { authorId = id }),
            "create_book_for_author",
            "POST"));

    links.Add(
        new LinkDto(_urlHelper.Link("GetBooksForAuthor", new { authorId = id }),
            "books",
            "GET"));

    return links;
}

[HttpGet("{id}", Name = "GetAuthor")]
public IActionResult GetAuthor(Guid id, [FromQuery] string fields)
{
   ...

    var links = CreateLinksForAuthor(id, fields);

    var linkedResourceToReturn = author.ShapeData(fields)
        as IDictionary<string, object>;

    linkedResourceToReturn.Add("links", links);

    return Ok(linkedResourceToReturn);
}

[HttpPost]
public IActionResult CreateAuthor([FromBody] AuthorForCreationDto author)
{
   ...
    var authorToReturn = Mapper.Map<AuthorDto>(authorEntity);

    var links = CreateLinksForAuthor(authorToReturn.Id, null);

    var linkedResourceToReturn = author.ShapeData(null)
        as IDictionary<string, object>;

    linkedResourceToReturn.Add("links", links);

    return CreatedAtRoute("GetAuthor", new {id = linkedResourceToReturn["Id"]}, linkedResourceToReturn);
}

```
```
GET ---> http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe

{
    "id": "76053df4-6687-4353-8937-b45556748abe",
    "name": "George RR Martin",
    "age": 69,
    "genre": "Fantasy",
    "links": [
        {
            "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe",
            "rel": "self",
            "method": "GET"
        },
        {
            "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe",
            "rel": "delete_author",
            "method": "DELETE"
        },
        {
            "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books",
            "rel": "create_book_for_author",
            "method": "POST"
        },
        {
            "href": "http://localhost:6058/api/authors/76053df4-6687-4353-8937-b45556748abe/books",
            "rel": "books",
            "method": "GET"
        }
    ]
}

```

The first thing we want to do is add a method to create the links for the authors resource.
