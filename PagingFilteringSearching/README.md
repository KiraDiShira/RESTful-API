- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Implementing Paging, Filtering, and Searching

- [Paging Through Collection Resources](#paging-through-collection-resources)
- [The Principle of Deferred Execution](#the-principle-of-deferred-execution)
- [Returning pagination metadata](#returning-pagination-metadata)

## Paging Through Collection Resources

Most APIs expose collection resources. In our case, that's authors and books, and these collection resources can grow quite large. It's considered best practice to always implement paging on each resource collection or, at least, on those resources that can also be created. This is to avoid unintended negative effects on performance when the resource collection grows. Not having paging on the list of ten authors might be okay, but if our API allows creating authors, this list can grow, and we don't want to end up with accidentally returning thousands of authors in one response, as that will definitely have an adverse effect on performance. 

If you remember from the second module, where we designed the outer-facing contract, options like paging, sorting, filtering, and others are passed through via the query string. These are not resources in their own right. They're parameters that manipulate a resource collection that's returned. 

```
http://host/api/authors?pageNumber=1&pageSize=5
```

As far as paging is concerned, the consumer should be able to choose the page number and page size, but that page size can cause problems as well. If we don't limit this, a consumer can pass through 100,000 as page size, which will still cause performance issues. The page size itself should be checked against a specified value to see if it isn't too large, and if no paging parameters are provided, we should only return first page by default. Always page even if it isn't specifically asked for by the consumer. We're manipulating collection resources, but for things like paging and the other options we're looking through this module to work correctly and have a positive impact on performance, we need to ensure this goes all the way through to our data store. If we have thousands of authors in our database, and we first return all those authors from the repository to the controller and then page them, well, we're still fetching way too much data from the database. We're not really helping performance in that regard then, but how can this happen technically, with link and Entity Framework Core? Well, it can happen thanks to the principle named deferred execution. Let's have a look at what that is.

## The Principle of Deferred Execution

When working with Entity Framework Core, we use linq to build our queries. With deferred execution, the query valuable itself never holds the query results, and only stores the query commands. Execution of the query is deferred until the query variable is iterated over. **Deferred execution** means that query execution occurs sometime after the query has been constructed. We can get this behavior by working with iQueryable implementing collections. iQueryable T allows us to execute a query against a specific data source, and while building upon it, it creates an expression tree. That's those query commands, but the query itself, isn't actually sent to the data store until iteration happens. Iteration can happen in different ways. One way is by using an iQueryable in a loop. Another, is by calling into something like ToList(), ToArray(), or ToDictionary() on it because that means converting the expression tree to an actual list of items. Another way is by calling singleton queries. Singleton queries are queries like average, count, and first, because, to get the count, or first item of an iQueryable, the list has to be iterated over. As long as we can avoid that, we can build our query by, for example, adding take and skip statements for paging and assure it's only executed after that. 

```c#
public class AuthorsResourceParameters
{
    const int maxPageSize = 20;
    public int PageNumber { get; set; } = 1;

    private int _pageSize = 10;
    public int PageSize
    {
        get
        {
            return _pageSize;
        }
        set
        {
            _pageSize = (value > maxPageSize) ? maxPageSize : value;
        }
    }
}

[HttpGet()]
public IActionResult GetAuthors(AuthorsResourceParameters authorsResourceParameters)
{
    var authorsFromRepo = _libraryRepository.GetAuthors(authorsResourceParameters);

...

public IEnumerable<Author> GetAuthors(AuthorsResourceParameters authorsResourceParameters)
{
    return _context.Authors.
        OrderBy(a => a.FirstName)
        .ThenBy(a => a.LastName)
        .Skip(authorsResourceParameters.PageSize * (authorsResourceParameters.PageNumber -1))
        .Take(authorsResourceParameters.PageSize)
        .ToList();
}

http://localhost:6058/api/authors?pageNumber=2&pageSize=5

```

## Returning pagination metadata

Let's think about what would be useful for the consumer of the API to know about a page set of data. At the very least, we should include URIs to request the previous and next pages, so the consumer doesn't have to construct those himself. Additional information, like total count or total amount of pages, is often included as well, just like page number and/or page size. 

How do we include this data in a response then? Well, what's often done, and you've probably already encountered this if you've worked with third party APIs, like Facebook's, is that paging information is included in the response body itself. Often with a metadata tag or paging info tag, but this isn't correct. 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/PagingFilteringSearching/Images/pfs1.PNG" />

If a consumer of the API requests a page of authors, with the application/json media type as value for the accept header, the API should return a JSON representation of the resource. An envelope that has a results field and metadata field, well, that doesn't match the media type we asked for. It's not a JSON representation of the authors collection; it's a different media type. Returning the paging info like this combined with application/json as media type, effectively breaks REST, as we are no longer adhering to the self-descriptive message constraint. That constraint states that each message must contain enough information on how to process it. Next to returning the wrong representation, the response will have content-type pattern with value application/json, which doesn't match the content of the response. In other words, we also don't tell the consumer how to process it. The consumer doesn't know how to interpret the response judging from the content-type. We will learn a lot more media types in the next module because, to be honest, application/json isn't such a good media type to ask for a header. But more on that later.

When requesting application/json, paging information isn't part of the resource representation. It's metadata related to that resource. Metadata, well, that's something that's put in a response header. The consumer can empower that header to get that information. We should create a custom pagination header, like X-Pagination.