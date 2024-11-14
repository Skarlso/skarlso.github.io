+++
author = "hannibal"
categories = ["Go", "Golang", "Web"]
date = "2016-11-02"
title = "How to do Google Sign-In with Go - Part 2"
url = "/2016/11/02/google-signin-with-go-part2"

+++

# Intro

Hi Folks.

This is a follow up on my previous post about Google Sign-In. In this post we will discover what to do with the information retrieved in the first encounter, which you can find here: [Google Sign-In Part 1](http://skarlso.github.io/2016/06/12/google-signin-with-go/).

# Forewords

## The Project

Everything I did in the first post, and that I'm going to do in this example, can be found in this project: [Google-OAuth-Go-Sample](https://github.com/Skarlso/google-oauth-go-sample).

Just to recap, we left off previously on the point where we successfully obtained information about the user, with a secure token and a session initiated with them. Google nicely enough provided us with some details which we can use. This information was in JSON format and looked something like this:

~~~json
{
  "sub": "1111111111111111111111",
  "name": "Your Name",
  "given_name": "Your",
  "family_name": "Name",
  "profile": "https://plus.google.com/1111111111111111111111",
  "picture": "https://lh3.googleusercontent.com/asdfadsf/AAAAAAAAAAI/Aasdfads/Xasdfasdfs/photo.jpg",
  "email": "your@gmail.com",
  "email_verified": true,
  "gender": "male"
}
~~~

In my example, to keep things simple, I will use the email address since that has to be unique in the land of Google. You could assign an ID to the user, and you could complicate things even further, but my goal is not to write an academic paper about cryptography here.

# Implementation

## Making something useful out of the data

In order for the app to recognise a user it must save some data about the user. I'm doing that in MongoDB right now, but that could be any form of persistence layer, like, SQLite3, BoltDB, PostgresDB, etc.

### After successful user authorization

Once the user used google to provide us with sufficient information about him/herself, we can retrieve data about that user from our records. The data could be anything that is linked to our unique identifier like: Character Profile, Player Information, Status, Last Logged-In, etcetc. For this, there are two things that need to happen after authorization: Save/Load user information and initiate a session.

The session can be in the form of a cookie, or a Redis storage, or URL re-writing. I'm choosing a cookie here.

### Save / Load user information

All I'm doing is a simple, *returning / new* user handling. The concept is simple. If the email isn't saved, we save it. If it's saved, we set a logic to our page render to greet the returning user.

In the `AuthHandler` I'm doing the following:

~~~go
...
seen := false
db := database.MongoDBConnection{}
if _, mongoErr := db.LoadUser(u.Email); mongoErr == nil {
    seen = true
} else {
    err = db.SaveUser(&u)
    if err != nil {
        log.Println(err)
        c.HTML(http.StatusBadRequest, "error.tmpl", gin.H{"message": "Error while saving user. Please try again."})
        return
    }
}
c.HTML(http.StatusOK, "battle.tmpl", gin.H{"email": u.Email, "seen": seen})
...
~~~

Let's break this down a bit. There is a db connection here, which calls a function that either returns an error, or it doesn't. If it doesn't, that means we have our user. If it does, it means we have to save the user. This is a very simple case (disregard for now, that the error could be something else as well (If you can't get passed that, you could type check the error or check if the returned record contains the requested user information instead of checking for an error.)).

The template is than rendered depending on the `seen` boolean like this:

~~~html
<!DOCTYPE html>
<link rel="icon"
      type="image/png"
      href="/img/favicon.ico" />
<html>
  <head>
    <link rel="stylesheet" href="/css/main.css">
  </head>
  <body>
    {{if .seen}}
        <h1>Welcome back to the battlefield '{{ .email }}'.</h1>
    {{else}}
        <h1>Welcome to the battlefield '{{ .email }}'.</h1>
    {{end}}
  </body>
</html>
~~~

You can see here, that if `seen` is *true* the header message will say: "Welcome *back*...".

### Initiating a session ###

When the user is successfully authenticated, we activate a session so that the user can access pages that require authorization. Here, I have to mention that I'm using [Gin](https://github.com/gin-gonic/gin), so restricted end-points are made with groups which require a middleware.

As I mentioned earlier, I'm using cookies as session handlers. For this, a new session store has to be created with some secure token. This is achieved with the following code fragments ( note that I'm using a Gin session middleware which uses gorilla's session handler located here: [Gin-Gonic(Sessions)](https://github.com/gin-gonic/contrib)):

~~~go
// RandToken in handlers.go:
// RandToken generates a random @l length token.
func RandToken(l int) string {
	b := make([]byte, l)
	rand.Read(b)
	return base64.StdEncoding.EncodeToString(b)
}

// quest.go:
// Create the cookie store in main.go.
store := sessions.NewCookieStore([]byte(handlers.RandToken(64)))
store.Options(sessions.Options{
    Path:   "/",
    MaxAge: 86400 * 7,
})

// using the cookie store:
router.Use(sessions.Sessions("goquestsession", store))
~~~

After this `gin.Context` lets us access this session store by doing `session := sessions.Default(c)`. Now, create a session variable called `user-id` like this:

~~~go
session.Set("user-id", u.Email)
err = session.Save()
if err != nil {
    log.Println(err)
    c.HTML(http.StatusBadRequest, "error.tmpl", gin.H{"message": "Error while saving session. Please try again."})
    return
}
~~~

Don't forget to `save` the session. ;) That is it. If I restart the server, the cookie won't be usable any longer, since it will generate a new token for the cookie store. The user will have to log in again. **Note**: It might be that you'll see something like this, from `session`: `[sessions] ERROR! securecookie: the value is not valid`. You can ignore this error.

## Restricting access to certain end-points with the auth Middleware™

Now, that our session is alive, we can use it to restrict access to some part of the application. With Gin, it looks like this:

~~~go
authorized := router.Group("/battle")
authorized.Use(middleware.AuthorizeRequest())
{
    authorized.GET("/field", handlers.FieldHandler)
}
~~~

This creates a grouping of end-points under `/battle`. Which means, everything under `/battle` will only be accessible if the middleware passed to the `Use` function calls the next handler in the chain. If it aborts the call chain, the end-point will not be accessible. My middleware is pretty simple, but it gets the job done:

~~~go
// AuthorizeRequest is used to authorize a request for a certain end-point group.
func AuthorizeRequest() gin.HandlerFunc {
	return func(c *gin.Context) {
		session := sessions.Default(c)
		v := session.Get("user-id")
		if v == nil {
			c.HTML(http.StatusUnauthorized, "error.tmpl", gin.H{"message": "Please log in."})
			c.Abort()
		}
		c.Next()
	}
}
~~~

Note, that this only check if `user-id` is set or not. That's certainly not enough for a secure application. Its only supposed to be a simple example of the mechanics of the auth middleware. Also, the session usually contains more than one parameter. It's more likely that it contains several variables, which describe the user including a state for CORS protection. For CORS I'd recommend using [rs/cors](https://github.com/rs/cors).

If you would try to access http://127.0.0.1:9090/battle/field without logging in, you'd be redirected to an `error.tmpl` with the message: **Please log in.**.

# Final Words

That's pretty much it. Important parts are:

 + Saving the right information
 + Secure cookie store
 + CORS for sessions
 + Checks of the users details in the cookie
 + Authorised end-points
 + Session handling

Any questions, remarks, ideas, are very welcomed in the comment section. There are plenty of very nice Go frameworks which do Google OAuth2 out of the box. I recommend using them, as they save you a lot of legwork.

Thank you for reading!
Gergely.
