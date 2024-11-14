+++
author = "hannibal"
categories = ["go","capa"]
date = "2022-06-26T01:01:00+01:00"
title = "Hacking on CAPA - The journey of implementing a nontrivial feature in a barely known codebase"
url = "/2022/06/26/hacking-on-capa"
comments = true
+++

# Hacking on CAPA - The journey of implementing a nontrivial feature in a barely known codebase

Hello Dear readers.

Today, I would like to write about a project I've been working on these past months or so. This is a longer story
and hopefully an interesting one to read.

I'm going to write about the journey I took while trying to implement IPv6 based Kubernetes cluster for [CAPA](https://github.com/kubernetes-sigs/cluster-api-provider-aws)
and EKS.

The interesting points of this journey are twofold. First, understanding IPv6 in AWS land and how it's configured
and how it works. What are it's limitations and requirements? Topology, routing, security groups, launch configuration,
IAM roles, node policies... etc.

The second part is the codebase of CAPA, which was largely unknown to me. What I'm going to write about here is
how I navigated these hurdles.

## TL;DR

- I take lots of notes and re-work these notes into a semi-zettelkasten format
- I read RFCs and documentations a couple times
- I start exploring the codebase by reading package information and going through a known code path like, creating a
cluster and reading tests
- I had some prior knowledge due to working on a similar project at Weaveworks ( implementing ipv6 for eksctl )
- I had ample of help from one of the project maintainers who happens to work at Weaveworks
- I had some documentation already available on IPv6 and AWS and could compare to existing, working IPv6 clusters
using eksctl

## What is CAPA

