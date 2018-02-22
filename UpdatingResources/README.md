- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Updating resources

- [Updating a Resource](#updating-a-resource)
- [Updating Collection Resources](#updating-collection-resources)
- [Upserting](#upserting)
- [Partially Updating a Resource](#partially-updating-a-resource)
- [Upserting with PATCH](#upserting-with-patch)
- [HTTP Method Overview by Use Case](#http-method-overview-by-use-case)

## Updating a Resource 

```c#

[HttpPut("{id}")]
public IActionResult UpdateBookForAuthor(Guid authorId, Guid id, [FromBody] BookForUpdateDto book)
{
    if (book == null)
    {
        return BadRequest();
    }

    if (!_libraryRepository.AuthorExists(authorId))
    {
        return NotFound();
    }

    var bookForAuthorFromRepo = _libraryRepository.GetBookForAuthor(authorId, id);
    if (bookForAuthorFromRepo == null)
    {
        return NotFound();
    }

    Mapper.Map(book, bookForAuthorFromRepo);

    _libraryRepository.UpdateBookForAuthor(bookForAuthorFromRepo);

    if (!_libraryRepository.Save())
    {
        throw new Exception($"Updating book {id} for author {authorId} failed on save.");
    }

    return NoContent(); //anche Ok(...) è lecito, ma richiede più traffico sulla rete
}

```

## Updating Collection Resources

As we remember from when we talked about deleting a collection resource in the previous module, a resource is just a resource. Single or collection, there's nothing that forbids an update. Say we send such a request to `api/authors/{AuthorId}/books`. That means we'd be updating the books resource to a new value, in this case a set of books. We are replacing the current set of books with a new set of books. Put is for full updates after all. The request body replaces what's at the URI the put request is sent to, which means that the correct way to handle this is to delete all the previous books for that author, and then create the list of books that's inputted for that author, so we end up with a completely-new books resource. Very important here is the distinction between what we're actually doing and what the side effects could be. We are still updating a resource, books, so put is warranted. The fact that some book resources are deleted, and others are created when sending that put request is just a side effect. So the reasoning is a bit the same as for deleting collection resources. It's allowed, but in general it's advised against, because it can be quite destructive. The full list of books must be replaced. We won't implement this in our API, but if you need to, you can. Just be aware of the potential consequences.

## Upserting

In the beginning of the module, we learned about the possibility to create a new resource with put or patch instead of post. That principle is called upserting, but it requires a bit of explanation, because it's often misinterpreted and can cause quite a bit of confusion. 

In a lot of systems, it's the server that's responsible for creating the identifier of a resource. Part of it is often the underlying key in the data store, an integer value or a GUID, for example. A consumer might post a new author to the authors resource, and the server response with that newly-created author in the response body, and the location of the author resource, which the server generated in the location header. In fact, in most systems, the server decides on the resource URI, but REST does not have this as a requirement. It's perfectly valid to have a system where the consumer can do this, or where it's allowed for both the consumer and the server. Now if the key in the database is part of the resource URI, this doesn't just work with AutoNum fields. It does, however, when working with GUIDs. So yeah, you got me, **this is one of the reasons I chose GUIDs instead of ints to store data in the backend store**.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur1.PNG" />

```c#

[HttpPut("{id}")]
public IActionResult UpdateBookForAuthor(Guid authorId, Guid id,
    [FromBody] BookForUpdateDto book)
{
    if (book == null)
    {
        return BadRequest();
    }

    if (!_libraryRepository.AuthorExists(authorId))
    {
        return NotFound();
    }

    var bookForAuthorFromRepo = _libraryRepository.GetBookForAuthor(authorId, id);
    if (bookForAuthorFromRepo == null)
    {
        var bookToAdd = Mapper.Map<Book>(book);
        bookToAdd.Id = id;

        _libraryRepository.AddBookForAuthor(authorId, bookToAdd);

        if (!_libraryRepository.Save())
        {
            throw new Exception($"Upserting book {id} for author {authorId} failed on save.");
        }

        var bookToReturn = Mapper.Map<BookDto>(bookToAdd);

        return CreatedAtRoute("GetBookForAuthor",
            new { authorId = authorId, id = bookToReturn.Id },
            bookToReturn);
    }

    Mapper.Map(book, bookForAuthorFromRepo);

    _libraryRepository.UpdateBookForAuthor(bookForAuthorFromRepo);

    if (!_libraryRepository.Save())
    {
        throw new Exception($"Updating book {id} for author {authorId} failed on save.");
    }

    return NoContent();
}

```

## Partially Updating a Resource

Full updates with put aren't always advised, if only for the overhead it creates. In fact, if we would look at the data, which is a standard that's essentially a set of best practices for creating RESTful APIs, we'd notice that standard state, **patch should be preferred over put**, and patch, that's partial updates. 

But that's not sufficient, we need a way to pass a change set to a resource using this HTTP patch method. In other words, what should the body of a patch request look like? Luckily there's a standard for this, the **JSON patch standard**. This defines a JSON document structure for expressing a sequence of operations to apply to a JSON document. You can look at that structure as a change set, a set of operations that'll be applied to the resource at patch request with that JSON patch document in the body is sent to. The `application/json-patch+json` media type is used to identify such patch documents. 

Let's have a look at such a request body. Imagine this is a patch request to a specific book for an author. 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur2.PNG" />

It starts with straight brackets signifying an array, that array, that's a list of operations that have to be applied to the resource. We're seeing two operations in the example on screen. The first one is a replace operation signified by op. Path signifies the path to the property. These are the property names of the resource, the Dto, and not on whatever lies beneath that layer. Value signifies a new value, so the title property gets the value new title. The second operation removes the description of a book. The operation is set to remove, the path is thus /description. There's no value, the properties value will be removed. In some dynamic systems, the property itself will be removed from the resource, but when working with Dto classes, it should be set to its default value. Once this request is received, the API will apply it to the resource. Each operation is applied after the other one, and a request is only successful when all operations can be applied. 

There's six different operations possible. 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur3.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur4.PNG" />

```c#

[HttpPatch("{id}")]
public IActionResult PartiallyUpdateBookForAuthor(Guid authorId, Guid id, [FromBody] JsonPatchDocument<BookForUpdateDto> patchDoc)
{
    if (patchDoc == null)
    {
        return BadRequest();
    }

    if (!_libraryRepository.AuthorExists(authorId))
    {
        return NotFound();
    }

    var bookForAuthorFromRepo = _libraryRepository.GetBookForAuthor(authorId, id);
    if (bookForAuthorFromRepo == null)
    {
        return NotFound();
    }

    var bookToPatch = Mapper.Map<BookForUpdateDto>(bookForAuthorFromRepo);

    patchDoc.ApplyTo(bookToPatch);

    Mapper.Map(bookToPatch, bookForAuthorFromRepo);

    _libraryRepository.UpdateBookForAuthor(bookForAuthorFromRepo);

    if (!_libraryRepository.Save())
    {
        throw new Exception($"Patching book {id} for author {authorId} failed on save.");
    }

    return NoContent();
}

```

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur5.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur6.PNG" />

## Upserting with PATCH

```c#
[HttpPatch("{id}")]
public IActionResult PartiallyUpdateBookForAuthor(Guid authorId, Guid id, [FromBody] JsonPatchDocument<BookForUpdateDto> patchDoc)
{
    if (patchDoc == null)
    {
        return BadRequest();
    }

    if (!_libraryRepository.AuthorExists(authorId))
    {
        return NotFound();
    }

    var bookForAuthorFromRepo = _libraryRepository.GetBookForAuthor(authorId, id);

    if (bookForAuthorFromRepo == null)
    {
        var bookDto = new BookForUpdateDto();
        patchDoc.ApplyTo(bookDto);

        var bookToAdd = Mapper.Map<Book>(bookDto);
        bookToAdd.Id = id;

        _libraryRepository.AddBookForAuthor(authorId, bookToAdd);

        if (!_libraryRepository.Save())
        {
            throw new Exception($"Upserting book {id} for author {authorId} failed on save.");
        }

        var bookToReturn = Mapper.Map<BookDto>(bookToAdd);
        return CreatedAtRoute("GetBookForAuthor",
            new { authorId = authorId, id = bookToReturn.Id },
            bookToReturn);
    }

    var bookToPatch = Mapper.Map<BookForUpdateDto>(bookForAuthorFromRepo);

    patchDoc.ApplyTo(bookToPatch);

    Mapper.Map(bookToPatch, bookForAuthorFromRepo);

    _libraryRepository.UpdateBookForAuthor(bookForAuthorFromRepo);

    if (!_libraryRepository.Save())
    {
        throw new Exception($"Patching book {id} for author {authorId} failed on save.");
    }

    return NoContent();
}
```

## HTTP Method Overview by Use Case

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur7.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur8.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/UpdatingResources/Images/ur9.PNG" />
