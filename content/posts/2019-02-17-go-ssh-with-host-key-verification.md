+++
author = "hannibal"
categories = ["go", "golang", "ssh", "security"]
date = "2019-02-17T21:01:00+01:00"
title = "Go SSH with Host Key Verification"
url = "/2019/02/17/go-ssh-with-host-key-verification"
description = "Go SSH with Host Key Verification"
featured = "ssh.png"
featuredalt = "ssh"
featuredpath = "date"
linktitle = "Go SSH with host key verification"
+++

Hi folks.

Following a long search and reading lots of debates and possibilities of doing SSH within Go, I was shocked to see that not a great many tools and people use SSH with host key verification. What I usually see is this:

~~~go
HostKeyCallback: ssh.InsecureIgnoreHostKey()
~~~

This is terrible. Now, I realise that doing HostKeyVerification can be tedious, but don't fear. It's actually easy
now that the Go team provided the knownhosts subpackage in their crypto SSH package located here:
[KnownHosts](https://godoc.org/golang.org/x/crypto/ssh/knownhosts).

This part in particular is interesting: [New](https://godoc.org/golang.org/x/crypto/ssh/knownhosts#New).

Using new with a known_hosts file a code can be written like this one to verify host keys:

~~~go
package main

import (
	"bytes"
	"fmt"
	"io/ioutil"
	"log"

	"golang.org/x/crypto/ssh"
	kh "golang.org/x/crypto/ssh/knownhosts"
)

func main() {
	user := "user"
	address := "192.168.0.17"
	command := "uptime"
	port := "9999"

	key, err := ioutil.ReadFile("/Users/user/.ssh/id_rsa")
	if err != nil {
		log.Fatalf("unable to read private key: %v", err)
	}

	// Create the Signer for this private key.
	signer, err := ssh.ParsePrivateKey(key)
	if err != nil {
		log.Fatalf("unable to parse private key: %v", err)
	}

	hostKeyCallback, err := kh.New("/Users/user/.ssh/known_hosts")
	if err != nil {
		log.Fatal("could not create hostkeycallback function: ", err)
	}

	config := &ssh.ClientConfig{
		User: user,
		Auth: []ssh.AuthMethod{
			// Add in password check here for moar security.
			ssh.PublicKeys(signer),
		},
		HostKeyCallback: hostKeyCallback,
	}
	// Connect to the remote server and perform the SSH handshake.
	client, err := ssh.Dial("tcp", address+":"+port, config)
	if err != nil {
		log.Fatalf("unable to connect: %v", err)
	}
	defer client.Close()
	ss, err := client.NewSession()
	if err != nil {
		log.Fatal("unable to create SSH session: ", err)
	}
	defer ss.Close()
	// Creating the buffer which will hold the remotly executed command's output.
	var stdoutBuf bytes.Buffer
	ss.Stdout = &stdoutBuf
	ss.Run(command)
	// Let's print out the result of command.
	fmt.Println(stdoutBuf.String())
}
~~~

Here is the whole thing as a [Gist](https://gist.github.com/Skarlso/34321a230cf0245018288686c9e70b2d).

Please try and avoid using Insecure host keys. It is easier, but can harm so much. Software like these:
[Man in The Middle Proxy](https://mitmproxy.org/) thrive in an environment that doesn't do it, or doesn't in other ways
mitigate this problem.

Be wise and be safe.
Thanks for reading,
Gergely.