Before I begin, let's talk about what [CAPA](https://github.com/kubernetes-sigs/cluster-api-provider-aws) is. It's a project which implements
[cluster API](https://cluster-api.sigs.k8s.io/) for Kubernetes using AWS. CAPI ( Cluster API ) is simply a set of API
projects and tooling for Kubernetes to basically use Kubernetes programmatically. It provides a way to create
clusters using a management cluster which runs the API code using a couple CRDs and controllers.

Once you create your management cluster, you can apply a YAML template which will use the AWS API to create a set of
objects for an EKS cluster. These are for example, IAM roles, a VPC, subnets, routing tables, security groups, IGWs, NAT gateways,
instances and many many more things. Once that's done it will keep reconciling the cluster. Detect diffs and manage its
state until you remove the resources for it ( the CRDs installed onto the management cluster ).

## IPv6

The first thing to tackle was understanding and knowing IPv6. Though, funny enough, as we'll see later, it wasn't a hard
requirement. Never the less, it did me good to study up on it.

### How do I approach the unknown

The reason why I wanted to understand IPv6 was because of how to calculate subnets. Subnetting is an interesting topic
in the land of IPs and one that isn't easy to grasp if you aren't into these kinds of things in the first place.

I did learn how to calculate CIDRs for IPv4 so I thought it should be kind of in the same ballpark. It actually took me
a while to find a resource which neatly explained the math behind the calculations and not just throw a website into my
face into which you input a prefix range and done, you have your subnets. I rarely watch videos but [this](https://www.youtube.com/watch?v=KSiZ751-Zs8) one has
a neat explanation with detailed calculations and samples.

What I do in these cases, is as follows. I open up a new page in my trusty notebook and start listing what things I
can read or watch to understand the subject at hand better. Then I slowly start working down my list, taking notes from
the material or video in a fashion similar as what is depicted in [Effective Note taking](https://www.amazon.de/-/en/Fiona-McPherson/dp/1927166527/) book
written by Dr. Fiona McPherson. I create concept graphs and various visuals and just write down my thoughts.

I'll try to connect to future notes or write ideas next to it, things like "Oh this will be useful when I do x,y,z".

A page from one my notes:

![page1](/img/2022/06/27/notes-01.jpeg)

This is in pre-review format. Once I'm done with a material, I'll start translating the notes and get the gist of it into a
semi-zettelkasten type of note system using Obsidian. Or if it makes more sense, I'll use a single obsidian note to
collect all the things together. For this implementation I'm doing on CAPA I have a single file which points to various
other files with various other information about the code and about IPv6 itself.

Once I translated my written notes, I'll move on to the next topic. I have lots of tags and always create back-links. I use
something akin to [Zettelkasten](https://zettelkasten.de/posts/overview/) but with a slight modification that sometimes
my atomic notes contain a bit more information than a single idea.

#### Why use a notebook in the digital age?

I like writing by hand. It gives me a motoric impression of the subject and drawing pictures is a lot easier. Also, I
found if I take notes on the computer in Obsidian directly, I tend to ramble more because typing is so easy. Which leaves
my notes full of unnecessary information that I have to go through when refining. Lots more to read. Further, typing is
so easy indeed, that it's too easy. There are no motoric impressions, no muscle memory built no aid in thinking. And lastly,
the linear nature of a notebook forces me to focus on a single thing at once unless I want my notes completely jumbled up.
If I'm on the computer I found I tend to jump around different subjects all the time and not finish what I started.

This blog post's idea has its origins in a notebook.

### Understanding IPv6

The first step I took was to understand IPv6 itself. I already knew a couple of things about it, but I didn't intimately knew
its structure and composition. The best way to know about these things is go to the source. I read a couple of RFCs
about it. Notably [RFC 8200 - The protocol itself](https://datatracker.ietf.org/doc/html/rfc8200) and [RFC 4291 - The Address itself](https://datatracker.ietf.org/doc/html/rfc4291).

Reading these and taking notes and understanding the underlying technology made me not be afraid of it anymore. But also
it had the added benefit of me understanding why some of the decisions were made at AWS. I have a saying.

If you are afraid of it, you have to do it. If some technology or software concept or hard problem intimidates you THAT
is a clear sign that you have to understand it and make it your own.

Next, I watched the video about subnetting and read some posts, of which I saved the links and took some notes that I
connected back to my existing notes, to get a large picture and the immediate usage and interesting parts of IPv6.

It was time to put it all in perspective.

### Understanding IPv6 in relation to AWS

As always with AWS finding documentation which describes a feature in its completeness is near impossible. I found
various blog posts, some hints here and there and when you open the EKS console page, there is an announcement at the
top that they started supporting IPv6. That page contains some information about restrictions and requirements. But
nothing profound enough to know how to call some APIs or what to set up or how routing works exactly. Or what to pass to
the VPC. Or how to slice subnets not using eksctl and cloudformation.

Lucky for me, I had some prior knowledge in the area. I work at https://weave.works/. And at `eksctl` a couple of
co-workers of mine already did some research and created some documentations that I was able to follow. But ultimately,
that was using a different approach because it's using CloudFormation. But the notes on security groups and VPCs was
invaluable.

The other guide which helped me was the [VPC usage guide](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ug.pdf) and
the IPv6 section. And the EKS usage guide and how to migrate to IPv6 clusters. There were numerous restrictions which
helped me understand what I needed to create. I read these over 5-6 times.

From all of these, notes taken into my trusty notebook in which I gathered together all the relevant information that I
could connect to things I needed, I slowly put together an approach for CAPA.

I also had the benefit that I could compare my CAPA created network topology to that of an `eksctl` created cluster. I
could verify that the cluster created by CAPA had similar routings, VPCs and subnet configuration.

For example, a "private" ( there is no such thing as private in the land of IPv6 ( there are no private/local ips ) )
VPC configuration requires an Egress Only Internet Gateway for IPv6 clusters so network translation can still work.

The most difficult part though to discover was subnetting.

#### The journey of subnetting IPv6

My first approach was the mathematical one. I implemented code which would slice an IPv6 address into the needed number
of subnets based on AZs that looked like this:

```go
func SplitIntoSubnetsIPv6(cidrBlock string, numSubnets int) ([]*net.IPNet, error) {
	_, parent, err := net.ParseCIDR(cidrBlock)
	if err != nil {
		return nil, errors.Wrap(err, "failed to parse CIDR")
	}

	subnetBits := math.Ceil(math.Log2(float64(numSubnets)))
	networkLen, addrLen := parent.Mask.Size()
	modifiedNetworkLen := networkLen + int(subnetBits)

	if modifiedNetworkLen > addrLen {
		return nil, errors.Errorf("cidr range %s cannot accommodate %d subnets", cidrBlock, numSubnets)
	}
	addr, err := netaddr.ParseIPv6Net(cidrBlock)
	if err != nil {
		return nil, errors.Wrapf(err, "failed to parse network address %s", cidrBlock)
	}

	var subnets []*net.IPNet
	for i := 0; i < numSubnets; i++ {
		subnetIP := addr.NthSubnet(uint(modifiedNetworkLen), uint64(i))
		if subnetIP == nil {
			return nil, errors.Errorf("exceeded the subnet space with len %d", modifiedNetworkLen)
		}
		parsedSubnetIP, _, err := net.ParseCIDR(subnetIP.Long())
		if err != nil {
			return nil, err
		}
		v := &net.IPNet{
			IP:   parsedSubnetIP,
			Mask: net.CIDRMask(modifiedNetworkLen, 128),
		}
		subnets = append(subnets, v)
	}
	return subnets, nil
}
```

It's using a library to calculate the actual new IP based on `NthSubnet`. I did some bit shifting myself, but that turned
out to be not so error resilient and didn't handle some edge cases that the library already did.

However, this research all turned out to be mute once I re-read my notes and some of the docs and I internalized the point
that all subnets have a fixed prefix length of `64`. And further reading about subnet IDs in IPv6 and creating IPv6
clusters using the console it became apparent that subnets are super easy, barely an inconvenience.

See, it basically works like this. You get an IPv6 pool and an IPv6 address with a fixed prefix length of 56. You need to
switch that to 64 and just increase the subnet ID for any further subnets. This looks like this:

```go
func SplitIntoSubnetsIPv6(cidrBlock string, numSubnets int) ([]*net.IPNet, error) {
	_, ipv6CidrBlock, err := net.ParseCIDR(cidrBlock)
	if err != nil {
		return nil, fmt.Errorf("failed to parse cidr block %s with error: %w", cidrBlock, err)
	}
	// update the prefix to 64.
	ipv6CidrBlock.Mask = net.CIDRMask(64, 128)
	var (
		subnets []*net.IPNet
	)
	for i := 0; i < numSubnets; i++ {
		ipv6CidrBlock.IP[subnetIDLocation]++
		newIP := net.ParseIP(ipv6CidrBlock.IP.String())
		v := &net.IPNet{
			IP:   newIP,
			Mask: net.CIDRMask(64, 128),
		}
		subnets = append(subnets, v)
	}
	return subnets, nil
}
```

Boom. And that's all there is to it! That's mostly what the CloudFormation is doing internally too.

### Why do you still need subnets?

Once I gathered all resources that needed updates, it was time to write them down. Subnets and networking are obviously
the two big hitters. With IPv4 we needed the separation and a thing called `SecondaryCidrBlock` because you can easily
run out of ips. With IPv6 you don't need to worry about that. However, we do still need to subnet, because we want
resiliency through availability zones. Each AZ gets a subnet and an assigned IPv6 address.

### Private networking

Another thing of the past is private networking. There no private addresses or local addresses for IPv6. All IPv6
addresses are public. Which means, we need another way of defining a private network. That is achieved through security
groups and routing. The routes will have a new resource called `EgressOnlyInternetGateway`. Which is basically what it
says. An IWG which works with IPv6 addresses, allows for outgoing communication through IPv6 but blocks all incoming
data through IPv6.

## Implementation Time

Alright. With these things out of the way, it was time to get acquainted with the source code.

### Approaching an unknown codebase

I had the a couple of things on my side.

- I had ample of experience with working with controllers in Kubernetes
- I had personally wrote couple of projects which worked with AWS and the GO SDK
- I had experience in working with AWS in general
- I had one of the maintainers as a colleague working here at Weaveworks

How did these things help me? Let's take a look.

The fact that I already worked with a lot of controllers makes me aware of a couple of things when working with such a
codebase. What are these?

- The API is versioned in a certain way like `alphav1` and `alphav2` and `betav1` and so on
- Lots and LOTS of generated code
- Controller lifecycle
    - What does this mean? This has a couple of baggages when you think about it. The code has a reconciliation loop
      which takes care of creating, updating and deleting resources based on their state. But this also means that you
      have to always consider existing infrastructure and objects in Kubernetes. And when deleting you always have to be
      aware of finalizers.
- Unit testing and mocking calls and Kubernetes
- Certain package structure and code constructs

All these helped me massively to discover where things are and why. Richard ( the maintainer ) talked us through a general
flow and from there it was easier to go with the flow. And he was constantly available when I was stuck at a certain part
like conversions...

### Analysing the code

When faced with the unknown you must find what you might no in relation. I already knew it creates a cluster and that
cluster has resources and log outputs. So... follow the flow. I started checking out the code and followed a `create`
code path. How a cluster is reconciled.

I started taking notes and write down which package and path did what. I started following `reconcileCluster`.
This is the main flow. It contained everything. I glanced over the sub flows of reconciling providers and security groups
and the likes, but I mainly kept with the main flow.

This limitation allowed me to focus on the most important bits. Where code lives, why and how it is configured.
Once I knew all of these things, I could focus on various sub-flows. Since my main focus was altering the network and
instance bits, I focused on `ReconcileNetwork` flow and Machines.

Again, same thing. Go over the main flow, follow the code path, see how things are set up where they live and why, but
don't go too deep. Once the main flow is understood, follow the sub-flow, like `reconcileVPC`.

### Update, test, update, test...

Once I got a feel for things and noted down places where I probably will have to update something, I started updating
things incrementally. Small bits at a time. My first update, for example, was this trivial thing:

```go
	// EnableIPv6 requests an Amazon-provided IPv6 CIDR block with a /56 prefix length for
	// the VPC. You cannot specify the range of IP addresses, or the size of the
	// CIDR block.
	// +optional
	EnableIPv6 bool `json:"enableIPv6"`
```

Basically I started updating the config types. It was a small, trivial thin to add but it got the ball rolling. Which
is the most important part of tackling a hard problem. When the codebase is daunting and you have no idea where to even
begin, a tiny update can make all the differences. It gets you started. It gives you an entry point.

I started adding more new settings and types and once I had all the settings I could think of, I started writing code
which actually used them. First, gradually. Things like these came in to picture:

```go
		s.scope.VPC().EnableIPv6 = vpc.EnableIPv6
		s.scope.VPC().IPv6CidrBlock = vpc.IPv6CidrBlock
		s.scope.VPC().IPv6Pool = vpc.IPv6Pool
```

Just a trivial assignment. Then a tiny bit of logic...

```go
	if !s.scope.VPC().EnableIPv6 {
		return &infrav1.VPCSpec{
			ID:        *out.Vpc.VpcId,
			CidrBlock: *out.Vpc.CidrBlock,
			Tags:      converters.TagsToMap(out.Vpc.Tags),
		}, nil
	}
```

Deal with things that returned with normal flow. Here, the important part is that CIDR block is defined instead of
`IPv6CidrBlock`.

These small, incremental updates, eventually lead to this pr: [PR](https://github.com/kubernetes-sigs/cluster-api-provider-aws/pull/3513). As of this writing it needs some validations added
and an e2e test, but it's pretty much complete.

## Testing

Tests are super important in a new codebase ( in an old one too, but that's a different story ). Reading the tests helps
me understand somewhat how the code is structured and what it does and why and what calls are expected to happen. What I
do is, that I fiddle with the code and start to run the unit tests and see what things fail. That gives me more
insight into how the thing I just changed works. This is invaluable when I'm facing code which is difficult to decipher.

Write tests folks! I don't care if you use TDD or not, jut PLEASE write tests!

# Conclusion

If you made it this far, congratulations! You deserve a cookie. This has been my journey in understanding a fairly new
thing and implementing it into a largely unknown codebase. How I approached it and how I eventually ended up writing
around +2500 lines of code. You got some insight into how I take notes and how I figure out a larger problem in an
unknown environment. I hope it was a good read and worth your time.

Thank you for reading!

Gergely.