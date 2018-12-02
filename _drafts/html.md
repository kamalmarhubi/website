<!DOCTYPE html>
<title>Writing raw HTML</title>
<article>

  <h1>Writing raw HTML</h1>

  <div class="sidenote-container">

    <p>I'm writing this post by hand in HTML like it's 1998. I'm doing this
    because I got tired of how slow and unreliable my existing static site
    generator setup was. It was preventing me from writing. This post is about
    how I got here.

    <aside role="note">
      1998 is when we got the internet in Oman, so I definitely
      wasn't writing HTML before then.
    </aside>

  </div>

  <section>

    <h2>Where this site started</h2>

    <p>This site has been <a href="https://jekyllrb.com/">jekyll</a>/<a href=
    "https://github.com/octopress/octopress">octopress</a> based since before I
    ever even wrote anything for it.  <a href="https://jvns.ca/">Julia</a>
    helped me set it up <em>just in case</em>. (In true Julia fashion, she wrote
    <a
    href="https://jvns.ca/blog/2014/10/08/how-to-set-up-a-blog-in-5-minutes/">a
    blog post about it</a>.) The idea was to make it so there was no friction if
    I ever wanted to publish something.

    <p>The site actually sat more or less empty for a little over six months
    before I wrote a post:

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
        Output showing <a href="TODO">first commit</a> in early October 2014,
        and <a href="TODO">first post</a> in late May 2015.
      </figcaption>

    </figure>

    <p>I started writing when I went to the <a href=
    "https://recurse.com/">Recurse Center</a>, and having the site good to go
    saved me a bunch of yak shaving in that first week.

  </section>
  <section>

    <h2>Enough with static site generators</h2>

    <p>The setup has been pretty handy over the last few years. But lately,
    every time I go to write something, I have to fight the setup a bit. I've
    never really written any Ruby, so I don't have a reliable Ruby environment
    on my machine.

    <p>I started trying to move things over to <a href="https://gohugo.io/">Hugo</a>. 
    It has the static binary thing going for it: drop it in version control and
    you'll always be able to build your site. No more bundler headaches!
    
    <p>But after a couple of hours, I hadn't made all that much progress. It
    just felt like I was swapping one set of overly complicated things for
    another.

  </section>
  <section>

    <h2>What I'm doing instead</h2>

    <p>I'm going to start with the latest output from my existing site, and
    start adding new posts as raw HTML. They won't look anything at all like the
    old ones; I'll fix that gradually.

    <p>In the meantime, it'll be ugly and inconsistent, and that's
    <em>fine</em>.

    <p>I'll fix pain points as they come up. I think the first one will be
    something to generate an RSS or Atom feed, but we'll see. I've put an empty
    file named YAGNI at the root to remind me not to over generalize when
    solving problems. You can take a look if you like: <a href="TODO">YAGNI</a>.

    <p>Some kind of templating might follow, but I think it'll be some time till
    I get so annoyed that I fix it.

  </section>
  <section>

    <h2>Accessibility as a side effect</h2>

    <div class="sidenote-container">
      <p>A while ago, <a href="https://meowni.ca/">Monica</a> opened <a
      href="TODO">a PR against my site's repo</a> to add <a href="TODO">ARIA</a>
      labels for parts of my site. This accessibility stuff is important, but
      because I didn't really know how my site worked, I couldn't easily add more
      of it.

      <aside role="note">
        It took me more than six months to respond to the PR, partly because I'm
        terrible, but also partly because I hated dealing with the site's setup.
      </aside>
    </div>

    <p>Writing this post directly in HTML makes it easy to use proper semantic
    markup. There are <code>&lt;section&gt;</code>s and
    <code>&lt;aside&gt;</code>s. It gives me an clear place to put in things
    like an <code>aria-labelledby</code> attribute on the <code>git log</code>
    output above, or <code>role</code> attributes.
    
    <p>Of course, there's still loads of room for improvement, but I never would
    have done any of it in the old setup.

  </section>
  <section>

    <h2>some TODO conclusion thing</h2>

</article>
