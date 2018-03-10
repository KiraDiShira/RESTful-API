- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Getting Started with HATEOAS 

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
