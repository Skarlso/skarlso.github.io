+++
author = "hannibal"
categories = ["Go", "Meetup"]
date = "2018-02-06T23:01:00+01:00"
title = "Go Budapest Meetup"
url = "/2018/02/06/go-budapest-meetup"

+++

# Intro

So I was at [Go Budapest Meetup](https://www.meetup.com/go-budapest) yesterday, where the brilliant [Johan Brandhorst](https://jbrandhorst.com/)
gave a talk about his project based on [gRPC](https://grpc.io/) using [gRPC-web](https://github.com/improbable-eng/grpc-web) +
[GopherJS](https://github.com/gopherjs/gopherjs) + [protobuf](https://github.com/google/protobuf). He also has some Go
contributions and check out his project here: [Protobuf](https://github.com/johanbrandhorst/protobuf). It's GopherJS Bindings for
ProtobufJS and gRPC-Web.

It was interesting to see where these projects could lead and I see the potential in them. I liked the usage of Protobuf and gRPC,
I don't have THAT much experience with them. However after yesterday, I'm eager to find an excuse to do something with these libraries.
I used gRPC indirectly, well, the result of it, when dealing with Google Cloud Platform's API. Which is largely generated code through
gRPC and protobuf.

He also presented a bi-directional stream communication between the gRPC-web client and the server which was an interesting feat
to produce. It did involve the use of [errgroup](https://godoc.org/golang.org/x/sync/errgroup). Which is nice.

I didn't look THAT much into WebAssembly however, again, after yesterday, I will. He gave a shout out to WebAssembly developers
that he is ready to tackle the Go bindings for WASM!

It was a good change of pace to look at some Go code being written, I'll be sure to visit the meetup again, in about three months
when the next one will come.

Maybe, I'll even give a talk if they are looking for speakers. ;)

A huge thank you to [Emarsys Budapest](https://www.emarsys.com/en/about-us/) for organizing the event and bringing Johan to us
for his talk.

Thanks,
Gergely
