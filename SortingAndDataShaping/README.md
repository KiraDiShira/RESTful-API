- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Implementing Sorting and Data Shaping 

- [Sorting Collection Resources](#sorting-collection-resources)

## Sorting Collection Resources

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/SortingAndDataShaping/Images/Sds1.PNG" />

```c#
public class AuthorsResourceParameters
{
    ...
    public string OrderBy { get; set; } = "Name";
}

```

 Let's have a look at that OrderBy class. If you look through the overloads, we see there is no overload that allows us to pass in a string. And we really want to avoid having to write a huge switch statement for all possible sorting combinations. Now, luckily, this isn't a new requirement. In fact, it's for these types of requirements that Microsoft created the System.Linq.Dynamic library. Through that library, we are able to sort on strings. So let's add that package. 

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/SortingAndDataShaping/Images/Sds2.PNG" />

Okay, so now we should be able to sort by string. But doing all that in this repository method won't exactly lead to reusability. We can separate it out into an extension method on IQueryable. So what you want to end up with is a sort of ApplySort extension method on IQueryable that accepts the OrderBy string from our query string parameters, and a mapping dictionary.

We can create a PropertyMappingService

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/SortingAndDataShaping/Images/Sds3.PNG" />

```c#
public class PropertyMappingValue
{
    public IEnumerable<string> DestinationProperties { get; }
    public bool Revert { get; }

    public PropertyMappingValue(IEnumerable<string> destinationProperties,
        bool revert = false)
    {
        DestinationProperties = destinationProperties;
        Revert = revert;
    }
}

public class PropertyMapping<TSource, TDestination> : IPropertyMapping
{
    public Dictionary<string, PropertyMappingValue> _mappingDictionary { get; }
    public PropertyMapping(Dictionary<string, PropertyMappingValue> mappingDictionary)
    {
        _mappingDictionary = mappingDictionary;
    }
}

public class PropertyMappingService : IPropertyMappingService
{
    private Dictionary<string, PropertyMappingValue> _authorPropertyMapping =
       new Dictionary<string, PropertyMappingValue>(StringComparer.OrdinalIgnoreCase)
       {
           { "Id", new PropertyMappingValue(new List<string>() { "Id" } ) },
           { "Genre", new PropertyMappingValue(new List<string>() { "Genre" } )},
           { "Age", new PropertyMappingValue(new List<string>() { "DateOfBirth" } , true) },
           { "Name", new PropertyMappingValue(new List<string>() { "FirstName", "LastName" }) }
       };

    private IList<IPropertyMapping> propertyMappings = new List<IPropertyMapping>();

    public PropertyMappingService()
    {
        propertyMappings.Add(new PropertyMapping<AuthorDto, Author>(_authorPropertyMapping));
    }
    public Dictionary<string, PropertyMappingValue> GetPropertyMapping
        <TSource, TDestination>()
    {
        // get matching mapping
        var matchingMapping = propertyMappings.OfType<PropertyMapping<TSource, TDestination>>();

        if (matchingMapping.Count() == 1)
        {
            return matchingMapping.First()._mappingDictionary;
        }

        throw new Exception($"Cannot find exact property mapping instance for <{typeof(TSource)},{typeof(TDestination)}");
    }
}

using System.Linq.Dynamic.Core;

namespace Library.API.Helpers
{
    public static class IQueryableExtensions
    {
        public static IQueryable<T> ApplySort<T>(this IQueryable<T> source, string orderBy,
            Dictionary<string, PropertyMappingValue> mappingDictionary)
        {
            if (source == null)
            {
                throw new ArgumentNullException("source");
            }

            if (mappingDictionary == null)
            {
                throw new ArgumentNullException("mappingDictionary");
            }

            if (string.IsNullOrWhiteSpace(orderBy))
            {
                return source;
            }
            // the orderBy string is separated by ",", so we split it.
            var orderByAfterSplit = orderBy.Split(',');

            // apply each orderby clause in reverse order - otherwise, the 
            // IQueryable will be ordered in the wrong order
            foreach (var orderByClause in orderByAfterSplit.Reverse())
            {
                // trim the orderByClause, as it might contain leading 
                // or trailing spaces. Can't trim the var in foreach,
                // so use another var.
                var trimmedOrderByClause = orderByClause.Trim();

                // if the sort option ends with with " desc", we order
                // descending, otherwise ascending
                var orderDescending = trimmedOrderByClause.EndsWith(" desc");

                // remove " asc" or " desc" from the orderByClause, so we 
                // get the property name to look for in the mapping dictionary
                var indexOfFirstSpace = trimmedOrderByClause.IndexOf(" ");
                var propertyName = indexOfFirstSpace == -1 ?
                    trimmedOrderByClause : trimmedOrderByClause.Remove(indexOfFirstSpace);

                // find the matching property
                if (!mappingDictionary.ContainsKey(propertyName))
                {
                    throw new ArgumentException($"Key mapping for {propertyName} is missing");
                }

                // get the PropertyMappingValue
                var propertyMappingValue = mappingDictionary[propertyName];

                if (propertyMappingValue == null)
                {
                    throw new ArgumentNullException("propertyMappingValue");
                }

                // Run through the property names in reverse
                // so the orderby clauses are applied in the correct order
                foreach (var destinationProperty in propertyMappingValue.DestinationProperties.Reverse())
                {
                    // revert sort order if necessary
                    if (propertyMappingValue.Revert)
                    {
                        orderDescending = !orderDescending;
                    }
                    source = source.OrderBy(destinationProperty + (orderDescending ? " descending" : " ascending"));
                }
            }
            return source;
        }
    }   
}

public PagedList<Author> GetAuthors(AuthorsResourceParameters authorsResourceParameters)
{
    //var collectionBeforePaging =
    //    _context.Authors
    //        .OrderBy(a => a.FirstName)
    //        .ThenBy(a => a.LastName)
    //        .AsQueryable();

    var collectionBeforePaging = _context.Authors.ApplySort(authorsResourceParameters.OrderBy,
        _propertyMappingService.GetPropertyMapping<AuthorDto, Author>());
        ...
 }
 
 private string CreateAuthorsResourceUri(
    AuthorsResourceParameters authorsResourceParameters,
    ResourceUriType type)
{
    switch (type)
    {
        case ResourceUriType.PreviousPage:
            return _urlHelper.Link("GetAuthors",
                new
                {
                    orderBy = authorsResourceParameters.OrderBy,
                    ...
                });
        case ResourceUriType.NextPage:
            return _urlHelper.Link("GetAuthors",
                new
                {
                    orderBy = authorsResourceParameters.OrderBy,
                    ...
                });

        default:
            return _urlHelper.Link("GetAuthors",
                new
                {
                    orderBy = authorsResourceParameters.OrderBy,
                    ...
                });
    }
}


```
