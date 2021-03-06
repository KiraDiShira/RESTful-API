- [Index](https://github.com/KiraDiShira/RESTful-API#restful-api)

# Introducing REST

- [Introduction](#introduction)
- [REST constraints](#rest-constraints)
- [The Richardson Maturity Model](#the-richardson-maturity-model)

## Introduction

REST isn't just about building an API which consists of a few HTTP services that return JSON.

Roy Fielding definition: **Representational state transfer** is intended to evoke an image of how a well-designed web application behaves: a network of web pages (a virtual state-machine) where the user progresses through an application by selecting links (state-transitions), resulting in the next page (representing the next state of the application) being transferred to the user and rendered for their use.

He first coined the term in his PhD dissertation to describe an architectural style of network systems (http://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm). This definition implies that we first had web apps, and then Mr. Fielding described how a well-designed version should behave. This also implies that **REST is not a standard, it's an architectural style**. When we implement it, we will use standards of course, but in principle REST is completely protocol agnostic. JSON isn't a part of REST, but theoretically not even HTTP is a part of REST, and that's really theoretical. To be honest, I haven't seen a single RESTful system that doesn't use HTTP; it's pretty much the web's golden standard, so that is what is used, and from this moment on we will assume this. 

Imagine you want to read your favorite newspaper online. You've opened your browser. That browser, that's an HTTP client. You point it to a URI. That's the unique resource identifier. It identifies where the resource lives. By doing that, the browser actually sends an HTTP request to that URI.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/IntroducingRest/Images/Rest1.PNG" />

The server then does some magic and sends an HTTP response message back to the browser. That HTTP response message contains a representation of the page you've navigated to. In our example, that would probably be some HTML and CSS. The browser then interprets that resource representation and shows it. In other words, the browser, our HTTP client, has changed state.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/IntroducingRest/Images/Rest2.PNG" />

Now let's say we click a link in our browser to access a specific article on the newspaper side. That one is again identified by a URI. A new request message is sent to the server, and the server again sends back a representation of the page, the resource. The browser interprets it and changes state. In other words, the client changes state depending on the representation of the resource we're accessing. And that's representational state transfer, or REST.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/IntroducingRest/Images/Rest3.PNG" />

## REST constraints

REST is defined by six constraints, one of which is optional. We should see these constraints as design decisions, and each of those can have both positive and negative impacts. So, for these constraints, the benefits are considered to outweigh the disadvantages. Let's start with the 

**1. Client-server**.

What this does is enforce client-server architecture. A client, or consumer of the API in our lingo, shouldn't be concerned with how data is stored, or how the representation is generated, that's transparent. A server, the API in our lingo, shouldn't be concerned with, for example, the user interface or user state or anything related to how the client is implemented. In other words, client and server can evolve separately.

**2. Statelessness.** This means that the necessary state to handle every request is contained within the request itself. When a client requests a resource, that request contains all the information necessary to service the request. It's one of the constraints that ensures RESTful APIs can scale so easily. We don't have things like server-side session state to keep in mind when scaling up.

**3. Cacheable.** This one states that each response message must explicitly state if it can be cached or not. Like this we can eliminate some client/ server interaction, and at the same time prevent clients from using out-of-date data.

**4. Layered system.** A REST-based solution can be comprised of multiple architectural layers, just as almost all application architectures we use today. These layers can be modified, added, removed, but no one layer can directly access a layer that's beyond the next one. That also means that a client cannot tell whether it's directly connected to the final layer or to another intermediary along the way. REST restricts knowledge to a single layer, which reduces the overall system complexity.

**5. Code on demand (optional)**. This one states that the server can extend or customize client functionality. For example, if your client is a web application, the server can transfer JavaScript code to the client to extend its functionality.

**6. The uniform interface**, it's divided into four sub-constraints. It states that the API and the consumers of the API share one single technical interface. As we're typically working with the HTTP protocol, this interface should be seen as a combination of resource URIs, where we can find resources, HTTP methods, how we can interact with them, like GET and POST, and HTTP media types, like application/json, application/xml, or more specific versions of that that are reflected in the request and response bodies. All of these are standards, by the way. This makes it is a good fit for cross-platform development.

But I said there are four sub-constraints for this uniform interface constraint. The first one is `identification of resources`. It states that individual resources are identified in requests using URIs, and those resources are conceptually separate from the representations that are returned to the client. The server doesn't send an entity from its database because our author resource doesn't necessarily map to an author in one database. Instead it sends the data, typically for RESTful APIs, as JSON, but HTML, XML, or custom formats are also possible. So, if the API supports XML and JSON, both representations are different from the server's internal representation, but it is the same resource.

Let's continue with the second sub-constraint. That's `manipulation of resources through representations`. When a client holds a representation of a resource, including any possible metadata, it has enough information to modify or delete a resource on the server, provided it has permission to do so. To continue with our example, the representation of an author and its metadata in the response message should be enough to successfully update or delete that author, if the API allows that. But what does that mean? Well, if the API supports deleting the resource, the response could include, for example, the URI to the author resource, because that's what's required for deleting it.

The third sub-constraint is the `self-descriptive message sub-constraint`. Each message must include enough information on how to process it. When a consumer requests data from an API, we send a request message, but that message also has headers and a body. If the request body contains a JSON representation of a resource, the message must specify that fact in its headers by including the media type, application/json for example. Like that, the correct parser can be invoked to process the request body, and it can be serialized into the correct class. Same goes for the other way around. Mind you this application/json media type is a simple sample. Media types actually play a very important role in REST.

And `HATEOAS`, well, that's the fourth sub-constraint. It means Hypermedia as the Engine of Application State. This is the one that a lot of RESTful systems fail to implement. Remember that example we had in the beginning of the module when we explained while looking at a newspaper site how the state of our browser changed when we clicked the link? Well, that link, that's hypertext. Hypermedia is a generalization of this. It adds other types like music, images, etc., and it's that hypermedia that should be the engine of application state. In other words, it drives how to consume and use the API. It tells the consumer what it can do with the API. Can I delete this resource? Can I update it? How can I create it, and where can I find it? This really boils down to a self-documenting API, and that documentation can then be used by the client application to provide functionality to the user. It's not an easy constraint.

Not all of these constraints are straightforward to implement, and to be completely correct, an architecture that skips on one of the required constraints or weakens it is considered not conformed to REST, and that immediately means that most APIs that are built today and are called RESTful, well, they aren't really RESTful. But does that mean that they are all bad APIs? In my line of work, most APIs I see when working for clients do not adhere to all these constraints. HATEOAS is a pretty typical one to skip, but that does not mean they're bad APIs. It does mean that you need to know the consequences of deviating from these constraints and understand the potential tradeoffs, which this course should help you with.

## The Richardson Maturity Model

The **Richardson Maturity Model** is a model developed by Leonard Richardson. It grades, if you will, APIs by their RESTful maturity, so it's interesting to look into it as it shows us how we can go from a simple API that doesn't really care about protocol nor standards, to an API that can be considered RESTful.

The first level is **level 0, The Swamp of POX**, or plain-old XML. This level states that the implementing protocol, HTTP, is used for remote interaction. But, we use it just as that and we don't use anything else from the HTTP protocol correctly. So for example, to get some altered data, you send over a POST request to some basic entry point URI, like host/myapi, and in the body you send some XML that contains info on the data you want.

**Level 1 is resources**. From this moment on, multiple URIs are used and each URI is mapped to a specific resource, so it extends on level 0 where there was only 1 URI. For example, we now have a URI, host/api/authors, to get a list of authors, and another one, host/api/authors, followed by an ID, to get the author with that specific ID. However, only one method like POST is still used for all interactions. So, the HTTP methods aren't used as they should be according to the standards. This is already one little part of the uniform interface constraint we see here. From a software design point of view, this means reducing complexity by working with different endpoints instead of one large service endpoint.

Then, we have the **second level, verbs**. To reach a level 2 API, the correct HTTP verbs, like GET, POST, PUT, PATCH, and DELETE, are used as they are intended by the protocol. In the example we see a GET request to get a list of authors, and a POST request containing a resource representation in the body for creating an author. The correct status codes are also included in this level, i.e. use a 200 Ok after a successful GET, a 201 Created after a successful POST, and so on. All of these are covered later on. This again adds to that uniform interface constraint. From a software design point of view, we have just removed unnecessary variation. We're using the same verbs to handle the same types of situations.

And **level 3, that's hypermedia controls**. This means that the API supports HATEOAS, another part of that uniform interface constraint. A sample GET request to the authors resource would then return not only the list of authors, but also links, hypermedia, that drive application state. We're covering this in detail in its own module. From a software design point of view, this means we've introduced discoverability, self-documentation. What is important to know is that according to Roy Fielding, who coined the term REST, a level 3 API is a precondition to be able to talk about a RESTful API. So, this maturity model does not mean there's such a thing as a level 1 RESTful API, a level 2 RESTful API, and so on. It means that there are APIs of different levels, and only when we reach level 3 we can start talking about a RESTful API.
