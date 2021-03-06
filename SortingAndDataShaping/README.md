- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Implementing Sorting and Data Shaping 

- [Sorting Collection Resources](#sorting-collection-resources)
- [Shaping Resources](#shaping-resources)
- [Shaping a Single Resource](#shaping-a-single-resource)

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

```
http://localhost:6058/api/authors?orderBy=age,genre desc
```

If we use this link:

```
http://localhost:6058/api/authors?orderBy=dateofbirth
```
we will get a 500 status code fault because of this exception:

```c#
// find the matching property
if (!mappingDictionary.ContainsKey(propertyName))
{
    throw new ArgumentException($"Key mapping for {propertyName} is missing");
}
```

And this is not good because this is a consumer error.

### Taking Consumer Errors into Account When Sorting

```c#
public bool ValidMappingExistsFor<TSource, TDestination>(string fields)
{
    var propertyMapping = GetPropertyMapping<TSource, TDestination>();

    if (string.IsNullOrWhiteSpace(fields))
    {
        return true;
    }

    // the string is separated by ",", so we split it.
    var fieldsAfterSplit = fields.Split(',');

    // run through the fields clauses
    foreach (var field in fieldsAfterSplit)
    {
        // trim
        var trimmedField = field.Trim();

        // remove everything after the first " " - if the fields 
        // are coming from an orderBy string, this part must be 
        // ignored
        var indexOfFirstSpace = trimmedField.IndexOf(" ");
        var propertyName = indexOfFirstSpace == -1 ?
            trimmedField : trimmedField.Remove(indexOfFirstSpace);

        // find the matching property
        if (!propertyMapping.ContainsKey(propertyName))
        {
            return false;
        }
    }
    return true;
}

[HttpGet(Name = "GetAuthors")]
public IActionResult GetAuthors(AuthorsResourceParameters authorsResourceParameters)
{
    if (!_propertyMappingService.ValidMappingExistsFor<AuthorDto, Author>(authorsResourceParameters.OrderBy))
    {
        return BadRequest();
    }
    ...
```

## Shaping Resources

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/SortingAndDataShaping/Images/Sds4.PNG" />

But with data shaping, we no longer want to return an IEnumerable of AuthorDTO. We want to return an IEnumerable of a new object, one that only contains the requested fields. We need a way to dynamically create an object that runtime starting from our AuthorDTO. So we're going to work with dynamically typed objects in this demo. This is where the ExpandoObject comes in. ExpandoObject represents an object whose members can be dynamically added and removed at runtime. The extension method we want to create, it doesn't matter to that extends. IEnumerable of T accepts our fields and returns an IEnumerable of dynamically typed objects, ExpandoObjects that only contain the fields we requested.

```c#
public static class IEnumerableExtensions
{
    public static IEnumerable<ExpandoObject> ShapeData<TSource>(
        this IEnumerable<TSource> source,
        string fields)
    {
        if (source == null)
        {
            throw new ArgumentNullException("source");
        }

        // create a list to hold our ExpandoObjects
        var expandoObjectList = new List<ExpandoObject>();

        // create a list with PropertyInfo objects on TSource.  Reflection is
        // expensive, so rather than doing it for each object in the list, we do 
        // it once and reuse the results.  After all, part of the reflection is on the 
        // type of the object (TSource), not on the instance
        var propertyInfoList = new List<PropertyInfo>();

        if (string.IsNullOrWhiteSpace(fields))
        {
            // all public properties should be in the ExpandoObject
            var propertyInfos = typeof(TSource)
                    .GetProperties(BindingFlags.Public | BindingFlags.Instance);

            propertyInfoList.AddRange(propertyInfos);
        }
        else
        {
            // only the public properties that match the fields should be
            // in the ExpandoObject

            // the field are separated by ",", so we split it.
            var fieldsAfterSplit = fields.Split(',');

            foreach (var field in fieldsAfterSplit)
            {
                // trim each field, as it might contain leading 
                // or trailing spaces. Can't trim the var in foreach,
                // so use another var.
                var propertyName = field.Trim();

                // use reflection to get the property on the source object
                // we need to include public and instance, b/c specifying a binding flag overwrites the
                // already-existing binding flags.
                var propertyInfo = typeof(TSource)
                    .GetProperty(propertyName, BindingFlags.IgnoreCase | BindingFlags.Public | BindingFlags.Instance);

                if (propertyInfo == null)
                {
                    throw new Exception($"Property {propertyName} wasn't found on {typeof(TSource)}");
                }

                // add propertyInfo to list 
                propertyInfoList.Add(propertyInfo);
            }
        }

        // run through the source objects
        foreach (TSource sourceObject in source)
        {
            // create an ExpandoObject that will hold the 
            // selected properties & values
            var dataShapedObject = new ExpandoObject();

            // Get the value of each property we have to return.  For that,
            // we run through the list
            foreach (var propertyInfo in propertyInfoList)
            {
                // GetValue returns the value of the property on the source object
                var propertyValue = propertyInfo.GetValue(sourceObject);

                // add the field to the ExpandoObject
                ((IDictionary<string, object>)dataShapedObject).Add(propertyInfo.Name, propertyValue);
            }

            // add the ExpandoObject to the list
            expandoObjectList.Add(dataShapedObject);
        }

        // return the list

        return expandoObjectList;
    }
}

public class AuthorsResourceParameters
{
    ...
    public string Fields { get; set; }
}


return Ok(authors.ShapeData(authorsResourceParameters.Fields));
```
We need to check if, for a given string of fields, all properties matching those fields exist on a given type.

```c#
public class TypeHelperService : ITypeHelperService
{
    public bool TypeHasProperties<T>(string fields)
    {
        if (string.IsNullOrWhiteSpace(fields))
        {
            return true;
        }

        // the field are separated by ",", so we split it.
        var fieldsAfterSplit = fields.Split(',');

        // check if the requested fields exist on source
        foreach (var field in fieldsAfterSplit)
        {
            // trim each field, as it might contain leading 
            // or trailing spaces. Can't trim the var in foreach,
            // so use another var.
            var propertyName = field.Trim();

            // use reflection to check if the property can be
            // found on T. 
            var propertyInfo = typeof(T)
                .GetProperty(propertyName, BindingFlags.IgnoreCase | BindingFlags.Public | BindingFlags.Instance);

            // it can't be found, return false
            if (propertyInfo == null)
            {
                return false;
            }
        }

        // all checks out, return true
        return true;
    }
}

        [HttpGet(Name = "GetAuthors")]
        public IActionResult GetAuthors(AuthorsResourceParameters authorsResourceParameters)
        {
            ...
            if (!_typeHelperService.TypeHasProperties<AuthorDto>(authorsResourceParameters.Fields))
            {
                return BadRequest();
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
                    fields = authorsResourceParameters.Fields,
                   ...
                });
        case ResourceUriType.NextPage:
            return _urlHelper.Link("GetAuthors",
                new
                {
                    fields = authorsResourceParameters.Fields,
                  ...
                });

        default:
            return _urlHelper.Link("GetAuthors",
                new
                {
                    fields = authorsResourceParameters.Fields,
               ...
                });
    }
}

```
<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/SortingAndDataShaping/Images/Sds5.PNG" />

And now we get back the IDs. But something's off. Our field name is no longer Camel-cased. Let's have a look at why that's the case, and what we can do about it.

Our field names are no longer Camel-cased because the default contract resolver that serializes the output doesn't correctly work with the dictionary. The key is serialized as is instead of being CamelCased, and as an ExpandoObject uses a dictionary underneath the covers, where the key is the property name, this is the result. But, no worries, we can change that by using another contract resolver.

Let's open the ConfigureServices method again on our startup class. 

```c#
services.AddMvc(setupAction =>
{
    setupAction.ReturnHttpNotAcceptable = true;
    setupAction.OutputFormatters.Add(new XmlDataContractSerializerOutputFormatter());
    setupAction.InputFormatters.Add(new XmlDataContractSerializerInputFormatter());
})
.AddJsonOptions(options =>
{
    options.SerializerSettings.ContractResolver =
    new CamelCasePropertyNamesContractResolver();
});
```

It's perfectly valid to use this on a single resource as well, so let's check that out in the next demo.

## Shaping a Single Resource

```c#
public static class ObjectExtensions
{
    public static ExpandoObject ShapeData<TSource>(this TSource source,
      string fields)
    {
        if (source == null)
        {
            throw new ArgumentNullException(nameof(source));
        }

        var dataShapedObject = new ExpandoObject();

        if (string.IsNullOrWhiteSpace(fields))
        {
            // all public properties should be in the ExpandoObject 
            var propertyInfos = typeof(TSource)
                    .GetProperties(BindingFlags.IgnoreCase | BindingFlags.Public | BindingFlags.Instance);

            foreach (var propertyInfo in propertyInfos)
            {
                // get the value of the property on the source object
                var propertyValue = propertyInfo.GetValue(source);

                // add the field to the ExpandoObject
                ((IDictionary<string, object>)dataShapedObject).Add(propertyInfo.Name, propertyValue);
            }

            return dataShapedObject;
        }

        // the field are separated by ",", so we split it.
        var fieldsAfterSplit = fields.Split(',');

        foreach (var field in fieldsAfterSplit)
        {
            // trim each field, as it might contain leading 
            // or trailing spaces. Can't trim the var in foreach,
            // so use another var.
            var propertyName = field.Trim();

            // use reflection to get the property on the source object
            // we need to include public and instance, b/c specifying a binding flag overwrites the
            // already-existing binding flags.
            var propertyInfo = typeof(TSource)
                .GetProperty(propertyName, BindingFlags.IgnoreCase | BindingFlags.Public | BindingFlags.Instance);

            if (propertyInfo == null)
            {
                throw new Exception($"Property {propertyName} wasn't found on {typeof(TSource)}");
            }

            // get the value of the property on the source object
            var propertyValue = propertyInfo.GetValue(source);

            // add the field to the ExpandoObject
            ((IDictionary<string, object>)dataShapedObject).Add(propertyInfo.Name, propertyValue);
        }

        // return
        return dataShapedObject;
    }
}

[HttpGet("{id}", Name = "GetAuthor")]
public IActionResult GetAuthor(Guid id, [FromQuery] string fields)
{
    if (!_typeHelperService.TypeHasProperties<AuthorDto>
        (fields))
    {
        return BadRequest();
    }

    var authorFromRepo = _libraryRepository.GetAuthor(id);

    if (authorFromRepo == null)
    {
        return NotFound();
    }

    var author = Mapper.Map<AuthorDto>(authorFromRepo);
    return Ok(author.ShapeData(fields));
}
```

An interesting fact here is that by allowing data shaping, we could potentially violate a rest constraint. We're currently in violation of the manipulation of resources through the presentations constraint that said that, when a client holds a representation of a resource, including any possible method data, it should have enough information to modify or delete a resource on the server, provided it has permission to do so. Let's send that previous request again. We are currently not adhering to that yet. We're just returning an ID. And to adhere to it, we should either include a resource URI in the response, or better yet, support HATEOAS. That's coming up in the next module. But let's imagine the ID is a resource URI for a second. We can't send this request only asking for the name. This time we don't even have that ID anymore, and that can't be good. So I just want to mention this here because not all APIs support HATEOAS, and if you don't want to support HATEOAS, and you're opting to add the resource URI to each response, just make sure you don't allow data shaping to omit this URI. And this kind of proves that it's not always that simple to adhere to these constraints and design a truly RESTful system. Before we know it, we introduce functionality that breaks our architectural style. Anyway, so far for data shaping. 
