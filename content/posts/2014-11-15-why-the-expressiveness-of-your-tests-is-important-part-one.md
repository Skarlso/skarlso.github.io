+++
author = "hannibal"
categories = ["groovy", "programming", "testing"]
date = "2014-11-15"
title = "Why the expressiveness of your Tests is important – Part One"
url = "/2014/11/15/why-the-expressiveness-of-your-tests-is-important-part-one/"

+++

Hello Everybody.

This time I'd like to write about the expressiveness of a Test. I think that it's very important to write understandable and expressive tests. In older times I was studying novel writing. There is a rule which a novel needs to follow. It goes something like this: "A novel needs to lead its reader and make him understand in the simplest way what's going on, with whom and why?". In other words, it's not a puzzle. It should be obvious what the test is trying to do and it should not require the reader to try and solve it in order to understand it.

<!--more-->

I'm planning this as a series since there are multiple problems with a test I can talk about here.

**Geb Tests**

Example:

~~~groovy
    def "login to the home page"() {
        given: "I am at homepages"
            go "http://localhost:8080/home"
            at HomePage

        when: "I entir my credential"
            filloutLoginForm("username", "password")
            at MyAccountPage

        then: "I can accass my wallet"
            openWallet()
            waitFor { contains("Balance") }
    }
~~~

Now, read this test. It doesn't really make any sense at the first read, right? You need to actually think what is going on there. Of course if you read it slow enough you'll get what it's trying to do. But you don't know what fillform does. Apparently it also submits the form because after fillform you are suddenly at MyAccountPage.

There are several things wrong with this one, let's start with the pageobject.

**PageObjects**

At and toAt return page objects. We can use that to actually make the calling explicit and make it more readable and identify where a function comes from.

~~~groovy
    def "login to the home page"() {
        given: "I am at homepages"
            go "http://localhost:8080/home"
            HomePage homePage = at HomePage

        when: "I entir my credential"
            homePage.filloutLoginForm("username", "password")
            MyAccountPage myAccountPage = at MyAccountPage

        then: "I can accass my wallet"
            myAccountPage.openWallet()
            waitFor { contains("Balance") }
    }
~~~

This reads much better now. You know where the function is coming from and your IDE will not go nuts from things it can't find. And you have autocompletion so there is no fear that you simply mistype something.

**Side effects**

Next step, let's remove some of the side effects.

~~~groovy
    def "login to the home page"() {
        given: "I am at homepages"
            go "http://localhost:8080/home"
            HomePage homePage = at HomePage

        when: "I entir my credential"
            homePage.filloutLoginForm("username", "password")
            homePage.submitLoginForm()
            MyAccountPage myAccountPage = at MyAccountPage

        then: "I can accass my wallet"
            myAccountPage.openWallet()
            waitFor { myAccountPage.accountIsDisplayed() }
    }
~~~

Now this is again much better. There are no steps left out. And you can test now the FillForm and the submit independently. Like, submitting the form without filling it out! Or filling it out and not submiting it. Reads better, is explicit, more easy to understand.

And the last one for today:

**Grammar**

I wonder if you noticed it. The grammar is a little bit off in the tests. A small mistake here and there. You might think that, who cares? That's a very bad thought. I think the correct grammar reflects caring. It reflects that we thought about this test and that we thought about the quality of it. Because it means that after you wrote it, you actually re-read the test to make sure it's understandable and readable.

So let us correct that:

~~~groovy
    def "login to the home page"() {
        given: "I am at homepages"
            go "http://localhost:8080/home"
            HomePage homePage = at HomePage

        when: "I entir my credential"
            homePage.filloutLoginForm("username", "password")
            homePage.submitLoginForm()
            MyAccountPage myAccountPage = at MyAccountPage

        then: "I can accass my wallet"
            myAccountPage.openWallet()
            waitFor { myAccountPage.accountIsDisplayed() }
    }
    def "As a player I can log in to check my account."() {
        given: "I am at the homepage"
            go "http://localhost:8080/home"
            HomePage homePage = at HomePage

        when: "I enter my log in credentials."
            homePage.filloutLoginForm("username", "password")
            homePage.submitLoginForm()
            MyAccountPage myAccountPage = at MyAccountPage

        then: "I'm directed to my account page."
            myAccountPage.openWallet()
            waitFor { myAccountPage.accountIsDisplayed() }
    }
~~~

I also took the liberty of re-phrasing some of the text so that it shows what the test is about and what the user really would like to achieve here. Now try reading that last one. Does it make more sense? Did you understand it at first go? Did it read like a good story?

There is a coding practice which goes something like this: "Good code is code which doesn't surprise you as you read it." Which means the exact thing happens which you thought of would happen. I think that applies to tests as well. The steps of the test shouldn't come to you as a surprise. Especially if you know what the application is supposed to do.

So that's all for today folks. Thank you for reading! If you have a nasty test which you would like me to dissect and make it better and human readable, please share it in the comment section and I will do my best to come up with a good solution for it.

And as always,

Have a nice day!

Gergely.