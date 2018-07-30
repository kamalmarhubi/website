<!doctype html>
<title
  >Writing raw HTML</title
><article
  ><h1
    >Writing raw HTML</h1
  ><div class="sidenote-container"
    ><p
      >I'm writing this post by hand in HTML like it's 1998.</p
    ><aside
      >1998 is when we got the internet in Oman, so I definitely wasn't writing HTML before then.</aside
  ></div
  
  ><section
    ><h2
      >Where this site started</h2
    ><p>This site has been <a href="TODO">jekyll</a>/<a href="TODO">octopress<a> based since before I ever even wrote anything for it.
<a href="TODO">Julia</a> helped me set it up <em>just in case</em>.
(In true Julia fashion, she wrote <a href="TODO">a blog post about it</a>.)
The idea was to make it so there was no friction if I ever wanted to publish something.
That worked great:
it stayed empty for six months.<p

    ><p>The site actually sat more or less empty for a little over six months before I wrote a post:
<figure role="img" aria-labelledby="git-log-caption">
<pre>
<samp>
$ <kbd>git rev-list HEAD -- _posts | tail -3 | xargs git show -s</kbd>
commit fb4abc07bfd42754ede1683dd7876c684bad0e5e
Author: Kamal Marhubi &lt;kamal@marhubi.com&gt;
Date:   Mon May 25 23:48:42 2015 -0400

    Add post: 'Recurse Centre OMG'

commit 5f93ac0045e797076247933e9dabad8d48d7709c
Author: Kamal Marhubi &lt;kamal@marhubi.com&gt;
Date:   Mon May 18 10:32:26 2015 -0400

    Delete default welcome post

commit b535826d51c7500930a2febeb7084f479e41b7d8
Author: Kamal Marhubi &lt;kamal@marhubi.com&gt;
Date:   Sat Oct 4 22:55:29 2014 -0400

    Initial commit
</samp>
</pre>
<figcaption class="aria-desc" id="git-log-caption">
Output showing <a href="TODO">first commit</a> in early October 2014, and <a href="TODO">first post</a> in late May 2015.
</figcaption>
</figure>

<p>I started writing when I went to the <a href="https://recurse.com/">Recurse Center</a>,
and having the site good to go saved me a bunch of yak shaving in that first week.

</section>



<section>
<h2>Enough with static site generators</h2>

<p>The setup has been pretty handy over the last few years. But lately, every time I go to write something, I have to fight the setup a bit. I've never really written any Ruby, so I don't have a reliable Ruby environment on my machine.

<p>I started trying to move things over to Hugo.
<!-- TODO link hugo -->
It has the static binary thing going for it: drop it in version control and you'll always be able to build your site.

<p>But after a couple of hours,
I hadn't made all that much progress,
and it just felt like I was swapping one set of overly complicated things for another.
</section>

<section>
<h2>What I'm doing instead</h2>
<p>I'm going to start with the latest output from my existing site,
and start adding new posts as raw HTML.
They won't look anything at all like the old ones; I'll fix that gradually.

<p>In the meantime, it'll be ugly and inconsistent, and that's <em>fine</em>.

<p>I'll fix pain points as they come up.
I think the first one will be something to generate an RSS or Atom feed,
but we'll see.

I've put an empty file named YAGNI at the root to remind me not to over generalize when solving problems.
You can take a look if you like: <a href="TODO">YAGNI</a>

<a>Some kind of templating might follow,
but I think it'll be some time till I get so annoyed that I fix it.

<h2>Accessibility as a side effect</h2>

<p>Writing plain HTML has had a pleasant side effect while writing
</article>
