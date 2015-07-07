---
title: "Recurse Center lab notes 2015-07-06: new Recursers, ES15, planning"
date: 2015-07-06
permalink: /blog/2015/07/06/recurse-center-lab-notes
---

Today was day one of the Summer 2 batch, meaning we got thirty new people! This
also means we had our first day without the Spring 2 batch, who I'll definitely
miss. But several of them showed up for a games night so it's almost like
they're still here.

It was interesting to see the intro talks again from six weeks in. A bit of
deja vu, and a good point to really stop and reflect. The advice and words from
the faculty was different the second time, even though the words were the same.
Our batch also gave advice or other thoughts to the new Recursers—Sophie
[jotted it all down][sophie-advice].

[sophie-advice]: https://sfrapoport.github.io/2015/07/06/Advice-for-Summer-2s.html

## EcmaScript 15

The morning was taken up with welcoming the new batch, so it was a while until
I got back to programming. I decided to start learning JavaScript / ES15 with
[es6kata][]. There's a lot of interdependence between the exercises! I started
off with the Arrays collection, but had to take a detour into destructuring. I
should have also taken a detour into iterables, as I wasn't clear on what an
ES15 iterator was, and it was needed.

[e6kata]: http://es6katas.org/

So far ES15 looks pretty nice! I'm almost glad I put of learning JS for so
long. Destructuring is especially good. It's similar to Python, but a bit more
versatile. The big features I noticed:

- fields of objects can be destructured like so:

  ~~~
  let dessert = { name: "eton mess", tasty: true };
  let {name} = dessert;
  console.log(name);  // => "eton mess"
  ~~~
  
  It was a bit mysterious to me at first, but this also applies to fields on
  builtin types, eg, `length` on `String`:


  ~~~
  let a = "hi";
  let {length} = a;
  console.log(length);  // => 2
  ~~~

  (I just realised I should check if this works for properties with getters or
   not.)

- you can choose a name to bind to different from the attribute name with a
  colon, eg,

  ~~~
  let a = "hi";
  let {length:l} = a;
  console.log(l);  // => 2
  ~~~

- you can supply defaults when destructuring which is cool! It works when you
  use the default name for a binding, as well as when you change it, eg,

  ~~~
  let dessert = { name: "eton mess", tasty: true };
  let {dairyFree=false} = dessert;  // be safe and assume desserts contain dairy
  console.log(dairyFree);  // => false
  ~~~

- all of this comes together most excellently in using destructuring on a
  function argument, eg,

  ~~~
  let dessert = { name: "eton mess", tasty: true };
  let eat = {name:n, tasty, dairyFree:df=false} => {
      if (!df || !tasty) { console.log("no thanks!"); return; }

      console.log(`mmm yes, I'll eat some ${n}!`);
  }

  eat(dessert);  // => "no thanks!"
  ~~~

  The default arguments let you have default values in a nice succinct way, and
  the binding renaming lets you have nice descriptive names in the interface but
  abbreviate in the implementation.

## Planning the second half of RC

Towards the end of the day, I had a worry session with Tom, one of the
facilitators. The short of it is I'm worried I'm not going to complete as many
things as I want to. I've been getting a fair amount of stuff done, but aside
from [`lsaddr`][lsaddr] I haven't got much to show for it. While at RC I want
to get more comfortable completing things.

Part of this is being afraid of spending too much time on a project that ends
up being too big for RC. Of course, at the other end of this, I'm also afraid
of spending too much of my time flitting between things. We ended up at the
idea of spending a solid day on each of 2-3 projects I have in mind to see
where I can get them.

So, my plan for tomorrow is to take [what I've learned about `mmap`][mmap-post] and start
putting it into a Cap'n Proto message builder that writes directly to a memory
mapped file. Wish me luck!

[mmap-post]: http://kamalmarhubi.com/blog/2015/06/24/recurse-center-lab-notes/
[lsaddr]: https://github.com/kamalmarhubi/lsaddr/

