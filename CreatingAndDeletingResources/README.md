# Creating and Deleting Resources

## Method Safety and Method Idempotency

The outer facing contract consists of three big concepts a consumer of an API uses to interact with that API.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/CreatingAndDeletingResources/Images/Cadr1.PNG" />

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/CreatingAndDeletingResources/Images/Cadr2.PNG" />

**Why is PUT idempotent?**

When you PUT a resource, these two assumptions are in play:

1. You are referring to an entity, not to a collection.

2. The entity you are supplying is complete (the entire entity).

Let's look at one of your examples.
```
{ "username": "skwee357", "email": "skwee357@domain.com" }
```

If you POST this document to /users, as you suggest, then you might get back an entity such as

```
.. /users/1

{
    "username": "skwee357",
    "email": "skwee357@domain.com"
}
```
If you want to modify this entity later, you choose between PUT and PATCH. A PUT might look like this:
```
PUT /users/1
{
    "username": "skwee357",
    "email": "skwee357@gmail.com"       // new email address
}
```
You can accomplish the same using PATCH. That might look like this:
```
PATCH /users/1
{
    "email": "skwee357@gmail.com"       // new email address
}
```
You'll notice a difference right away between these two. The PUT included all of the parameters on this user, but PATCH only included the one that was being modified (email).

When using PUT, it is assumed that you are sending the complete entity, and that complete entity replaces any existing entity at that URI. In the above example, the PUT and PATCH accomplish the same goal: they both change this user's email address. But PUT handles it by replacing the entire entity, while PATCH only updates the fields that were supplied, leaving the others alone.

Since PUT requests include the entire entity, if you issue the same request repeatedly, it should always have the same outcome (the data you sent is now the entire data of the entity). Therefore PUT is idempotent.

**Using PUT wrong**
What happens if you use the above PATCH data in a PUT request?
```
GET /users/1
{
    "username": "skwee357",
    "email": "skwee357@domain.com"
}
PUT /users/1
{
    "email": "skwee357@gmail.com"       // new email address
}

GET /users/1
{
    "email": "skwee357@gmail.com"      // new email address... and nothing else!
}
```
(I'm assuming for the purposes of this question that the server doesn't have any specific required fields, and would allow this to happen... that may not be the case in reality.)

Since we used PUT, but only supplied email, now that's the only thing in this entity. This has resulted in data loss.

This example is here for illustrative purposes -- don't ever actually do this. This PUT request is technically idempotent, but that doesn't mean it isn't a terrible, broken idea.

**How can PATCH be idempotent?**
In the above example, PATCH was idempotent. You made a change, but if you made the same change again and again, it would always give back the same result: you changed the email address to the new value.
```
GET /users/1
{
    "username": "skwee357",
    "email": "skwee357@domain.com"
}
PATCH /users/1
{
    "email": "skwee357@gmail.com"       // new email address
}

GET /users/1
{
    "username": "skwee357",
    "email": "skwee357@gmail.com"       // email address was changed
}
PATCH /users/1
{
    "email": "skwee357@gmail.com"       // new email address... again
}

GET /users/1
{
    "username": "skwee357",
    "email": "skwee357@gmail.com"       // nothing changed since last GET
}
```

**So when is PATCH not idempotent, then?**

To show why PATCH isn't idempotent, it helps to start with the definition of idempotence (from Wikipedia):

```
The term idempotent is used more comprehensively to describe an operation that will produce the same results if executed once or multiple times [...] An idempotent function is one that has the property f(f(x)) = f(x) for any value x.
```

In more accessible language, an idempotent PATCH could be defined as: After PATCHing a resource with a patch document, all subsequent PATCH calls to the same resource with the same patch document will not change the resource.

Conversely, a non-idempotent operation is one where f(f(x)) != f(x), which for PATCH could be stated as: After PATCHing a resource with a patch document, subsequent PATCH calls to the same resource with the same patch document do change the resource.

To illustrate a non-idempotent PATCH, suppose there is a /users resource, and suppose that calling  GET /users returns a list of users, currently:

```
[{ "id": 1, "username": "firstuser", "email": "firstuser@example.org" }]
```
Rather than PATCHing /users/{id}, as in the OP's example, suppose the server allows PATCHing /users. Let's issue this PATCH request:

```
PATCH /users
[{ "op": "add", "username": "newuser", "email": "newuser@example.org" }]
```
Our patch document instructs the server to add a new user called newuser to the list of users. After calling this the first time, GET /users would return:

```
[{ "id": 1, "username": "firstuser", "email": "firstuser@example.org" },
 { "id": 2, "username": "newuser", "email": "newuser@example.org" }]
 ```
 
Now, if we issue the exact same PATCH request as above, what happens? (For the sake of this example, let's assume that the /users resource allows duplicate usernames.) The "op" is "add", so a new user is added to the list, and a subsequent GET /users returns:

```
[{ "id": 1, "username": "firstuser", "email": "firstuser@example.org" },
 { "id": 2, "username": "newuser", "email": "newuser@example.org" },
 { "id": 3, "username": "newuser", "email": "newuser@example.org" }]
 ```
The /users resource has changed again, even though we issued the exact same PATCH against the exact same endpoint. If our PATCH is f(x), f(f(x)) is not the same as f(x), and therefore, **this particular PATCH is not idempotent.**

Although PATCH isn't guarateed to be idempotent, there's nothing in the PATCH specification to prevent you from making all PATCH operations on your particular server idempotent. RFC 5789 even anticipates advantages from idempotent PATCH requests:

A PATCH request can be issued in such a way as to be idempotent, which also helps prevent bad outcomes from collisions between two PATCH requests on the same resource in a similar time frame.

In Dan's example, his PATCH operation is, in fact, idempotent. In that example, the /users/1 entity changed between our PATCH requests, but not because of our PATCH requests; it was actually the Post Office's different patch document that caused the zip code to change. The Post Office's different PATCH is a different operation; if our PATCH is f(x), the Post Office's PATCH is g(x). Idempotence states that f(f(f(x))) = f(x), but makes no guarantes about f(g(f(x))).
