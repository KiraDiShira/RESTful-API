- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Implementing Paging, Filtering, and Searching

- [Paging Through Collection Resources](#paging-through-collection-resources)
- [The Principle of Deferred Execution](#the-principle-of-deferred-execution)

## Paging Through Collection Resources

Most APIs, just like ours, expose collection resources. In our case, that's authors and books, and these collection resources can grow quite large. It's considered best practice to always implement paging on each resource collection or, at least, on those resources that can also be created. This is to avoid unintended negative effects on performance when the resource collection grows. Not having paging on the list of ten authors might be okay, but if our API allows creating authors, this list can grow, and we don't want to end up with accidentally returning thousands of authors in one response, as that will definitely have an adverse effect on performance. 

If you remember from the second module, where we designed the outer-facing contract, options like paging, sorting, filtering, and others are passed through via the query string. These are not resources in their own right. They're parameters that manipulate a resource collection that's returned. 

```
http://host/api/authors?pageNumber=1&pageSize=5
```

As far as paging is concerned, the consumer should be able to choose the page number and page size, but that page size can cause problems as well. If we don't limit this, a consumer can pass through 100,000 as page size, which will still cause performance issues. The page size itself should be checked against a specified value to see if it isn't too large, and if no paging parameters are provided, we should only return first page by default. Always page even if it isn't specifically asked for by the consumer. We're manipulating collection resources, but for things like paging and the other options we're looking through this module to work correctly and have a positive impact on performance, we need to ensure this goes all the way through to our data store. If we have thousands of authors in our database, and we first return all those authors from the repository to the controller and then page them, well, we're still fetching way too much data from the database. We're not really helping performance in that regard then, but how can this happen technically, with link and Entity Framework Core? Well, it can happen thanks to the principle named deferred execution. Let's have a look at what that is.

## The Principle of Deferred Execution

When working with Entity Framework Core, we use linq to build our queries. With deferred execution, the query valuable itself never holds the query results, and only stores the query commands. Execution of the query is deferred until the query variable is iterated over. **Deferred execution** means that query execution occurs sometime after the query has been constructed. We can get this behavior by working with iQueryable implementing collections. iQueryable T allows us to execute a query against a specific data source, and while building upon it, it creates an expression tree. That's those query commands, but the query itself, isn't actually sent to the data store until iteration happens. Iteration can happen in different ways. One way is by using an iQueryable in a loop. Another, is by calling into something like ToList(), ToArray(), or ToDictionary() on it because that means converting the expression tree to an actual list of items. Another way is by calling singleton queries. Singleton queries are queries like average, count, and first, because, to get the count, or first item of an iQueryable, the list has to be iterated over. As long as we can avoid that, we can build our query by, for example, adding take and skip statements for paging and assure it's only executed after that. 
