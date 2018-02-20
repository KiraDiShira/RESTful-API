# Getting Resources

## Structuring Our Outer Facing Contract

The outer facing contract consists of three big concepts a consumer of an API uses to interact with that API.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr1.PNG" />

Method Definitions: https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html

When creating a resource, the HTTP response will contain a resource representation in its body. The format of those representations is what **media types** are used for, like application JSON. 

The uniform interface constraint does cover the fact that resources are identified by URIs. Each resource has its own URI, but as far as naming of resources is concerned, there isn't a standard that describes that, or at least not unless you want to dive into OData. There are, however, best practices for this.

A resource name in a URI should always be a noun. In other words, a RESTful URI should refer to a resource that is a thing, instead of referring to an action. So we shouldn't create a *getauthors* resource, that's an action. We should create an *authors* resource, that's a thing conveyed by a noun, and use the *GET* method to get it. To get one specific author then, we'd append it with a forward slash and the authorId. Using these nouns conveys meaning, it describes the resource, so we shouldn't call a resource orders, when it's in fact about authors. That principle should be followed throughout the API for predictability. It helps a consumer understand the API structure. 

If we have another non-hierarchical resource, say employees, we shouldn't name it api/something/something/employees, we should name it api/employees. A single employee then shouldn't be named id/employees, it should be named employees/{employeeId}. This helps keep the API contract predictable and consistent. 

There's quite a bit of a debate going on on whether or not we should pluralize these nouns. I prefer to pluralize them as it helps to convey meaning. When I see an authors resource, that tells us it's a collection of authors and not one author, but good APIs that don't pluralize nouns exist as well. If you prefer that you can, but do make sure to stay consistent. Either all resources should be pluralized nouns, or singular nouns, and not a mix. 

Another important thing you'd want to represent in an API contract is the hierarchy. Our data or models have structure. For example, an author has books that should be represented in the API contract. So if you want to define an author's books, where the books in the model hierarchy are children of an author, we should represent them as  `api/authors/{authorId}/books `. A single book should then be followed by the bookId. 

APIs often expose additional capabilities. Later on we'll learn about filtering and ordering resources, those parameters should be passed through the credit string, they aren't resources in their own right. So we shouldn't write something along the lines of  `api/authors/orderby/name `. There's a few contract smells in that URI. A plural noun should be followed by an Id, and not by another word, and orderby isn't a noun, and a URI like this would mean we'd have defined three different resources - authors, authors/orderby, and authors/orderby/name. So  `api/authors?orderby=name` is a better fit. 

And with that we've already covered a lot, but there is an exception. There always has to be an exception, life would be too easy without them. Sometimes there's these remote procedure calls, style-like calls, like calculate total, that don't easily map to resources. Most RPC-style like calls do map to resources, as we've just proven, but what if we need to calculate, say, the total amount of pages an author wrote? It's not that easy to create a resource from that using pluralized nouns. You'd end up with something like `api/authors/{authorId}/pagetotals`, and then what? Because we'd expect this to return a collection and not a number. You could go for something else, like `api/authorpagetotals/{Id}`, where the backend would then have to map that Id to an authorId, or it can even be the same Id, and that would work, and it does fit the URI design guidelines, but it does feel a bit out of place so this is one of these examples, or one of these exceptional cases, where I'd suggest to take a bit of a pragmatic approach, `api/authors/{authorId}/totalamountofpages`. It isn't according to these best practices, but as long as it's an exceptional case, it doesn't mean you've suddenly got a bad API. Remember there isn't standards for following for these naming guidelines, these are just guidelines.

So by following these simple rules we'll end up with a good resource URI design. But there's one more thing we need to talk about, Ids. Should these be integers? Typically auto-numbered from the database, or should they be GUIDs? In fact, REST stops at that outer facing contract. The layers underneath, including the data store, are of no importance to REST, so getting an author might mean that you're actually fetching data from three different data stores, including some fields from Active Directory, to compose that author resource representation. So it's of no importance, our resource isn't the same as what is in the backend store. These are two different concepts. But from that follows the question, what should we use as identifiers? REST is unrelated to the backend data store, yet often you'll see APIs that actually use the auto-numbered primary key Ids from the database. If the backend doesn't matter, what happens to the resource URIs if you change the backend? The resource URI should remain the same, but if resources are identified by their database auto-numbered fields, and we switch out our current SQL Server to a backend that uses another type of auto-number sequence like MongoDB, all of a sudden all our resource Ids can change. We can, of course, work around that on migration, but still, it's a good idea to keep this in mind when designing resource URIs. And there's a solution for this. GUIDs, unique and unguessable values you can use as primary key in every database. From that we can then switch out datastore technologies and our resource URIs will stay the same. We're also no longer potentially exposing implementation details, as those GUIDs don't give anything away about the underlying technology. They work with all of them, so that's why we'll use GUIDs in this course. This advantage, in my book, is readability. As a developer it's not that convenient to type over a GUID to test an API call, but that's what testing tools are for, and for the end product it shouldn't matter. It's not users like you and me who tend to talk to an API, say for during development, its other pieces of code. And for those it really doesn't matter if the URI contains a hard-to-type GUID. And with that it's time to start implementing this contract.

## Working with routing

Routing matches request URI to an action on a controller. There are two ways: convention-based and attribute-based.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr2.PNG" />

Use attribute at controller and action level: [Route], [HttpGet] ...

## Interatcting with resources through HTTP methods

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr3.PNG" />

**HEAD** is identical to GET with the notable difference that the API shouldn't return a response body, so no response payload. It can be used to obtain information on the resource like testing it for validity, for example, to test if a resource exists. 

**OPTIONS** represents a request for information about the communication options available on that URI. So in other words, OPTIONS will tell us whether or not we can GET the resource, POST it, DELETE it and so on. These OPTIONS are typically in the response headers and not in the body, so no response payload.

## Outer Facing Model vs. Entity Model

The outer facing model does only represents the resources that are sent over the wire in a specific format

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/GettingResources/Images/gr3.PNG" />

An author is stored in our database with a DateOfBirth, but that DateOfBirth, well that might not be what we want to offer up to the consumers of the API. They might be better off with the age. Another example might be concatenation. Concatenating the FirstName and LastName from an entity into one name field in the resource representation, and sometimes data might come from different places. An author could have a field, say, Royalties, that comes from another API our API must interact with. That alone leads to issues when using entity classes for the outer facing contract, as they don't contain that field. 

Keeping these models separate leads to more robust, reliably evolvable code. Imagine having to change a database table, that would lead to a change of the Entity class. If we're using that same Entity class to directly expose data via the API, our clients might run into problems because they're not expecting an additional renamed or removed field. So when developing, it's fairly important to keep these separate.
