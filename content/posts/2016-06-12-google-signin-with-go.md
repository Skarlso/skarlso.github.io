+++
author = "hannibal"
categories = ["Go", "Golang", "Web"]
date = "2016-06-12"
title = "How to do Google sign-in with Go"
url = "/2016/06/12/google-signin-with-go"

+++

Hi folks.

Today, I would like to write up a step - by - step guide with a sample web app on how to do Google Sign-In and authorization.

<!--more-->

Let's get started.

**EDIT**: A sample project of this, and [Part 2](http://skarlso.github.io/2016/11/02/google-signin-with-go-part2/), can be found
[here](https://github.com/Skarlso/goquestwebapp) or [here](https://github.com/Skarlso/google-oauth-go-sample).

Setup
=====

Google OAuth token
------------------

First what you need is, to register your application with Google, so you'll get a Token that you can use to authorize later calls to Google services.

You can do that here: [Google Developer Console](https://console.developers.google.com/iam-admin/projects). You'll have to create a new project. Once it's done, click on ```Credentials``` and create an OAuth token. You should see something like this: "To create an OAuth client ID, you must first set a product name on the consent screen.". Go through the questions, like, what type your application is, and once you arrive at stage where it's asking for your application's name -- there is a section asking for redirect URLs; there, write the url you wish to use when authorising your user. If you don't know this yet, don't fret, you can come back and change it later. Do NOT use ```localhost```. If you are running on your own, use http://127.0.0.1:port/whatever.

This will get you a ```client ID``` and a ```client secret```. I'm going to save these into a file which will sit next to my web app. It could be stored more securely, for example, in a database or a mounted secure, encrypted drive, and so and so forth.

Your application can now be identified through Google services.


The Application
===============

Libraries
---------

Google has a nice library to use with OAuth 2.0. The library is available here: [Google OAth 2.0](https://github.com/golang/oauth2). It's a bit cryptic at first, but not to worry. After a bit of fiddling you'll understand fast what it does. I'm also using [Gin](https://github.com/gin-gonic/gin), and Gin's session handling middleware [Gin-Session](https://github.com/gin-gonic/contrib/tree/master/sessions).

Setup - Credentials
-------------------

Let's create a setup which configures your credentials from the file you saved earlier. This is pretty straightforward.

~~~go
// Credentials which stores google ids.
type Credentials struct {
    Cid string `json:"cid"`
    Csecret string `json:"csecret"`
}

func init() {
    var c Credentials
    file, err := ioutil.ReadFile("./creds.json")
    if err != nil {
        fmt.Printf("File error: %v\n", err)
        os.Exit(1)
    }
    json.Unmarshal(file, &c)
}
~~~

Once you have the creds loaded, you can now go on to construct the OAuth client.

Setup - OAuth client
--------------------

Construct the OAuth config like this:

~~~go
conf := &oauth2.Config{
  ClientID:     c.Cid,
  ClientSecret: c.Csecret,
  RedirectURL:  "http://localhost:9090/auth",
  Scopes: []string{
    "https://www.googleapis.com/auth/userinfo.email", // You have to select your own scope from here -> https://developers.google.com/identity/protocols/googlescopes#google_sign-in
  },
  Endpoint: google.Endpoint,
}
~~~

It will give you a struct which you can then use to Authorize the user in the google domain. Next, all you need to do is call ```AuthCodeURL``` on this config. It will give you a URL which redirects to a Google Sign-In form. Once the user fills that out and clicks 'Allow', you'll get back a TOKEN in the ```code``` query parameter and a ```state``` which helps protect against CSRF attacks. Always check if the provided state is the same which you provided with AuthCodeURL. This will look something like this ```http://127.0.0.1:9090/auth?code=4FLKFskdjflf3343d4f&state=lhfu3f983j;asdf```. Small function for this:

~~~go
func getLoginURL(state string) string {
    // State can be some kind of random generated hash string.
    // See relevant RFC: http://tools.ietf.org/html/rfc6749#section-10.12
    return conf.AuthCodeURL(state)
}
~~~

Construct a button which the user can click and be redirected to the Google Sign-In form. When constructing the url, we must do one more thing. Create a secure state token and save it in the form of a cookie for the current user.

Random State and Button construction
------------------------------------

Small piece of code random token:

~~~go
func randToken() string {
	b := make([]byte, 32)
	rand.Read(b)
	return base64.StdEncoding.EncodeToString(b)
}
~~~

Storing it in a session and constructing the button:

~~~go
func loginHandler(c *gin.Context) {
    state = randToken()
    session := sessions.Default(c)
    session.Set("state", state)
    session.Save()
    c.Writer.Write([]byte("<html><title>Golang Google</title> <body> <a href='" + getLoginURL() + "'><button>Login with Google!</button> </a> </body></html>"))
}
~~~

It's not the nicest button I ever come up with, but it will have to do.

User Information
================

After you got the token, you can construct an authorised Google HTTP Client, which let's you call Google related services and retrieve information about the user.

Getting the Client
------------------

Before we construct a client, we must check if the retrieved state is still the same compared to the one we provided. I'm doing this before constructing the client. Together this looks like this:

~~~go
func authHandler(c *gin.Context) {
    // Check state validity.
    session := sessions.Default(c)
    retrievedState := session.Get("state")
    if retrievedState != c.Query("state") {
        c.AbortWithError(http.StatusUnauthorized, fmt.Errorf("Invalid session state: %s", retrievedState))
        return
    }
    // Handle the exchange code to initiate a transport.
  	tok, err := conf.Exchange(oauth2.NoContext, c.Query("code"))
  	if err != nil {
  		c.AbortWithError(http.StatusBadRequest, err)
          return
  	}
    // Construct the client.
    client := conf.Client(oauth2.NoContext, tok)
    ...
~~~

Obtaining information
---------------------

Our next step is to retrieve information about the user. To achieve this, call Google's API with the authorised client. The code for that is:

~~~go
...
resp, err := client.Get("https://www.googleapis.com/oauth2/v3/userinfo")
if err != nil {
    c.AbortWithError(http.StatusBadRequest, err)
    return
}
defer resp.Body.Close()
data, _ := ioutil.ReadAll(resp.Body)
log.Println("Resp body: ", string(data))
...
~~~

And this will yield a body like this:

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

Parse it, and you've got an email which you can store somewhere for registration purposes. At this point, your user is not yet Authenticated. For that, I'm going to post a second post, which describes how to go on. Retrieving the stored email address, and user session handling with Gin and MongoDB.

Putting it all together
=======================

~~~go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "encoding/json"
    "io/ioutil"
    "fmt"
    "log"
    "os"
    "net/http"

    "github.com/gin-gonic/contrib/sessions"
    "github.com/gin-gonic/gin"
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

// Credentials which stores google ids.
type Credentials struct {
    Cid     string `json:"cid"`
    Csecret string `json:"csecret"`
}

// User is a retrieved and authentiacted user.
type User struct {
    Sub string `json:"sub"`
    Name string `json:"name"`
    GivenName string `json:"given_name"`
    FamilyName string `json:"family_name"`
    Profile string `json:"profile"`
    Picture string `json:"picture"`
    Email string `json:"email"`
    EmailVerified string `json:"email_verified"`
    Gender string `json:"gender"`
}

var cred Credentials
var conf *oauth2.Config
var state string
var store = sessions.NewCookieStore([]byte("secret"))

func randToken() string {
	b := make([]byte, 32)
	rand.Read(b)
	return base64.StdEncoding.EncodeToString(b)
}

func init() {
    file, err := ioutil.ReadFile("./creds.json")
    if err != nil {
        log.Printf("File error: %v\n", err)
        os.Exit(1)
    }
    json.Unmarshal(file, &cred)

    conf = &oauth2.Config{
        ClientID:     cred.Cid,
        ClientSecret: cred.Csecret,
        RedirectURL:  "http://127.0.0.1:9090/auth",
        Scopes: []string{
            "https://www.googleapis.com/auth/userinfo.email", // You have to select your own scope from here -> https://developers.google.com/identity/protocols/googlescopes#google_sign-in
        },
        Endpoint: google.Endpoint,
    }
}

func indexHandler(c *gin.Context) {
    c.HTML(http.StatusOK, "index.tmpl", gin.H{})
}

func getLoginURL(state string) string {
    return conf.AuthCodeURL(state)
}

func authHandler(c *gin.Context) {
    // Handle the exchange code to initiate a transport.
    session := sessions.Default(c)
    retrievedState := session.Get("state")
    if retrievedState != c.Query("state") {
        c.AbortWithError(http.StatusUnauthorized, fmt.Errorf("Invalid session state: %s", retrievedState))
        return
    }

	tok, err := conf.Exchange(oauth2.NoContext, c.Query("code"))
	if err != nil {
		c.AbortWithError(http.StatusBadRequest, err)
        return
	}

	client := conf.Client(oauth2.NoContext, tok)
	email, err := client.Get("https://www.googleapis.com/oauth2/v3/userinfo")
    if err != nil {
		c.AbortWithError(http.StatusBadRequest, err)
        return
	}
    defer email.Body.Close()
    data, _ := ioutil.ReadAll(email.Body)
    log.Println("Email body: ", string(data))
    c.Status(http.StatusOK)
}

func loginHandler(c *gin.Context) {
    state = randToken()
    session := sessions.Default(c)
    session.Set("state", state)
    session.Save()
    c.Writer.Write([]byte("<html><title>Golang Google</title> <body> <a href='" + getLoginURL(state) + "'><button>Login with Google!</button> </a> </body></html>"))
}

func main() {
    router := gin.Default()
    router.Use(sessions.Sessions("goquestsession", store))
    router.Static("/css", "./static/css")
    router.Static("/img", "./static/img")
    router.LoadHTMLGlob("templates/*")

    router.GET("/", indexHandler)
    router.GET("/login", loginHandler)
    router.GET("/auth", authHandler)

    router.Run("127.0.0.1:9090")
}
}
~~~

This is it folks. I hope this helped. Any comments or advices are welcomed.

Google API Documentation
========================

The documentation to this whole process, and MUCH more information can be found here: [Google API Docs](https://developers.google.com/identity/protocols/OAuth2).

Thanks for reading,
Gergely.
