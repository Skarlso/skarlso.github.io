+++
author = "hannibal"
categories = ["java", "knowledge", "programming"]
date = "2012-06-13"
tags = ["exception"]
title = "Don’t throw Exception"
url = "/2012/06/13/dont-throw-exception/"

+++

Hi.

Today I want to talk about a common problem in many frameworks I encountered over the course of my carrier as a Java dev / automation engineer, whatnot.

Throwing Exceptions. That is in your method you have something like this:

~~~java
public void insertMethodNameHere(String param) throws Exception {}
~~~

This is bad. And I will tell you in short forms why.

**Hides exceptions**

This one should be obvious. When a method throws exception you can never be sure what kind of exceptions it handles. It will hide what problems it can encounter. It will hide possible threats and will generally mean nothing to you when it fails.

**Hides functionality**

One of the things you can do will come up with a method that throws at least six exception. Well guess what. That will tell you that the method has DESIGN ISSUES! The first rule of software development is that a method should do only one thing! Well if it throws six exceptions chances are it does more then one.

**Hard to debug**

You wont have a meaning full exception if it fails immediately. You will have to go through lines of codes and stack traces to find out what the hell happened and what threw what kind of exception where. That is just simply stupid. Why give yourself a hard time?

**So what to do instead?**

**Meaning full exceptions**

If you have to throw. Throw meaning full exceptions. Things like: LoginFailedExpcetion(String username, String password); In the message write:

~~~java
String.format("Failed login with username: %s; password: %s", username, password);
~~~

When this fails you will immediately know what happened without miles of stack trace to run through.

When you go ahead and start to re-factor your code to handle exceptions properly you will end up with methods throwing six - seven exceptions. Don't be afraid. That means that you finally are able to see that your code is doing many things that it is not supposed to do. Start by extracting bits and pieces of your code into smaller segments that throw a max of two exceptions. Write unit tests around the function and start running them as you re-factor. See what fails so you can track your changes as you go.

A general good advice is that your method should throw a maximum of **TWO** exception. If you have more, you are doing something more then you should. Refactor / Extract that bit into a smaller function.

**Handling exceptions**

What you really want to do is create a Layer that you will be using to capture and handle exceptions. You can than take that layer and hide it deep deep into your framework so that you never ever see it again. Just like with switch.

As you go you will have layers of layers of exceptions. You will have features that depend on each other and talk to each other in some ways.

Meaning full exception will help you find out what broke why. For example you have Login that throws a Login exception. On top of that you have an order that handles OrderFailedException. You will have an easy time seeing that the order failed because the login failed. Because you handled your login exception in the Login Layer. And you handled your OrderException in the order layer. There are no mixes. Keep it clean and keep it where it belongs.

**Conclusion**

Exceptions are part of Java just like String or int or long is. You use these wisely so why don't you apply that same logic to your Exception handling? Don't be afraid of having 20-25 exception classes. Group them together or leave them in their respective packages or have them in a deep layer but HAVE THEM. They WILL save time and time is always money. So they will save you money in the end when an error occurs. And errors will always occur.

Thanks for reading,

Gergely.