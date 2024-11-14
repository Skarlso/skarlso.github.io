+++
author = "hannibal"
categories = ["programming", "Python"]
date = "2015-04-21"
tags = ["rpg"]
title = "Django – RPG – Part 3"
url = "/2015/04/21/django-rpg-part-3/"

+++

Hello folks.

A small update to this. I created the model now, which is the database design for this app. It's very simple, nothing fancy. Also, I'm writing the app with Python 3 from now on.

Here is the model now:

~~~python

from django.db import models
from django.contrib.auth.models import User

# Create your models here.


class Item(models.Model):
    name = models.CharField(max_length=100, default="Item")
    damage = models.IntegerField(default=)
    defense = models.IntegerField(default=)
    consumable = models.BooleanField(default=False)

    def __str__(self):
        return self.name


class Inventory(models.Model):
    items = models.ManyToManyField(Item)

    def __str__(self):
        return self.items


class Character(models.Model):
    # By default Django uses the primery key of the related object.
    # Hence, no need to specify User.id.
    user = models.OneToOneField(User, null=True)
    name = models.CharField(max_length=100)
    inventory = models.ForeignKey(Inventory)

    def __str__(self):
        return self.name
~~~

Worth noting a few things here. The \_\_str\_\_ is only with Python 3. In Python 2 it would be unicode. And the OneToOne and the foreign key are automatically using Primary keys defined in the references model. The \_\_str\_\_ is there to return some view when you are debugging in the console instead of [<Item: Item object>].

In order to apply this change you just have to run this commend (given you set up your app in the settings.py as an INSTALLED_APP):

~~~

python manage.py makemigrations polls
~~~

This creates the migration script. And this applies it:

~~~python

python manage.py migrate
~~~

I love the fact that django creates incremental migration scripts out of the box. So if there was any problem at all, you can always roll back. Which comes very handy in certain situations.

That's it.

Thanks for reading!

Gergely.