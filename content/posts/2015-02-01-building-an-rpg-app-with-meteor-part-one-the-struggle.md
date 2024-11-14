+++
author = "hannibal"
categories = ["JavaScript", "programming"]
date = "2015-02-01"
tags = ["meteorjs"]
title = "Building an RPG App with Meteor – Part One – The struggle"
url = "/2015/02/01/building-an-rpg-app-with-meteor-part-one-the-struggle/"

+++

In my previous post, I was getting ready to enjoy some time with the JavaScript web framework Meteor.

This time I would like to bring it to a bit of overdrive. See, how re-factoring works on a larger scale model with multiple pages. And how it can organize assets, such as, images, multiple CSS, some plugins, you know, ordinary web stuff.

Let's dive in.

<!--more-->

I'm planning this to be a series of posts as I'm going along building up my RPG app. Let's define the rules.

# In the beginning

###

### Rules

&nbsp;

#### Inventory

&nbsp;

Our main character will have a basic inventory. He will have space to carry stuff around and a body to put stuff on. One ring on each hand, one weapon in each hand, helmet, armour, legs, and a necklace. That's it. For simplicities sake. The game mechanics will be like those old books which you could play, Fighting RPG Books, like the one Ian Livingstone was writing. This is one of my favourites; Robot commando:

#### Stats

A very basic stat system.

  * Strength
  * Agility
  * Constitution
  * Intelligence
  * Magic

&nbsp;

#### Fighting

&nbsp;

A very basic fighting system with the possibility of casting magic which, for simplicity, will count as attacks and can be dodged based on agility.

Let's say we have dice throwing with a couple of 6 sided ones. So X * 6 sided dice. Dodging will require agility, HP is defined by constitution, Intelligence will help in puzzles which require a throw against intelligence, Magic will define Mana Points.

Simple, right?

### Design

I'm not much of a front-end developer, so I don't really care about how it will look like. I'll try to squeeze in some very basic stuff, like ordering, but that's it.

### Game Play

&nbsp;

Basically there will be a story which can be loaded threw a JSON structured file. The file will hold information about what a current page has. The probable things a page can contain at any given time:

  * Current location description
  * Selectable proceed location ( page number )
  * Enemy -> Fight ( Might contain an option to not to attack the beast )
  * Riddle -> Solving it is determined by a throw against intelligence
  * Trap -> Springing it is determined by a throw against agility
  * Lootable items
  * Death

All of the above define an action that a player can, or HAS to take. If there is no ability to choose the player has to proceed as the page requests it. That might be easier to do if I just say if there is only one possible choose it's choosen automatically for you.

# Implementation

&nbsp;

I'll be using Meteor which is based on Node and MongoDB. Hence, my stuff will be in mongoDB. I have a fair knowledge of how mongodb works, I'll write down my progress as I go along.

Everything I'll do is of course under version control and can be followed here:

<a href="https://github.com/Skarlso/coolrpgapp" target="_blank">https://github.com/Skarlso/coolrpgapp</a>

#### Character

&nbsp;

I need to be able to create a character with a name. Meaning, I need to figure out how meteor handles input. I already know that it uses templates and <a href="https://github.com/meteor/meteor/blob/devel/packages/spacebars/README.md" target="_blank">Spacebars Compiler</a>. So what I want at this point is to enter a username and then click a button which will direct me to the story page. Simple, right.?

For data handling we will use Meteor's <a href="https://www.meteor.com/try/3" target="_blank">Collections</a>.

Using a form to submit the username looks like this:

~~~javascript

Usernames = new Mongo.Collection("usernames");

if (Meteor.isClient) {
  Template.body.events({
    "submit .new-user": function (event) {
      // This function is called when the new task form is submitted

      var username = event.target.username.value;

      Usernames.insert({
        username: username,
        createdAt: new Date() // current time
      });

      // Clear form
      event.target.username.value = "";

      // Prevent default form submit
      return false;
    }
  });
}

if (Meteor.isServer) {
  Meteor.startup(function () {
    // code to run on server at startup
  });
}
~~~

Of course there is no way to know if that actually succeeded so far unless I get a look at the DB. Navigate to the folder of your app and type in:

~~~
meteor mongo
~~~

This will open a console to your database where you can query it like you would normally do with a mongodb console. Hence for me it's:

~~~bash

db.usernames.find() # which return this ->
meteor:PRIMARY>; db.usernames.find()
{ "username" : "olaf", "createdAt" : ISODate("2015-02-01T16:58:24.100Z"), "_id" : "MS67d95ShFkc3yHiX" }
{ "username" : "skarlso", "createdAt" : ISODate("2015-02-01T16:59:18.792Z"), "_id" : "ig8DJngmGKLca2dqS" }
~~~

As you can see, I already have two characters in the system. This is so far very easy but it does not redirect me to a new page displaying the beginning of my journey. Let's try a redirect.

# Complications

Turns out it's not that easy to get a redirect going. If I would be a beginner at this, I would give up right now and move on. The guide, or the tutorial does not contain any HINTS at least that I have to use a different method if I want a multi-layered multi-paged app. Of course Meteor provides a built in, easy to use, easy to add, answer-to-everything-you-ever-would-want-to-do, Login feature. But guys, it's not useful. I would go as far as say it's completely useless. Do you actually know someone who uses it? I would never use a built in something which is completely hidden from me and have no idea what it does. The ability to control what's happening is THE most important thing in every developers life.

So after I did a bit of digging and StackOverflowing ( which replaces the tutorial AND the user guide (and is a trademarked expression)), I found out that you can add <a href="https://atmospherejs.com/cmather/iron-router" target="_blank">Iron-Router</a> which was built specifically for this purpose.

~~~
meteor add iron:router
~~~

So all of a sudden my Page is completely screwed up with Iron Router information. Again, there is no information on this on Meteors page or in the guide nor in the COMPLETE guide so, I'm left Googling.

A very helpful StackOverflow ( again, and I'm wondering why people don't bother with the guide in the first place just go to stackoverflow straight ) answer explains to me the following:

"_You have to define a subscription handle (an object returned by Meteor.subscribe) in order to use it's reactive ready method : we'll reference it in the myDataIsReady helper to track data availability, and the helper will automatically rerun when the state of ready changes._"

Okay, so subscriptions are mentioned in the SECURITY section of the guide regarding detecting specific users and private data and so on and so forth. All right so that's used by iron routing as well which means I have to build that in, and not to mention first of all understanding how Iron Router works.

I'm going to stop here now. After spending a couple of hours I can determine that this stuff is not intuitive and "easy". I don't know enough about JavaScript and redirecting and Iron Router to be able to use Meteor out of the box. Which means I have to educate myself a bit before returning to this stuff.

Stay tuned for more.

And as always,
Thanks for reading!
