# RESTful-API

## Introducing REST

REST isn't just about building an API which consists of a few HTTP services that return JSON.

Roy Fielding definition: **Representational state transfer** is intended to evoke an image of how a well-designed web application behaves: a network of web pages (a virtual state-machine) where the user progresses through an application by selecting links (state-transitions), resulting in the next page (representing the next state of the application) being transferred to the user and rendered for their use.

He first coined the term in his PhD dissertation to describe an architectural style of network systems (http://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm). This definition implies that we first had web apps, and then Mr. Fielding described how a well-designed version should behave. This also implies that REST is not a standard, it's an architectural style. When we implement it, we will use standards of course, but in principle REST is completely protocol agnostic. JSON isn't a part of REST, but theoretically not even HTTP is a part of REST, and that's really theoretical. To be honest, I haven't seen a single RESTful system that doesn't use HTTP; it's pretty much the web's golden standard, so that is what is used, and from this moment on we will assume this. But I did want to mention this to drive the point home that it is an architectural style and nothing more than that. It's up to us to fill in the blanks with the protocols and standards we want to use.

Now, as the style is intended to evoke how a well-designed web application behaves, we can use a web application to explain that huge definition we just saw. Imagine you want to read your favorite newspaper online. You've opened your browser. That browser, that's an HTTP client. You point it to a URI. That's the unique resource identifier. It identifies where the resource lives. By doing that, the browser actually sends an HTTP request to that URI.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/Images/Rest1.PNG" />

The server then does some magic and sends an HTTP response message back to the browser. That HTTP response message contains a representation of the page you've navigated to. In our example, that would probably be some HTML and CSS. The browser then interprets that resource representation and shows it. In other words, the browser, our HTTP client, has changed state.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/Images/Rest2.PNG" />

Now let's say we click a link in our browser to access a specific article on the newspaper side. That one is again identified by a URI. A new request message is sent to the server, and the server again sends back a representation of the page, the resource. The browser interprets it and changes state. In other words, the client changes state depending on the representation of the resource we're accessing. And that's representational state transfer, or REST.

<img src="https://github.com/KiraDiShira/RESTful-API/blob/master/Images/Rest3.PNG" />

## REST constraints

REST is defined by six constraints, one of which is optional. We should see these constraints as design decisions, and each of those can have both positive and negative impacts. So, for these constraints, the benefits are considered to outweigh the disadvantages. Let's start with the **client-server constraint**.

What this does is enforce client-server architecture. A client, or consumer of the API in our lingo, shouldn't be concerned with how data is stored, or how the representation is generated, that's transparent. A server, the API in our lingo, shouldn't be concerned with, for example, the user interface or user state or anything related to how the client is implemented. In other words, client and server can evolve separately.

**The second constraint is statelessness.** This means that the necessary state to handle every request is contained within the request itself. When a client requests a resource, that request contains all the information necessary to service the request. It's one of the constraints that ensures RESTful APIs can scale so easily. We don't have things like server-side session state to keep in mind when scaling up.

Then, we have the **cacheable constraint**. This one states that each response message must explicitly state if it can be cached or not. Like this we can eliminate some client/ server interaction, and at the same time prevent clients from using out-of-date data.

**Layered system contraint.** A REST-based solution can be comprised of multiple architectural layers, just as almost all application architectures we use today. These layers can be modified, added, removed, but no one layer can directly access a layer that's beyond the next one. That also means that a client cannot tell whether it's directly connected to the final layer or to another intermediary along the way. REST restricts knowledge to a single layer, which reduces the overall system complexity.

And then there's the optional **code on demand constraint**. This one states that the server can extend or customize client functionality. For example, if your client is a web application, the server can transfer JavaScript code to the client to extend its functionality.

**The uniform interface constraint**, it's divided into four sub-constraints. It states that the API and the consumers of the API share one single technical interface. As we're typically working with the HTTP protocol, this interface should be seen as a combination of resource URIs, where we can find resources, HTTP methods, how we can interact with them, like GET and POST, and HTTP media types, like application/json, application/xml, or more specific versions of that that are reflected in the request and response bodies. All of these are standards, by the way. This makes it is a good fit for cross-platform development.

But I said there are four sub-constraints for this uniform interface constraint. The first one is `identification of resources`. It states that individual resources are identified in requests using URIs, and those resources are conceptually separate from the representations that are returned to the client. The server doesn't send an entity from its database because our author resource doesn't necessarily map to an author in one database. Instead it sends the data, typically for RESTful APIs, as JSON, but HTML, XML, or custom formats are also possible. So, if the API supports XML and JSON, both representations are different from the server's internal representation, but it is the same resource.

Let's continue with the second sub-constraint. That's `manipulation of resources through representations`. When a client holds a representation of a resource, including any possible metadata, it has enough information to modify or delete a resource on the server, provided it has permission to do so. To continue with our example, the representation of an author and its metadata in the response message should be enough to successfully update or delete that author, if the API allows that. But what does that mean? Well, if the API supports deleting the resource, the response could include, for example, the URI to the author resource, because that's what's required for deleting it.

The third sub-constraint is the `self-descriptive message sub-constraint`. Each message must include enough information on how to process it. When a consumer requests data from an API, we send a request message, but that message also has headers and a body. If the request body contains a JSON representation of a resource, the message must specify that fact in its headers by including the media type, application/json for example. Like that, the correct parser can be invoked to process the request body, and it can be serialized into the correct class. Same goes for the other way around. Mind you this application/json media type is a simple sample. Media types actually play a very important role in REST.

And `HATEOAS`, well, that's the fourth sub-constraint. It means Hypermedia as the Engine of Application State. This is the one that a lot of RESTful systems fail to implement. Remember that example we had in the beginning of the module when we explained while looking at a newspaper site how the state of our browser changed when we clicked the link? Well, that link, that's hypertext. Hypermedia is a generalization of this. It adds other types like music, images, etc., and it's that hypermedia that should be the engine of application state. In other words, it drives how to consume and use the API. It tells the consumer what it can do with the API. Can I delete this resource? Can I update it? How can I create it, and where can I find it? This really boils down to a self-documenting API, and that documentation can then be used by the client application to provide functionality to the user. It's not an easy constraint.
