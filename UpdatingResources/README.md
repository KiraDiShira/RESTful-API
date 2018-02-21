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
