---
title: "Recurse Center lab notes: 2015-06-08"
date: 2015-06-08
permalink: /blog/2015/06/08/recurse-center-lab-notes
---

# Linux IPC stuff
I'm getting closer to having some data on latencies for pipes and eventfds. I
reworked the little programs to easily dump timings for multiple runs. Next
step is to figure out how to visualise it. Likely I'll use iPython Notebook and
Pandas.

On the way, I found out a thing—one of those obvious-in-retrospect things. Both
the pipes and eventfd programs I wrote have the same structure. They start a
child process that acts as a server that responds as soon as it receives a
message. The child just runs a loop like this:

~~~ C
for (;;) {
  char msg;
  read(rx_fd, &msg, 1);
  write(tx_fd, "1", 1);
}
~~~

I implemented this for pipes first, and this works great. A `read` on a pipe
whose write end is closed will return 0 to signal end of file. A `write` on a
pipe whose read end is closed results in a `SIGPIPE` signal. The result is that
once the parent closes its pipes—whether explicitly or on exit—the child will
die because the default action for `SIGPIPE` is to terminate the process.

With eventfds, there is no analogous signal. It would actually be a bit odd.
Pipes have two ends, and each gets its own file descriptor. When the last file
descriptor referring to one of the ends is closed, then the `EOF` / `SIGPIPE`
behaviour kicks in. To get something similar to happen with eventfds, the
special behaviour would have to kick in when only one file descriptor
references the eventfd object. But right after the call to `eventfd`, there is
just one file descriptor referencing it. I suppose it could be special cased to
trigger when the number of open file descriptors goes _down_ to one, but it's
also completely reasonable that it wouldn't.

All that is to say, the child doesn't terminate when the parent does, and I ended up with this:

~~~
$ pgrep eventfd | head
6123
6131
6138
6145
6153
6160
6531
6538
6545
6553
$ pgrep eventfd | wc -l
172
~~~

That's a lot of zombies! Don't worry, I killed them. With FIRE^W `pkill`. I
still have to fix the problem, but the solution is pretty easy: kill the child
before the parent exits.

# Other stuff
The internet went down for a bit at RC today. I used the time to add an index
page to this blog, [making some people happy][index-tweet]. It felt silly to
use lack of internet to make internet, but there you go.

During the internet outage, I also brainstormed with [Ahmed] a bit on grammar
ideas for his Arabic programming language. I was thinking it would be cool to
integrate some Arabic grammar into the language. For example, use the
possessive for property access, and allow the [attached pronouns][pronouns] to
be a default argument like Perl's `$_`. We decided they wouldn't be a great
idea because it could end up in a sort of uncanny valley where something it
looks like should work but it doesn't.

[index-tweet]: https://twitter.com/kamalmarhubi/status/607967764620431361
[ahmed]: https://twitter.com/SimplyAhmaz1ng
[pronouns]: https://en.wikipedia.org/wiki/Arabic_grammar#Enclitic_pronouns

I paired with [Jess] who is learning Haskell to do some lattice-based
cryptography stuff. But step 1 is tic-tac-toe. We did some refactoring, and I
pointed out a couple of useful things like using `@` in a pattern to name the
whole match. Somehow I'm pretty happy helping people with Haskell, even though
I'm not really that interested in it anymore myself. Maybe it's just because
it's something I know and so can help with.

[jess]: https://twitter.com/optimistsinc

Allison Parish is a resident for the next two weeks. She gave a talk in the
evening on [new interfaces for textual expression][nite]. Do take a look at
that link, her projects are really cool! She's aiming to put together a bunch
of prototypes while at RC, and I'm really excited to see what she gets up to.

[nite]: http://www.decontextualize.com/projects/nite/

# Random

On the way to eBay for Allison's talk, I stopped by Union Square to buy an
apple. I also got a ‘cocktail’ of apple juices for free—I stopped to look at a
stall that was packing up, and their demo bottles were mostly empty.  The
person poured them into one bottle and just gave it to me.

I then took my apple and juice to a table on Broadway and sat down. Two people
next to me were eating burgers and speaking Québecois French. I wasn't going to
insert myself until I heard ‘Rachel et Saint-Hubert’, which is an intersection
close to where I live. They were discussing the burger, and so I asked (in
French) if they were talking about L'Anecdote, a burger place at that corner. I
think this was a fun surprise for them, and we had a little chat about what
brought us to New York. Even more fun, I was wearing my ‘[Farine Five
Roses][five-roses]’ t-shirt, marking me—in one of their words—as _vrai
Montréalais_!

[five-roses]: https://en.wikipedia.org/wiki/Five_Roses_Flour
