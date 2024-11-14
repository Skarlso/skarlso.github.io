+++
author = "hannibal"
categories = ["programming", "Python"]
date = "2015-04-12"
tags = ["django"]
title = "Django – RPG – Part 2"
url = "/2015/04/12/django-rpg-part-2/"

+++

Hello.

Continuing where we left off with the Django RPG project. Next up is implementing a rudimentary registration and adding the ability to create a character. Maybe even, design the database through django's modelling.

<!--more-->

Since we are using Django's very own authentication model, I think we are covered in terms of users. Let's add two things for now. An Index page, where there is a link to login and a link to registration.

Adding the index first. Later I would like to switch to a base template model, but for now, I created a simple index.html page. That only contains the two links to the two views. The views are a simple function call in the views.py too which the URLConfig will later point to.

For now, the index function looks like this:

~~~python

def index(request):
  '''
  myrpg/rpg/views.py
  '''
	title = "My RPG"
	return render_to_response('index.html', {'title':title})
~~~

Note, that the title here is utterly unimportant but because I want to switch to a base.html template I'll leave it here for later usage.

That concludes the index. Now, let's create the registration. That is a little more complex, but still rather easy. We are just checking of the user already exists or not, if so, display and error, if not, create the user.

~~~python
# myrpg/rpg/views.py
def registration(request):
	state = "Please register."
	username = password = email = ''
	if request.POST:
		username = request.POST.get('username')
		email = request.POST.get('email')
		password = request.POST.get('password')
		if User.objects.filter(username = username).exists():
			# raise forms.ValidationError("Username %s is already in use." % username)
			state = "Username %s is already in use. Please try another." % username
		else:
			try:
				user = User.objects.create_user(username = username, email = email, password = password)
				state = "Thank you for registering with us %s!" % user.username
			except:
				state = "Unexpected error occured: %s" % sys.exc_info()[]

	return render_to_response('registration.html', {'state': state}, context_instance = RequestContext(request))
~~~

Here, I'm checking to see of the username already exists with the filter. This is by using Django's model which models the database like hibernate. It's a simple query. And I'm doing this, because this is faster than raising an exception. Later on, I'll be switching to a validation framework and django's own auth view. Because, why not.

The URL conf looks like this:

~~~python
#myrpg/rpg/urls.py
from django.conf.urls import url

from . import views

urlpatterns = [
    url(r'^$', views.index, name='index'),
    url(r'^login/$', views.login_user, name='login'),
    url(r'^registration/$', views.registration, name='registration'),
]
~~~

And this now, resides in a file under the RPG app and not the main one. The main one includes this one, like this:

~~~python
#myrpg/urls.py
from django.conf.urls import include, url
from django.contrib import admin

urlpatterns = [
    # Examples:
    # url(r'^$', 'myrpg.views.home', name='home'),
    # url(r'^blog/', include('blog.urls')),

    url(r'^admin/', include(admin.site.urls)),
    url(r'^rpg/', include('rpg.urls')),
]
~~~

That's it for now. As always, you can check out the code under github.

Tune in next time, when I'll attempt to create a view to create a Character for a logged in user and link it to the user. I'll do this with django's model framework.

Thanks for reading,

Gergely.