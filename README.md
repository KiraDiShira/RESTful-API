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

