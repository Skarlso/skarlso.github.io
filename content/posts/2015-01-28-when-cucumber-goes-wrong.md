+++
author = "hannibal"
categories = ["testing"]
date = "2015-01-28"
tags = ["cucumber"]
title = "When cucumber goes wrong"
url = "/2015/01/28/when-cucumber-goes-wrong/"

+++

Hi,

Let's face the horrible truth:

**It's rare / never happens that a manager / scrum master / product owner actually reads your cucumber test cases.**

Back in the old days, this was one of the selling points of human readable tests and DSLs. It sounds nice and I'm sure in a utopia it also works.

BDD is a very nice approach to write tests if used in a correct way. And I can relate that at some point, a manager or the product owner, actually writes up a draft of the tests. But that enthusiasm very rarely stays for the rest of the project.

Especially when you get to the point where your Cucumber test cases start to look something like this:

~~~
Scenario: User list
  Given I post to "/users.json" with:
    """
    {
      "first_name": "Steve",
      "last_name": "Richert"
    }
    """
  And I keep the JSON response at "id" as "USER_ID"
  When I get "/users.json"
  Then the JSON response should have 1 user
  And the JSON response at "0" should be:
    """
    {
      "id": %{USER_ID},
      "first_name": "Steve",
      "last_name": "Richert"
    }
    """
~~~

If a product owner reads this, his reaction will be like: "What the hell is this? What's users.json? Why is it there? Why should I even care? What's a JSON response? Why should it match with the request? And what, if I keep the id at USER_ID? Huh?"

It's easy to get overwhelmed by things like this scenario when you start introducing actors into your tests and payloads to your public API. And suddenly you'll end up with cucumber features which no other will be able to understand but the person who wrote it.

I'm a little bit skeptic that it ever worked as intended. Sure, for a little while. But the dynamic nature of tests will surface soon enough. You can't hide it forever.

The above example, if the payload and user would be hidden in a reusable code fragment behind the implementation, would look a bit more readable:

~~~
Scenario: User list
  Given I post to user list with data
  | firstname | Steve |
  | lastname  | Richert |
  When I get a response from the SUT
  Then the response should have the same user
~~~

See? Easier to understand. I don't care about the payload. I don't care about the user ID, in fact, I would rather see this test as a unit test somewhere deep down in the bowls of the system. Although I can understand that you want a set of automated UATs.

I'm sure Cucumber has a couple of success stories behind his back, I just didn't happen to come across them as of late. But please, if you have one, share it with me so I can rest easily.

Gergely.