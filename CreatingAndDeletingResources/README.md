# Creating and Deleting Resources

## Method Safety and Method Idempotency

The outer facing contract consists of three big concepts a consumer of an API uses to interact with that API.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/CreatingAndDeletingResources/Images/Cadr1.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/CreatingAndDeletingResources/Images/Cadr2.PNG" />

We're building a RESTful API that uses these HTTP methods. It's a standard, so we should correctly implement it so other components we might use can rely on this being correctly implemented. This table here will help us decide what we should use to implement the functionality we'll want from our API. So if you ever wonder what method to use for which use case, this is a table you'll want to go back to and check.

**Why is PUT idempotent?**

When you PUT a resource, these two assumptions are in play:

1. You are referring to an entity, not to a collection.

2. The entity you are supplying is complete (the entire entity).

Let's look at one of your examples.
```
{ "username": "skwee357", "email": "skwee357@domain.com" }
```

If you POST this document to /users, as you suggest, then you might get back an entity such as

```
.. /users/1

{
    "username": "skwee357",
    "email": "skwee357@domain.com"
}
```
If you want to modify this entity later, you choose between PUT and PATCH. A PUT might look like this:
```
PUT /users/1
{
    "username": "skwee357",
    "email": "skwee357@gmail.com"       // new email address
}
```
You can accomplish the same using PATCH. That might look like this:
```
PATCH /users/1
{
    "email": "skwee357@gmail.com"       // new email address
}
```
You'll notice a difference right away between these two. The PUT included all of the parameters on this user, but PATCH only included the one that was being modified (email).

When using PUT, it is assumed that you are sending the complete entity, and that complete entity replaces any existing entity at that URI. In the above example, the PUT and PATCH accomplish the same goal: they both change this user's email address. But PUT handles it by replacing the entire entity, while PATCH only updates the fields that were supplied, leaving the others alone.

Since PUT requests include the entire entity, if you issue the same request repeatedly, it should always have the same outcome (the data you sent is now the entire data of the entity). Therefore PUT is idempotent.

**Using PUT wrong**
What happens if you use the above PATCH data in a PUT request?
```
GET /users/1
{
    "username": "skwee357",
    "email": "skwee357@domain.com"
}
PUT /users/1
{
    "email": "skwee357@gmail.com"       // new email address
}

GET /users/1
{
    "email": "skwee357@gmail.com"      // new email address... and nothing else!
}
```
(I'm assuming for the purposes of this question that the server doesn't have any specific required fields, and would allow this to happen... that may not be the case in reality.)

Since we used PUT, but only supplied email, now that's the only thing in this entity. This has resulted in data loss.

This example is here for illustrative purposes -- don't ever actually do this. This PUT request is technically idempotent, but that doesn't mean it isn't a terrible, broken idea.

**How can PATCH be idempotent?**
In the above example, PATCH was idempotent. You made a change, but if you made the same change again and again, it would always give back the same result: you changed the email address to the new value.
```
GET /users/1
{
    "username": "skwee357",
    "email": "skwee357@domain.com"
}
PATCH /users/1
{
    "email": "skwee357@gmail.com"       // new email address
}

GET /users/1
{
    "username": "skwee357",
    "email": "skwee357@gmail.com"       // email address was changed
}
PATCH /users/1
{
    "email": "skwee357@gmail.com"       // new email address... again
}

GET /users/1
{
    "username": "skwee357",
    "email": "skwee357@gmail.com"       // nothing changed since last GET
}
```

**So when is PATCH not idempotent, then?**

To show why PATCH isn't idempotent, it helps to start with the definition of idempotence (from Wikipedia):

```
The term idempotent is used more comprehensively to describe an operation that will produce the same results if executed once or multiple times [...] An idempotent function is one that has the property f(f(x)) = f(x) for any value x.
```

In more accessible language, an idempotent PATCH could be defined as: After PATCHing a resource with a patch document, all subsequent PATCH calls to the same resource with the same patch document will not change the resource.

Conversely, a non-idempotent operation is one where f(f(x)) != f(x), which for PATCH could be stated as: After PATCHing a resource with a patch document, subsequent PATCH calls to the same resource with the same patch document do change the resource.

To illustrate a non-idempotent PATCH, suppose there is a /users resource, and suppose that calling  GET /users returns a list of users, currently:

