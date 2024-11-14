+++
author = "hannibal"
categories = ["blog"]
date = "2015-12-07"
title = "Welcome To My New Blog"
url = "/2015/12/07/welcome-to-my-new-blog/"

+++

Hello Folks
===========

Welcome to my new blog. I decided to move away for a number of reasons, but setting up a static page blog site is very cool if you don't directly use a database. Since posts are just posts and I have a different way of hosting images, this really was just a matter of time.

And [Hugo](https://gohugo.io/) / Github pages provided the tools which made this move possible.

Also, I love writing this post in Markdown. I always liked the formatting rules of it, so this is quiet the blast.

Code will look a little more readble now as well:

~~~go
func main() {
    handlerChain := alice.New(Logging, PanicHandler)
    router := mux.NewRouter().StrictSlash(true)
    router.Handle("/create", handlerChain.ThenFunc(createIssue)).Methods("POST")
    router.Handle("/", handlerChain.ThenFunc(renderMainPage)).Methods("GET")
    router.PathPrefix("/css/").Handler(http.StripPrefix("/css/", http.FileServer(http.Dir("./css"))))
    log.Printf("Starting server to listen on port: 8989...")
    http.ListenAndServe(":8989", router)
}
~~~

Much easier on the eyes. And linking is a breeze as well.

Things to notice
----------------

There is now a content on the side which will list the sections in a post. And there is an estimated read timer in the post's title. It takes average reading speed and wordcount into account.

Anyhow, thanks for joining me in the new realm, and happy reading!
Gergely.
