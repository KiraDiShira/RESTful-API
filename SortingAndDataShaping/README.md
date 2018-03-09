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
<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/SortingAndDataShaping/Images/Sds2.PNG" />
