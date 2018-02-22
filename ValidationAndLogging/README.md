- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Working with validation and logging

- [ Working with Validation in a RESTful World](#working-with-validation-in-a-restful-world)
- [Working with Validation on POST](#working-with-validation-on-post)
- [Working with Validation on PUT](#working-with-validation-on-put)

##  Working with Validation in a RESTful World

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/ValidationAndLogging/Images/val1.PNG" />

Mind you we don't return those errors to let the consumer know whether he or the server made a mistake, that's what the StatusCodes are for. However, often the response body contains a set of validation error messages that the client application could use to show to the user of the application that consumes the API. Most applications require some rules on their objects, be it on a Dto, a business, or a domain object, or an entity. If those rules don't check out, the request should be halted. To **create these validation rules**, we can use ASP.NET Core's built-in approach or a third-party component. In ASP.NET Core, the default approach to this is using data annotations on our properties. For all common property validation rules, annotations exist, like required or max length. Next to that, we can also define custom rules, rules we can't just easily define with annotations.

From a conceptual point of view, what's of importance is what we want to validate. The rule is that we typically validate rules on input, and not on output, so for `post`, `put`, and `patch`, but for not for `get`. 

This again drives home the point that we should use separate Dtos for separate operations. And that brings us to step two, checking validation rules. 

For **checking validation rules** in ASP.NET Core, the building concept of `ModelState` is used. `ModelState` is a dictionary containing both the state of the model and model binding validation. It represents a collection of name and value pairs that were submitted to the API, one for each property. It also contains a collection of error messages for each value submitted. Whenever a request comes in, the rules we define in step one are checked. If one of them doesn't check out, the `ModelState`'s valid property will be false, and this property will also be false if an invalid value for a property type is passed in. 

The final step, **reporting validation errors**. When a validation error happens, the consumer of the API needs to be notified. It's a mistake of the client, so that warrants a 400 level StatusCode. There's one that's specifically used for this, **422 Unprocessable Entity**. As we learned before, the 422 StatusCode means that the server understood the content type of the request entity, and the syntax of the request entity is correct, but it wasn't able to process the contained instructions. For example, the syntax is correct, but the semantics are off, and that's a perfect fit for what we need. But when we say reporting validation errors, we're not only talking about the StatusCode, we're also talking about response body. What we write out in the response body isn't regulated by REST, nor by any of the standards we're using, but for validation issues, this will often be a list of properties and their related error messages. You can see an example:

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/ValidationAndLogging/Images/val2.PNG" />

If the client application needs to be able to do something more than simply binding these errors to a control, often an error code is included as well, so the client application can act on that error code, also quite useful for multi-language applications. In APS.NET CORE MVC, typically the ModelState's validation errors are serialized, so we end up with a list of property names and related validation errors. But this can vary depending on what we're building the API for.

## Working with Validation on POST

First we need to define the rules, and we can do that with data annotations.

```c#
public class BookForCreationDto
{
    [Required]
    [MaxLength(100)]
    public string Title { get; set; }

    [MaxLength(500)]
    public string Description { get; set; }
}

public class UnprocessableEntityObjectResult : ObjectResult
{
    public UnprocessableEntityObjectResult(ModelStateDictionary modelState)
        : base(new SerializableError(modelState))
    {
        if (modelState == null)
        {
            throw new ArgumentNullException(nameof(modelState));
        }
        StatusCode = 422;
    }
}

[HttpPost]
public IActionResult CreateBookForAuthor(Guid authorId, [FromBody] BookForCreationDto book)
{
    if (book == null)
    {
        return BadRequest();
    }

    if (!ModelState.IsValid)
    {
        //return 422
        return new UnprocessableEntityObjectResult(ModelState);
    }
    ...
}

```

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/ValidationAndLogging/Images/val3.PNG" />

what we see here are the default messages that come with the annotations. We can change those.

```c#
public class BookForCreationDto
{
    [Required(ErrorMessage = "you should fill out a title.")]
    [MaxLength(100)]
    public string Title { get; set; }

    [MaxLength(500)]
    public string Description { get; set; }
}
```

This is just simple validation. Some business or validation rules can't be covered with data annotations.

```c#
[HttpPost]
public IActionResult CreateBookForAuthor(Guid authorId, [FromBody] BookForCreationDto book)
{
    if (book == null)
    {
        return BadRequest();
    }

    if (book.Description == book.Title)
    {
        ModelState.AddModelError(nameof(BookForCreationDto), "The provided description should be different from the title");
    }

    if (!ModelState.IsValid)
    {
        return new UnprocessableEntityObjectResult(ModelState);
    }
```
## Working with Validation on PUT

So isn't this exactly the same as on post? Well, for some resources, if not most, that's probably the case, but it doesn't have to be.

```c#
public abstract class BookForManipulationDto
{
    [Required(ErrorMessage = "You should fill out a title.")]
    [MaxLength(100, ErrorMessage = "The title shouldn't have more than 100 characters.")]
    public string Title { get; set; }

    [MaxLength(500, ErrorMessage = "The description shouldn't have more than 500 characters.")]
    public virtual string Description { get; set; }
}

public class BookForCreationDto : BookForManipulationDto
{
}

public class BookForUpdateDto : BookForManipulationDto
{
    [Required(ErrorMessage = "You should fill out a description.")]
    public override string Description
    {
        get
        {
            return base.Description;
        }

        set
        {
            base.Description = value;
        }
    }
}

[HttpPut("{id}")]
public IActionResult UpdateBookForAuthor(Guid authorId, Guid id,
    [FromBody] BookForUpdateDto book)
{
    if (book == null)
    {
        return BadRequest();
    }

    if (book.Description == book.Title)
    {
        ModelState.AddModelError(nameof(BookForUpdateDto),
            "The provided description should be different from the title.");
    }

    if (!ModelState.IsValid)
    {
        return new UnprocessableEntityObjectResult(ModelState);
    }

```

Our custom rule here in the UpdateBookForAuthor action, well, that's almost the same as the custom rule in the CreateBookForAuthor action. We could factor that out into a separate method, or create some sort of validation service that would handle all the validation for us, but the main issue is in my personal opinion with how ASP.NET and ASP.NET Core by default handle this validation. Annotations mix in rules with models, and that's not really a good separation of concerns. And having to write validation rules in two different places, the model and the controller, for the same model, doesn't feel right either. It's good enough for our purposes, because after all, we are talking about RESTful architectures, so that's what we're focusing on. However, I do want to give a tip for when you're building more complex applications, then it might be a good idea to keep an eye on something like **fluent validation**, which offers a fluent interface to build validation rules for your objects. From version 6.4 and onwards, .NET Core is supported.

