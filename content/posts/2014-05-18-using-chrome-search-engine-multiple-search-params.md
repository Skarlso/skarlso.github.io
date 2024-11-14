+++
author = "hannibal"
categories = ["knowledge", "programming", "testing"]
date = "2014-05-18"
title = "Using Chrome Search Engine – Multiple Search Params"
url = "/2014/05/18/using-chrome-search-engine-multiple-search-params/"

+++

Hello Everybody.

Today I would like to write a few words about Chrome's Search Engines.

You're probably already using it for a couple of things, like Google, or Amazon searches or YouTube or anything like that. But are you using it to access environments and testing tools faster, with queries?

For example, here is a quick Jira Search made easy:

Keyword: jira

URL: `https://atlas.projectname.com/jira/browse/PROJECT-%s`

So just type: jira|space|9999

Will immediately bring you to your ticket.

"Bah, why would I want that?" - you ask.

Well, it's easy, and quick access, but wait. There is more. How about you want to access a test environment that changes only a number?

Keyword: testenv

URL: `https://qa%s.projectname.com/testenv`

Just type: testenv|space|14

"Humbug!" - you might say. "What if I have a different URL for an admin site and my main web site AND the number, hmmm? Also I have that stuff bookmarked anyways." - you might add in.

Well, don't fret. By default, Chrome, does not provide this. I know FF does, but I don't like FF. That's that. So I have to make due with what I have. And indeed there is a solution for using multiple search parameters. It's is a JavaScript you can add in into the URL part and Chrome will interpret that. You can find that JavaScript in a few posts but you will find that THAT script is actually Wrong. Here is the **fixed** Script, courtesy of yours truly:

~~~javascript
var s='%s';
url='https://%s.test%s.projectname.com/';
query='';
urlchunks=url.split('%s');
schunks=s.split(';');
for(i=; i&lt;urlchunks.length; i++) {
	query+=urlchunks[i];
	if (typeof schunks[i] != 'undefined') {
		query+=schunks[i];
	}
}
location.replace(query);
~~~

So no you will have an entry like this:

Keyword: testenv

URL: paste in the upper script

And try. testenv|space|admin;14 => which should result in: https://admin.test14.projectname.com/

The location.replace at the end will bring you to the web page. It's interesting to note the s will be replaced by admin;14 which is a nice magic by JavaScript.

**NOTE**: This only works on a page like google.co.uk. For chrome pages, like the new tab, omnibox has this ability disabled unfortunately.

"Well then it's completely useless, isn't it?" - you might say. Well, it's usage is limited in power, that's true. But it's still useful as I'm sure you have a couple of pages open anyways which you don't mind using up.? And you have to remember less keywords only a few powerful ones.

Credit for telling about Chrome Search Engines power in the first place goes to. \*drumrolls\* => <a href="http://www.testfeed.co.uk/" target="_blank">http://www.testfeed.co.uk/</a>

Anyhow.

As always, thanks for reading.

Gergely.
