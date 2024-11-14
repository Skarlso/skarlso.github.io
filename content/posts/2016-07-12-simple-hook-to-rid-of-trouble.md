+++
author = "hannibal"
categories = ["Git"]
date = "2016-07-12"
title = "Simple hook to rid of trouble"
url = "/2016/07/12/simple-hook-to-rid-of-trouble"

+++

Hi folks.

This is but a simple git hook to run a test in order to ensure you can push. It also ignores the vendor folder if you happen to have on in your directory.

Edit the file under ```.git/hooks/pre-push.sample``` and add this at the end before the ```exit 0```.

~~~bash
go test $(go list ./... |grep -v vendor)
RESULT=$?
if [ $RESULT -ne 0 ]; then
    echo "Failed test run. Disallowing push."
    exit 1
fi
~~~

After this, rename the file to ```pre-push``` removing the .sample from it.

If you now, mess something up, you should see something like this before your push:

~~~bash
# github.com/Skarlso/goprogressquest
./create.go:40: undefined: sha1 in sha1.Sum
./create.go:41: undefined: fmt in fmt.Sprintf
./create.go:115: undefined: json in json.Unmarshal
./create.go:130: undefined: json in json.Unmarshal
FAIL	github.com/Skarlso/goprogressquest [build failed]
Failed test run. Disallowing push.
error: failed to push some refs to 'git@github.com:Skarlso/goprogressquest.git'
~~~

That is all.

Cheers,
Gergely.
