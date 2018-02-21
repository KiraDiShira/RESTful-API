# Updating resources

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
