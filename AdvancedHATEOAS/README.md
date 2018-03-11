- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Advanced HATEOAS, Media Types, and Versioning 

- [HATEOAS and Content Negotiation](#hateoas-and-content-negotiation)
- [Working Towards Self-discoverability with a Root Document](#working-towards-self-discoverability-with-a-root-document)
- [Revisiting Media Types](#revisiting-media-types)
- [Versioning in a RESTful World](#versioning-in-a-restful-world)

## HATEOAS and Content Negotiation

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/AdvancedHATEOAS/Images/ah1.PNG" />

Onscreen is part of a response from a GET request to the author's resource. We've got two fields, value containing the authors and links containing a self-link, which you see onscreen. The links are part of the resource body. 

That begs the question, **Is this really the correct place to put these links?**

If we think back about pagination, we talked about where the pagination information should go. We concluded that it's metadata, so it should be in the header. It describes the resource. And that's true for fields like total count, current page, and so on. But what about those next page and previous page links? We put the paging links in the response body and kept the rest of the paging information as metadata in the header. So are these links part of the resource or do they describe the resource, i.e. are they metadata? Same goes for the other links, links to self and so on. 

If it's metadata describing the resource, they should go in the header. If they are explicit parts of the resource, they can go in the response body. What we're actually dealing with here are two different representations of the resource.

With content negotiation, we ask through the accept header to get back a JSON representation of the resource. We ask for media type application/json, but what we return isn't a representation of the author's resource. It's something else. Additional semantics, the links, on top of the JSON representation. So the links should not be part of the resource when we ask for media type application/json. We've effectively created another media type which were wrongly returning when asking for application/json. So by doing this, we're breaking the self-descriptive message of constraint, which states each message must include enough info to describe how to process the message. 

If we return a representation with links with an accept header of application/json, we're not only returning the wrong representation, the response will have a content-type header with value application/json, which does not match the content of the response. So the consumer of the API does not know how to interpret the response judging from the content type. In other words, we also don't tell the consumer how to process it.

The solution is creating a new media type. So how do we do that? Well, there's a format for this, a vendor-specific media type. You can see an example of that:

```
application/vnd.marvin.hateoas+json
```
First part after application is vnd, the vendor prefix. That's a reserve principle has to begin the mime type with. It indicates that this is a content type that's vendor specific. It's then followed by a custom identifier of the vendor and possibly additional values. A good one to use in our case would be vnd.marvin.hateoas, as you see onscreen. Vnd plus my company, which happens to be called Marvin, the paranoid android from Douglas Adams' Hitchiker's Guide to the Galaxy books and then followed by HATEOAS, stating we want to include those resources links. Then we add a plus sign and JSON. 

What we're actually stating here is that we want a resource representation in JSON with HATEOAS links. If that new media type is requested, the links should be included. The consumer must know about this media type and how to process it. That's what documentation is for. If this media type isn't requested, so we simply request application/json, the links should not be included. 

```c#
    services.AddMvc(setupAction =>
    {
        ...

        var jsonOutputFormatter = setupAction.OutputFormatters
            .OfType<JsonOutputFormatter>().FirstOrDefault();

        if (jsonOutputFormatter != null)
        {
            jsonOutputFormatter.SupportedMediaTypes.Add("application/vnd.marvin.hateoas+json");
        }
    })
    
    
 [HttpGet(Name = "GetAuthors")]
public IActionResult GetAuthors(AuthorsResourceParameters authorsResourceParameters,
    [FromHeader(Name = "Accept")] string mediaType)
{
    if (!_propertyMappingService.ValidMappingExistsFor<AuthorDto, Author>
       (authorsResourceParameters.OrderBy))
    {
        return BadRequest();
    }

    if (!_typeHelperService.TypeHasProperties<AuthorDto>
        (authorsResourceParameters.Fields))
    {
        return BadRequest();
    }

    var authorsFromRepo = _libraryRepository.GetAuthors(authorsResourceParameters);

    var authors = Mapper.Map<IEnumerable<AuthorDto>>(authorsFromRepo);

    if (mediaType == "application/vnd.marvin.hateoas+json")
    {
        var paginationMetadata = new
        {
            totalCount = authorsFromRepo.TotalCount,
            pageSize = authorsFromRepo.PageSize,
            currentPage = authorsFromRepo.CurrentPage,
            totalPages = authorsFromRepo.TotalPages,
        };

        Response.Headers.Add("X-Pagination",
            Newtonsoft.Json.JsonConvert.SerializeObject(paginationMetadata));

        var links = CreateLinksForAuthors(authorsResourceParameters,
            authorsFromRepo.HasNext, authorsFromRepo.HasPrevious);

        var shapedAuthors = authors.ShapeData(authorsResourceParameters.Fields);

        var shapedAuthorsWithLinks = shapedAuthors.Select(author =>
        {
            var authorAsDictionary = author as IDictionary<string, object>;
            var authorLinks = CreateLinksForAuthor(
                (Guid)authorAsDictionary["Id"], authorsResourceParameters.Fields);

            authorAsDictionary.Add("links", authorLinks);

            return authorAsDictionary;
        });

        var linkedCollectionResource = new
        {
            value = shapedAuthorsWithLinks,
            links = links
        };

        return Ok(linkedCollectionResource);
    }
    else
    {
        var previousPageLink = authorsFromRepo.HasPrevious ?
            CreateAuthorsResourceUri(authorsResourceParameters,
            ResourceUriType.PreviousPage) : null;

        var nextPageLink = authorsFromRepo.HasNext ?
            CreateAuthorsResourceUri(authorsResourceParameters,
            ResourceUriType.NextPage) : null;

        var paginationMetadata = new
        {
            previousPageLink = previousPageLink,
            nextPageLink = nextPageLink,
            totalCount = authorsFromRepo.TotalCount,
            pageSize = authorsFromRepo.PageSize,
            currentPage = authorsFromRepo.CurrentPage,
            totalPages = authorsFromRepo.TotalPages
        };

        Response.Headers.Add("X-Pagination",
            Newtonsoft.Json.JsonConvert.SerializeObject(paginationMetadata));

        return Ok(authors.ShapeData(authorsResourceParameters.Fields));
    }
}

```
```
GET ---> http://localhost:6058/api/authors
Headers: Accept: application/json

[
    {
        "id": "f74d6899-9ed2-4137-9876-66b070553f8f",
        "name": "Douglas Adams",
        "age": 65,
        "genre": "Science fiction"
    },
    {
        "id": "76053df4-6687-4353-8937-b45556748abe",
        "name": "George RR Martin",
        "age": 69,
        "genre": "Fantasy"
    },
    {
        "id": "a1da1d8e-1988-4634-b538-a01709477b77",
        "name": "Jens Lapidus",
        "age": 43,
        "genre": "Thriller"
    },
    {
        "id": "412c3012-d891-4f5e-9613-ff7aa63e6bb3",
        "name": "Neil Gaiman",
        "age": 57,
        "genre": "Fantasy"
    },
    {
        "id": "25320c5e-f58a-4b1f-b63a-8ee07a840bdf",
        "name": "Stephen King",
        "age": 70,
        "genre": "Horror"
    },
    {
        "id": "578359b7-1967-41d6-8b87-64ab7605587e",
        "name": "Tom Lanoye",
        "age": 59,
        "genre": "Various"
    }
]

Response header: x-pagination →{"previousPageLink":null,"nextPageLink":null,"totalCount":6,"pageSize":10,"currentPage":1,"totalPages":1}
```

```
GET ---> http://localhost:6058/api/authors
Headers: Accept: application/vnd.marvin.hateoas+json

{
    "value": [
        {
            "id": "f74d6899-9ed2-4137-9876-66b070553f8f",
            "name": "Douglas Adams",
            "age": 65,
            "genre": "Science fiction",
            "links": [
                {
                    "href": "http://localhost:6058/api/authors/f74d6899-9ed2-4137-9876-66b070553f8f",
                    "rel": "self",
                    "method": "GET"
                },
                {
                    "href": "http://localhost:6058/api/authors/f74d6899-9ed2-4137-9876-66b070553f8f",
                    "rel": "delete_author",
                    "method": "DELETE"
                },
                {
                    "href": "http://localhost:6058/api/authors/f74d6899-9ed2-4137-9876-66b070553f8f/books",
                    "rel": "create_book_for_author",
                    "method": "POST"
                },
                {
                    "href": "http://localhost:6058/api/authors/f74d6899-9ed2-4137-9876-66b070553f8f/books",
                    "rel": "books",
                    "method": "GET"
                }
            ]
        },
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
        },
        {
            "id": "a1da1d8e-1988-4634-b538-a01709477b77",
            "name": "Jens Lapidus",
            "age": 43,
            "genre": "Thriller",
            "links": [
                {
                    "href": "http://localhost:6058/api/authors/a1da1d8e-1988-4634-b538-a01709477b77",
                    "rel": "self",
                    "method": "GET"
                },
                {
                    "href": "http://localhost:6058/api/authors/a1da1d8e-1988-4634-b538-a01709477b77",
                    "rel": "delete_author",
                    "method": "DELETE"
                },
                {
                    "href": "http://localhost:6058/api/authors/a1da1d8e-1988-4634-b538-a01709477b77/books",
                    "rel": "create_book_for_author",
                    "method": "POST"
                },
                {
                    "href": "http://localhost:6058/api/authors/a1da1d8e-1988-4634-b538-a01709477b77/books",
                    "rel": "books",
                    "method": "GET"
                }
            ]
        },
        {
            "id": "412c3012-d891-4f5e-9613-ff7aa63e6bb3",
            "name": "Neil Gaiman",
            "age": 57,
            "genre": "Fantasy",
            "links": [
                {
                    "href": "http://localhost:6058/api/authors/412c3012-d891-4f5e-9613-ff7aa63e6bb3",
                    "rel": "self",
                    "method": "GET"
                },
                {
                    "href": "http://localhost:6058/api/authors/412c3012-d891-4f5e-9613-ff7aa63e6bb3",
                    "rel": "delete_author",
                    "method": "DELETE"
                },
                {
                    "href": "http://localhost:6058/api/authors/412c3012-d891-4f5e-9613-ff7aa63e6bb3/books",
                    "rel": "create_book_for_author",
                    "method": "POST"
                },
                {
                    "href": "http://localhost:6058/api/authors/412c3012-d891-4f5e-9613-ff7aa63e6bb3/books",
                    "rel": "books",
                    "method": "GET"
                }
            ]
        },
        {
            "id": "25320c5e-f58a-4b1f-b63a-8ee07a840bdf",
            "name": "Stephen King",
            "age": 70,
            "genre": "Horror",
            "links": [
                {
                    "href": "http://localhost:6058/api/authors/25320c5e-f58a-4b1f-b63a-8ee07a840bdf",
                    "rel": "self",
                    "method": "GET"
                },
                {
                    "href": "http://localhost:6058/api/authors/25320c5e-f58a-4b1f-b63a-8ee07a840bdf",
                    "rel": "delete_author",
                    "method": "DELETE"
                },
                {
                    "href": "http://localhost:6058/api/authors/25320c5e-f58a-4b1f-b63a-8ee07a840bdf/books",
                    "rel": "create_book_for_author",
                    "method": "POST"
                },
                {
                    "href": "http://localhost:6058/api/authors/25320c5e-f58a-4b1f-b63a-8ee07a840bdf/books",
                    "rel": "books",
                    "method": "GET"
                }
            ]
        },
        {
            "id": "578359b7-1967-41d6-8b87-64ab7605587e",
            "name": "Tom Lanoye",
            "age": 59,
            "genre": "Various",
            "links": [
                {
                    "href": "http://localhost:6058/api/authors/578359b7-1967-41d6-8b87-64ab7605587e",
                    "rel": "self",
                    "method": "GET"
                },
                {
                    "href": "http://localhost:6058/api/authors/578359b7-1967-41d6-8b87-64ab7605587e",
                    "rel": "delete_author",
                    "method": "DELETE"
                },
                {
                    "href": "http://localhost:6058/api/authors/578359b7-1967-41d6-8b87-64ab7605587e/books",
                    "rel": "create_book_for_author",
                    "method": "POST"
                },
                {
                    "href": "http://localhost:6058/api/authors/578359b7-1967-41d6-8b87-64ab7605587e/books",
                    "rel": "books",
                    "method": "GET"
                }
            ]
        }
    ],
    "links": [
        {
            "href": "http://localhost:6058/api/authors?orderBy=Name&pageNumber=1&pageSize=10",
            "rel": "self",
            "method": "GET"
        }
    ]
}

Response header: x-pagination →{"totalCount":6,"pageSize":10,"currentPage":1,"totalPages":1}
```

## Working Towards Self-discoverability with a Root Document

```c#
[Route("api")]
public class RootController : Controller
{
    private IUrlHelper _urlHelper;

    public RootController(IUrlHelper urlHelper)
    {
        _urlHelper = urlHelper;
    }

    [HttpGet(Name = "GetRoot")]
    public IActionResult GetRoot([FromHeader(Name = "Accept")] string mediaType)
    {
        if (mediaType == "application/vnd.marvin.hateoas+json")
        {
            var links = new List<LinkDto>();

            links.Add(
                new LinkDto(_urlHelper.Link("GetRoot", new { }),
                    "self",
                    "GET"));

            links.Add(
                new LinkDto(_urlHelper.Link("GetAuthors", new { }),
                    "authors",
                    "GET"));

            links.Add(
                new LinkDto(_urlHelper.Link("CreateAuthor", new { }),
                    "create_author",
                    "POST"));

            return Ok(links);
        }

        return NoContent();
    }
}
```

```
GET ---> http://localhost:6058/api/
Headers: Accept: application/json

(204 - no content)
```

```
GET ---> http://localhost:6058/api/
Headers: Accept: application/vnd.marvin.hateoas+json

(200 - OK)

[
    {
        "href": "http://localhost:6058/api",
        "rel": "self",
        "method": "GET"
    },
    {
        "href": "http://localhost:6058/api/authors",
        "rel": "authors",
        "method": "GET"
    },
    {
        "href": "http://localhost:6058/api/authors",
        "rel": "create_author",
        "method": "POST"
    }
]
```

## Revisiting Media Types

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/AdvancedHATEOAS/Images/ah2.PNG" />

On screen, we see part of the response body for a book, requested with our new media type in the Accept header. There's two issues with this, first the format on the links. This format isn't a standard, it's something we're cooking up. The client will have to know how to interpret this format. And it learns that from the API documentation where we describe the custom media type, and where we describe Rel Values.

As we learned before, media types are also part of our outer facing contract. And then there's another issue. Hyper media allows the application controls, the links, to be supplied on demand. So, say we have a link to update the book, as we see on screen. How does the consumer know what exactly to send as far as the representation goes? We touched upon media types before, and we learned how to create our own media types. It defines the presentation of a resource. It's a central principle in the RESTful world. And we actually already have a great example we can use to explain this. 

Remember that we have different representations for getting an author and creating an author?

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/AdvancedHATEOAS/Images/ah3.PNG" />

When we get one, we get back an id, a name, an age, and a genre. To create one, you must pass in a first name and last name, a date of birth, and a genre, but no id. We just used application/json as media types for both of these, as most APIs do. But, in fact, these are different representations of the same author's resource. Using application/json is actually a mistake. That just tells us we want data format to be JSON. But we need to be more specific. 

So how do we correct this? By using media types. Instead of using application/json, you could create vendor-specific media types for these representations. An author representation to get can then be represented with media type author.friendly, for a friendlier format with age and name. And alt for the presentation to create can then be represented with media type author.full. And we can go even further. We could support different representations when we get an author. Pass in the author.full media type, would return the author with first name, last name and date of birth. We just learned how to work with these media types in the previous two demos. So, we already know how to support this. And this almost automatically brings us to versioning. After all, we are talking about an evolvable API. And that includes changes in representations.

## Versioning in a RESTful World

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/AdvancedHATEOAS/Images/ah4.PNG" />
