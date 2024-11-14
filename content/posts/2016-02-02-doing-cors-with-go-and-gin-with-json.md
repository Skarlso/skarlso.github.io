+++
author = "hannibal"
categories = ["Go", "Programming", "Performance"]
date = "2016-02-02"
title = "Doing CORS in Go with Gin and JSON"
url = "/2016/02/02/doing-cors-in-go-with-gin-and-json"

+++

Basics
------

Hello folks.

This will be a quick post about how to do CORS with jQuery, Gin in Go with a very simple ajax GET and Json.

I'm choosing JSON here because basically I don't really like JSONP. And actually, it's not very complicated to do CORS, it's just hidden enough so that it doesn't become transparent.

First, what is CORS? It's Cross-Platform Resource Sharing. It has been invented so that without your explicit authorization in the header of a request, Javascript can't reach outside of your domain and be potentially harmful to your visitors.

Now, suppose you have an architecture like this.

![Architecture](/img/architecture.png)

You have multiple agents sitting on multiple nodes. You have one central server, and you have multiple front-ends. Everybody can only talk to the Server but the server does talk to everyone. You would like to have a dynamic front-end and would like to display data with ajax calls. Since your front-end sits on a different server, you will have to do something about CORS. This is how I solved it...

I'm using [Gin](https://github.com/gin-gonic/gin) for my REST service for [Dockmaster](https://github.com/Skarlso/dockmaster2). For this two work, you need to adjust two component.

Server
------

There is thing called a Preflight-Check. In essence, the preflight check is sent BEFORE the actual request to check if the next request is allowed to go out of the domain. The preflight check is sent to the same URI just with OPTIONS method. In order to tell the caller that the next one will be safe, you need three things.

First, you need to set two Headers.
#1 -> Access-Control-Allow-Origin to "*".
#2 -> Access-Control-Allow-Headers to "access-control-allow-origin, access-control-allow-headers".

These are the minimum headers you can set. If you allow Access-Control-Allow-Origin you also have to allow it in the headers section because the next request will expect it to be there. Also, note here that setting Origin to * is only recommended in development environment. Otherwise it should be set to whatever your domain is.

Second, you need to respond to the OPTIONS method with a 200. In order to do that, I added a simple rule with the same end-point but with OPTIONS.

~~~go
func main() {
    router := gin.Default()
    v1 := router.Group(APIBASE)
    {
        v1.GET("/list", listContainers)
        v1.POST("/add", addContainers)
        v1.POST("/delete", deleteContainers)
        v1.GET("/inspect/:agentID/:containerID", inspectContainer)
        v1.OPTIONS("/inspect/:agentID/:containerID", preflight)
        v1.POST("/stopAll", stopAll)
        v1.OPTIONS("/stopAll", preflight)
    }
    router.Run(":8989")
}

func preflight(c *gin.Context) {
    c.Header("Access-Control-Allow-Origin", "*")
    c.Header("Access-Control-Allow-Headers", "access-control-allow-origin, access-control-allow-headers")
    c.JSON(http.StatusOK, struct{}{})
}
~~~

You can see that the preflight method is there for two end-points. I added it to those end-points which will reach over the domain. The others are all local, thus they don't need that. This leads to a little duplication, but that is fine. I have a very fine control over what actually is allowed to go outside of the domain.

So, how do we call this?


Frontend
--------

In the front-end's web layout, I'm doing an Ajax GET, which looks like this:

~~~javascript
                $.ajax({
                    url: 'http://localhost:8989/api/1/inspect/'+data.agentid+'/'+data.id,
                    type: 'GET',
                    dataType:"json",
                    headers: {"Access-Control-Allow-Origin": "*", "Access-Control-Allow-Headers": "access-control-allow-origin, access-control-allow-headers"},
                    processData: false,
                    success: function(data) {
                        var json = JSON.stringify(data, null, 4)
                        independentPopup.html("<pre >"+json+"</pre>");
                        $(link).after(independentPopup);
                    }
                });
~~~

After the headers are set, the request will work nicely.

Y U No Middleware?
------------------

And now you could say that, why not just have a middleware which will always accept OPTIONS for every end-point. Because I like it better this way. Some would argue that this is too granular, but fact is, that in my opinion, this is more readable and immediatly visible. However, if you DO want to do that, you have several options to your disposal.

[Cors Basic Http Middleware](https://github.com/itsjamie/gin-cors) and for Gin [Gin CORS Middleware](https://github.com/itsjamie/gin-cors).


Summary
-------

This is it. You can see the code in its entirety on Github. Have a better idea on how to do it? Please! Do not hesitate to share. I always like to learn.

Thank you for reading!

And as always,
Have a nice day!

Gergely.
