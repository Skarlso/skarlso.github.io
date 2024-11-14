+++
author = "hannibal"
categories = ["knowledge", "problem solving", "programming", "Python"]
date = "2014-02-11"
title = "How to check content header on unknown number of items – Python"
url = "/2014/02/11/how-to-check-content-header-on-unknown-number-of-items-python/"

+++

Hello guys.

I'd like to share a little something with you. It's what I cooked up in Python to check an unknown number of content items in a web application.

Basically the script runs from a script folder under Grails. It goes through all the configured folders where there is static content like images, javascript, css and so on and so forth.

And then with curl it calls these items up in using their respective paths'. This works best on localhost if you have your local environment configured to access these elements because in some places direct access is restricted.

This script only check static content. Dynamically generated content would have to be hard coded to check.

It only generated a file currently with ERROR on a not match an success on match and not found if it encounters an item which it doesn't know about.

So without further ado. The Script:

~~~python
#!/usr/bin/python

import pycurl, sys, os, urllib

class Storage:
    def __init__(self):
        self.contents = ''

    def store(self, buf):
	if 'Content-Type' in buf:
            self.contents = buf

    def __str__(self):
        return self.contents

#print retrieved_headers

filesInDir = []
headers = []
headerRestrictions = {'.css': 'Content-Type: text/css', '.jpg': 'Content-Type: image/jpeg', '.ico': 'image/vnd.microsoft.icon', '.html': 'Content-Type: text/html', '.js': 'Content-Type: application/javascript', '.gif': 'Content-Type: image/gif', '.png': 'Content-Type: image/png', '.swf': 'Content-Type: application/x-shockwave-flash', '.json': 'Content-Type: application/json', '.htc': 'Content-Type: text/x-component', '.xml': 'Content-Type: application/xml'}

for dirname, dirnames, filenames in os.walk('../web-app'):
    # editing the 'dirnames' list will stop os.walk() from recursing into there.
    if '.git' in dirnames:
        # don't go into any .git directories.
        dirnames.remove('.git')

    if 'WEB-INF' in dirnames:
        # don't go into any WEB-INF directories.
        dirnames.remove('WEB-INF')

    if 'test' in dirnames:
        # don't go into any test directories.
        dirnames.remove('test')

    if 'META-INF' in dirnames:
        # don't go into any META-INF directories.
        dirnames.remove('META-INF')

    for filename in filenames:
	trimmedDir = dirname.replace("web-app/", "")
	trimmedDir = trimmedDir.replace("../", "")
	filesInDir.append(os.path.join(trimmedDir, filename))
    #    print os.path.join(dirname, filename)

f = open("headersandfiles.txt", "w")

for fileName in filesInDir:
    retrieved_body = Storage()
    retrieved_headers = Storage()
    c = pycurl.Curl()
    fileName = fileName.replace(" ", "%20")
    url = 'http://localhost:8080/%s' % fileName
    c.setopt(c.URL, url)
    c.setopt(c.WRITEFUNCTION, retrieved_body.store)
    c.setopt(c.HEADERFUNCTION, retrieved_headers.store)
    c.perform()
    c.close()
    fileLine = ""
    fileNameBase, fileExtension = os.path.splitext(fileName)

    if headerRestrictions.has_key(fileExtension):
#	print "Header:%s, Content:%s" % (headerRestrictions[fileExtension], retrieved_headers.__str__())
        fileLine = "CORRECT: Content: %s; Header: %s" % (fileName, retrieved_headers) if headerRestrictions[fileExtension] == retrieved_headers.__str__().strip() else "ERROR: Content: %s; Header: %s; URL: %s" % (fileName, retrieved_headers, "http://localhost:8080/%s\n" % fileName)
    else:
	fileLine = "NOT FOUND: Content: %s; Header: %s" % (fileName, retrieved_headers)

    f.write(fileLine)
    headers.append(retrieved_headers.__str__())

f.close()
~~~

Hope you like it. Feel free to improve however you want.

Thanks for reading,

Gergely.