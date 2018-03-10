- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Advanced HATEOAS, Media Types, and Versioning 

- [HATEOAS and Content Negotiation](#hateoas-and-content-negotiation)

## HATEOAS and Content Negotiation

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/AdvancedHATEOAS/Images/ah1.PNG" />

Onscreen is part of a response from a GET request to the author's resource. We've got two fields, value containing the authors and links containing a self-link, which you see onscreen. The links are part of the resource body. 

That begs the question, **Is this really the correct place to put these links?**

If we think back about pagination, we talked about where the pagination information should go. We concluded that it's metadata, so it should be in the header. It describes the resource. And that's true for fields like total count, current page, and so on. But what about those next page and previous page links? We put the paging links in the response body and kept the rest of the paging information as metadata in the header. So are these links part of the resource or do they describe the resource, i.e. are they metadata? Same goes for the other links, links to self and so on. 

If it's metadata describing the resource, they should go in the header. If they are explicit parts of the resource, they can go in the response body. What we're actually dealing with here are two different representations of the resource.

With content negotiation, we ask through the accept header to get back a JSON representation of the resource. We ask for media type application/json, but what we return isn't a representation of the author's resource. It's something else. Additional semantics, the links, on top of the JSON representation. So the links should not be part of the resource when we ask for media type application/json. We've effectively created another media type which were wrongly returning when asking for application/json. So by doing this, we're breaking the self-descriptive message of constraint, which states each message must include enough info to describe how to process the message. 

If we return a representation with links with an accept header of application/json, we're not only returning the wrong representation, the response will have a content-type header with value application/json, which does not match the content of the response. So the consumer of the API does not know how to interpret the response judging from the content type. In other words, we also don't tell the consumer how to process it.

The solution is creating a new media type. So how do we do that? Well, there's a format for this, a vendor-specific media type. You can see an example of that:

```
application/vnd.marvin.hateoas+json
```


onscreen. First part after application is vnd, the vendor prefix. That's a reserve principle has to begin the mime type with. It indicates that this is a content type that's vendor specific. It's then followed by a custom identifier of the vendor and possibly additional values. A good one to use in our case would be vnd.marvin.hateoas, as you see onscreen. Vnd plus my company, which happens to be called Marvin, the paranoid android from Douglas Adams' Hitchiker's Guide to the Galaxy books and then followed by HATEOAS, stating we want to include those resources links. Then we add a plus sign and JSON. What we're actually stating here is that we want a resource representation in JSON with HATEOAS links. If that new media type is requested, the links should be included. The consumer must know about this media type and how to process it. That's what documentation is for. If this media type isn't requested, so we simply request application/json, the links should not be included. Let's see how we can support this with a demo.

```
