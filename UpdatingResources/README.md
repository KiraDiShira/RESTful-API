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