```
[{ "id": 1, "username": "firstuser", "email": "firstuser@example.org" }]
```
Rather than PATCHing /users/{id}, as in the OP's example, suppose the server allows PATCHing /users. Let's issue this PATCH request:

```
PATCH /users
[{ "op": "add", "username": "newuser", "email": "newuser@example.org" }]
```
Our patch document instructs the server to add a new user called newuser to the list of users. After calling this the first time, GET /users would return:

```
[{ "id": 1, "username": "firstuser", "email": "firstuser@example.org" },
 { "id": 2, "username": "newuser", "email": "newuser@example.org" }]
 ```
 
Now, if we issue the exact same PATCH request as above, what happens? (For the sake of this example, let's assume that the /users resource allows duplicate usernames.) The "op" is "add", so a new user is added to the list, and a subsequent GET /users returns:

```
[{ "id": 1, "username": "firstuser", "email": "firstuser@example.org" },
 { "id": 2, "username": "newuser", "email": "newuser@example.org" },
 { "id": 3, "username": "newuser", "email": "newuser@example.org" }]
 ```
The /users resource has changed again, even though we issued the exact same PATCH against the exact same endpoint. If our PATCH is f(x), f(f(x)) is not the same as f(x), and therefore, **this particular PATCH is not idempotent.**

Although PATCH isn't guarateed to be idempotent, there's nothing in the PATCH specification to prevent you from making all PATCH operations on your particular server idempotent. RFC 5789 even anticipates advantages from idempotent PATCH requests:

A PATCH request can be issued in such a way as to be idempotent, which also helps prevent bad outcomes from collisions between two PATCH requests on the same resource in a similar time frame.

In Dan's example, his PATCH operation is, in fact, idempotent. In that example, the /users/1 entity changed between our PATCH requests, but not because of our PATCH requests; it was actually the Post Office's different patch document that caused the zip code to change. The Post Office's different PATCH is a different operation; if our PATCH is f(x), the Post Office's PATCH is g(x). Idempotence states that f(f(f(x))) = f(x), but makes no guarantes about f(g(f(x))).

## Create a resource

Then we need to get the inputted author, that'll be provided in the request body. That means we can use the `FromBody` attribute for that. If you use this in our parameter list, we signify that the parameter should be de-serialized from the request body, but what should it be de-serialized to? So we have this `AuthorDto` for output,

```c#
public class AuthorDto
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
    public string Genre { get; set; }
}
```


but for post, we need a Dto for input, and that should be a different type than this AuthorDto. This AuthorDto contains an ID, and we're creating an author in a system for which the responsibility of generating the Id is at level of the API. So the Dto shouldn't contain that Id field. And there's more, we return the age of the author, but in the backend, we store the date of birth, so we do need that date of birth as input. And name, well, we store first name and last name fields, and not a concatenation of those, as is the case in our `AuthorDto` class. So let's create an `author for CreationDto`. 

```c#
public class AuthorForCreationDto
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTimeOffset DateOfBirth { get; set; }
    public string Genre { get; set; }
}
```

Mind you there are system where the properties on the class used for output are exactly the same as those on the class used for input, but even in those cases, I'd suggest to keep these separate. It leads to a model that's more in line with the API functionality, make change already factoring afterwards much easier, and when validation comes into play, you typically want validation on input, but not necessarily on output. So I suggest to use a separate Dto for creating, updating, and returning resources. 

```c#
[HttpGet("{id}", Name = "GetAuthor")]
public IActionResult GetAuthor(Guid id)
{
   ...
}

[HttpPost]
public IActionResult CreateAuthor([FromBody] AuthorForCreationDto author)
{
    if (author == null)
    {
        return BadRequest();
    }

    var authorEntity = Mapper.Map<Author>(author);

    _libraryRepository.AddAuthor(authorEntity);

    if (!_libraryRepository.Save())
    {
        throw new Exception("Creating an author failed on save");
        //return StatusCode(500, "A problem happened with handling your request.");
    }

    var authorToReturn = Mapper.Map<AuthorDto>(authorEntity);

    return CreatedAtRoute("GetAuthor", new {id = authorToReturn.Id}, authorToReturn);
}
```
What if there is a fault persisting entity in the database? We want to return, a 500 internal server error with a generic error message. We only return a generic error message, because the consumer of the API really doesn't need to know what exactly went wrong, it just needs to know that it's not its responsibility. Our generic exception handler will not catch this, as save doesn't throw an exception, but we can use the StatusCode method for that, passing in the StatusCode. The StatusCode is 500, and we provide a generic message. But we already configured the exception handler middleware in the previous module to return a 500 internal server error with a generic message if an unhandled exception occurs. So another option is to throw an exception from the controller, and let the middleware handle it. Now is this a good approach or a bad approach? Well it's, it kind of depends. Throwing exceptions is expensive, it's a performance hit, so that would lead us to returning the StatusCode from the controller as a best practice, but on the other hand, that also means that we'll have code to return 500 internal server errors in different places, on the global level and in the controller itself. At this moment, that's not too much of a problem, but once we start implementing logging, that would also mean we'd want to provide logging code on each StatusCode 500 we return. I've seen both approaches, and there's something to be said for both. In this case, we're going to have the middleware handle all our responses that warrant a 500 internal server error. That'll come in nicely when we need to implement logging for these types of StatusCodes later on. We'll only have to write that code in one place, being at the configuration of the exception handler middleware.

what should we return. In case of a successful post, we should return a 201 created response. For that, we can use the CreatedAtRoute method. This method allows us to return a response with the location header, and that location header, that will then contain the URI where the newly created author can be found. So the first thing we need to pass into this method is the route name that's going to be used for generating the URI. And if we scroll up a bit, we see that it should refer to the action to get a single author, that's our GetAuthor action. So, let's give this a name we can refer to, say GetAuthor, there we go. And we pass in GetAuthor as the first parameter of our CreatedAtRoute method. Now to get an author, we need the author ID. We need to pass that in as a route value, so the correct URI can be generated containing that ID. To do that, we pass in an anonymous type, and we give that anonymous type one field, ID, which is the name used in our route template, and we give it a value of authorToReturn.Id. And lastly, we want to pass in the actual authorToReturn Dto. This one will get serialized into the response body.

if we do a POST with this payload:

```c#
{
	"firstName" : "James",
	"lastName": "Ellroy",
	"dateOfBirth" : "1948-03-04T00:00:00",
	"genre" : "Thriller"
}
```

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/CreatingAndDeletingResources/Images/Cadr3a.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/CreatingAndDeletingResources/Images/Cadr4.PNG" />

## Creating a Child Resource

```c#
public class Book
{
    [Key]       
    public Guid Id { get; set; }

    [Required]
    [MaxLength(100)]
    public string Title { get; set; }

    [MaxLength(500)]
    public string Description { get; set; }

    [ForeignKey("AuthorId")]
    public Author Author { get; set; }

    public Guid AuthorId { get; set; }
}
```

We already know from the previous demo that we don't want the Id in our new class, as the server is responsible for choosing the URI, but what about that AuthorId, there's already an AuthorId in the URI, so if we allow the AuthorId in the payloads, well, we might end up with an issue we want to avoid, and that issue is that a post to the book's resource for author A would create a book for author B. 

```c#
public class BookForCreationDto
{
    public string Title { get; set; }
    public string Description { get; set; }
}

[Route("api/authors/{authorId}/books")]
public class BooksController : Controller
{    
    ...
	
    [HttpPost]
    public IActionResult CreateBookForAuthor(Guid authorId, [FromBody] BookForCreationDto book)
    {
        if (book == null)
        {
            return BadRequest();
        }

        if (!_libraryRepository.AuthorExists(authorId))
        {
            return NotFound();
        }

        var bookEntity = Mapper.Map<Book>(book);

        _libraryRepository.AddBookForAuthor(authorId, bookEntity);

        if (!_libraryRepository.Save())
        {
            throw new Exception($"Creating a book for author {authorId} failed on save");
        }

        var bookToReturn = Mapper.Map<BookDto>(bookEntity);

        return CreatedAtRoute("GetBookForAuthor", new { authorId = authorId, id = bookToReturn.Id }, bookToReturn);
    }
}

```

## Creating Child Resources Together with a Parent Resource

We need to add child resources (Books) to parent (AuthorForCreationDto)

```c#
public class AuthorForCreationDto
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTimeOffset DateOfBirth { get; set; }
    public string Genre { get; set; }

    public ICollection<BookForCreationDto> Books { get; set; } = new List<BookForCreationDto>();
}
```
