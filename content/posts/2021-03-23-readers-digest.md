+++
author = "hannibal"
categories = ["books"]
date = "2021-03-23T01:01:00+01:00"
title = "Reader's Digest 2021-03"
url = "/2021/03/23/readers-digest"
comments = true
+++

# Reader's Digest

I thought it would a cool idea if I kept a summary of the things I've read or listened to on a monthly
basis. Here is March of 2021 so far. Enjoy.

## The Aurora Database paper

The paper about Aurora database from AWS can be found here: [Paper](http://nil.csail.mit.edu/6.824/2020/papers/aurora.pdf).
It details the design decision taken to support a highly available, fault tolerant, fast replicating
database. They take the following approach... They modified mysql database such as that they only send
around the redo log and the redo log is enough to recover / replicate in order to achieve write and
read consistency. They separate the data into Protected Groups and speed up terabytes of recovery by
doing 10 Gigabyte segments in parallel. The database IS the logs. By only replicating the log instead
of the data and the data page, they save millions in networks costs. The main gain however, is that
the storage was modified to understand the application. Instead of using General store they use a storage
which understand the data. In this case, decoupling storage from the database, as so many do, was actually
a drawback.

## GCatch paper

This paper is a static concurrency bug analyser for Go found here [Paper](https://songlh.github.io/paper/gcatch.pdf).

It's ingenious! It's a static analyser which finds mostly blocking bugs using channels in Go. In Go, it's
really easy to write concurrent software using something called [Channels](https://tour.golang.org/concurrency/2).
They are basically coroutines multiplexed onto kernel threads and thus you can have a million of them
running around doing stuff. Go effectively made IO operators CPU bound with them. Coroutines aren't new,
however, it's really easy to mess up code with channels in subtle ways. Analyzers exist, however, GCatch
argues that they can't find the most subtle of bugs, only some surface bugs really.

This paper proposes a tool which does inter-procedural, path-sensitive analysis and uses Z3 to find paths
which can lead to deadlocks in code that uses locking primitives and channels. It also contains five other
prominent tools. It converts mutexes into channels internally with buffer size zero and sends on it on
Lock and reads from it on Unlock, then performs a bunch of path combinations and goes through those
suspicious paths and performs its analysis.

They found a hundred and something bugs in Docker and Kubernetes. Things like, sending on a channel in
`select` when in fact, a timeout already returned, thus that Go routine is not indefinitely stuck. Since
it can't send its output on the channel, the program didn't quit so it's not GC-d. A simple fix is to
make the channel of size 1 so even if there is a chance that the scope quit it can still send and quit.
Like Exec.

It's an interesting read and the tool is awesome, however.... It was written with Go 14 and it's proving to
be difficult to port to current version using modules. I would hate to see this tool getting left behind
because it can't be turned into a linter.

## Rhythm of War - Brandon Sanderson

[Amazon](https://www.amazon.com/Rhythm-War-Stormlight-Archive-Book-ebook/dp/B0826NKZHR).

An epic continuation of this saga with over a 1000 pages long and 54 hours of listening time on Audible.
This story has been ongoing for a while now. Brandon Sanderson came out with the first book back in 2010.
This is the continuation of the Stormlight Archive series. These are massive master pieces. I first came
along Brandon Sanderson when I read the Mistborn series. That was another epic novel. I love reading
Sanderson because he comes up with some unique ways of magic or magic like abilities which have some
divine sense in the end, or have some interesting explanation. And their abilities are almost always
used in interesting ways.

For example, a simple ability to pull or push metal. Turns out that results in things like, shooting
coins, or literally flying as the person tosses a coin to the ground and pushes on it, pushing themselves
upwards in the end.

I could write many many pages about each and every fantastic novel, but I'm going to stick to this one
expecting that people know about the series.

I've listened to this one as I'm insane busy, I couldn't have read a 1000 something pages book.
The fantastic work of Micheal Kramer and Kate Reading is always a treat to listen too. They are both
excellent readers always making the characters live through their words.

SPOILERS:

This time we mostly follow Eshonai's and Venli's but we finally also get what we wanted all these years.
Finally, Kaladin and Shallan face their inner demons. And even though they aren't fully okay, Kaladin
speaks his fourth oat and Shallan remembers her past. As much as I love this story, I don't believe I
would have been able to listen or read another 1000 pages without these two resolving their problems.
You root for them so hard, it's exhausting.

I won't spoil everything but the twist at the end left me dumbstruck! It was such an amazing finish.

The story follows the fused as they invade Urithiru. There is a side story for Navani and Jasna
doing their own thing and we do root for Navani and her fantastic discoveries regarding light and powers,
but Jasna is a side character in this story. Another main character is Witt. We finally get to know
who he is and where he comes from. We also understand now that the Fused are actually from another planet
in the same system and Odium just wants to get off this system and fight a holy war with some ancient Gods
somewhere. A lot of things which made no sense are revealed finally. I recommend it if you have the time
to listen or read it.

## How to take smart notes

[Amazon](https://www.amazon.com/How-Take-Smart-Notes-Nonfiction-ebook/dp/B06WVYW33Y).

This one's review will be condensed because it would be rather lengthy otherwise. It's basically talking about
how to use the [Zettelkasten](https://zettelkasten.de/) system. But it does so much more then that. It challenges
the way you think, the way you learn the way things are taught in school and the way you process and store
information. Condensed I would say these are the main points:

- Connect new information to existing information. Information without connection isn't worth much and will be
remembered poorly or not at all.
- Always read with a pen in your hand and take notes about what you are reading.
- Always use your own words and never just copy blindly; by doing this, you will better understand what you just
read. The same goes to things like, writing a blog in which you explain something you think you know. It reveals
the black holes in your knowledge which you didn't even know exist.
- Don't try to group based on topics. That will result in forced connections and will leave you confined within
that topic. Topics should emerge from your notes and then gathered into indexes which contain links to related
notes and information.
- Tags are useful but don't over do them. If you have a 1000 tags your information will be lost and hard to find
because things that are unrelated will show up in the searches. So go easy on the tags
- Note taking is a chore. It's not something that you just do and it just works. Good note taking requires effort.
You take notes while you read then transcribe them into Zettelkasten and throw away the rest. Those are transient
notes. Zettelkasten notes focus on the gist of things. On the meat!

## Conclusion

And that's all for this months. Rhythm of War, the papers and the note taking book pretty much took all my
time away, so not much else got done since January. But I still think this is a nice finish. Especially
considering Rhythm of War was such a huge epic.

And as always,
Thanks for reading,
Gergely.
