---
title: Slots vs namedtuple in Python
date: 2016-09-19T12:46:45-04:00
---


Last week, I got nerd-sniped by a simple question: for value objects in Python, which is faster—`__slots__` or `namedtuple`?

My initial answer was

> My gut feeling is that `__slots__` and `namedtuple` should perform the same for field access with the major difference being how you declare the type, and mutability. Not verified.

> Of course, the usual profile-before-you-optimize and benchmarking advice applies.

Almost immediately after writing that, I realized that a ‘gut feeling’ wasn't exactly a great thing to go on, and I should be able to do better. Rather than immediately benchmarking, though, I let my curiosity about how `__slots__` works get the better of me.

After a bit of poking around in the CPython source code, I had a different hypothesis: `namedtuple` should probably be slower than slots. It turned out I was right—here's how I got there!


## Value objects in Python

First off, let's have a Boring Illustrative Example! Say you're writing something that works with `(x, y)` points in a plane. You can use plain tuples in Python, but indexing to get components is a bit ugly:

~~~
def distance(p1, p2):
    return sqrt((p1[0] - p2[0])**2 + (p1[0] - p2[0])**2)
~~~

and destructuring all of them is verbose *and* ugly:

~~~
def distance(p1, p2):
    p1_x, p1_y = p1
    p2_x, p2_y = p2
    return sqrt((p1_x - p2_x)**2 + (p1_y - p2_y)**2)
~~~

It could be nicer to have a simple object with `x` and `y` attributes. Then we can just write as

~~~
def distance(p1, p2):
    return sqrt((p1.x - p2.x)**2 + (p1.y - p2.y)**2)
~~~


### namedtuple
[`namedtuple`][namedtuple-docs] is a simple way to make a type that does what we want. After declaring a namedtuple,

~~~
Point = namedtuple('Point', 'x y')
~~~

we can create them in a couple of ways:

~~~
p1 = Point(0, 0)
p2 = Point(x=3, y=4)
~~~

Under the hood, the components are stored in a tuple. The `x` and `y` attributes are set up using the `@property` decorator to read from the tuple.

[namedtuple-docs]: TODO


### Slots

Instead of making `Point` a namedtuple, we could make it an ordinary class and use the `__slots__` special member on the class. This stores instance variables packed in a list instead of in a dictionary. [The docs][slots-docs] say the main reason to do this is to save space on the dictionary for classes with only a few attributes.


Here's how the definition of `Point` would look using slots:

~~~
class Point:
    __slots__ = ('x', 'y')

    def __init__(self, x, y):
        self.x = x
        self.y = y
~~~

This is more verbose than using `namedtuple`, and also has less nice stuff. For example, if we used `namedtuple`, we'd get a `__repr__` for free, so that points print out like `Point(x=3, y=4)` instead of `<__console__.Point instance at 0x7fd97a2ad758>`. It's also mutable, while the namedtuple would be immutable.[^fn-attr]

[slots-docs]: TODO

[^fn-attr]:
    A lot of those differences can be sorted out with Hynek's [`attr`][attr-link] module, which [glyph wrote about quite recently][attr-glyph]. It gives you `__init__`, `__repr__` and a bunch of ther magic methods, cutting down on boilerplate. With `attr`, an immutable `Point` using slots would look like this:
    
    ~~~
    import attr
    
    @attr.s(slots=True, frozen=True)
    class Point:
        x = attr.ib()
        y = attr.ib()
    ~~~

[attr-link]: https://hynek.me/projects/attrs/
[attr-glyph]: https://glyph.twistedmatrix.com/2016/08/attrs.html


## How they're implemented

### namedtuple

I already knew roughly how `namedtuple` works: it plugs in the class name and field names into a template that creates a subclass of the builtin `__tuple__` type. This is super clear if you look at [the source][namedtuple-src].

Since we're most particularly interested in how attribute access performs, let's focus on that part. At the end of the `_class_template`, there's a placeholder `{field_defs}`. These get filled in according to `_field_template`:

~~~
## some imports from the _class_template
from builtins import property as _property
from operator import itemgetter as _itemgetter

_field_template = '''\
    {name} = _property(_itemgetter({index:d}), doc='Alias for field number {index:d}')
'''
~~~

So each attribute gets a [`property`][property-docs] that simply looks up attribute's index in the tuple using [`operator.itemgetter`][itemgetter-docs].

As a tiny note, it's interesting that `namedtuple` sets `__slots__` on the generated class to be empty. This prevents creating a dict for each instance, which makes perfect sense, since they attributes are stored in the tuple.

[property-docs]: TODO
[itemgetter-docs]: TODO
[namedtuple-src]: https://hg.python.org/cpython/file/v3.5.2/Lib/collections/__init__.py#l300


#### So, what do property and itemgetter do?

A key thing here is knowing what the `property` builtin does. It creates a [descriptor][descriptor-docs], with `__get__`, `__set__`, and `__del__` methods. If you access an attribute like `p.x`, and `p` has no dict, it will look up `x` in `p.__class__` and attempt to use it as a descriptor, so we end up with something like this:


~~~
## accessing an attribute like this
p.x 

## effectively does  this
p.__class__.x.__get__(p)
~~~

So each field gets a descriptor stored in the class object, and the `__get__` method is the `itemgetter`.

As for `itemgetter`, it's [a pure Python callable object][itemgetter-src] that partially applies the `[]` indexing. So `itemgetter(1)('abcd')` would return `'b'`. The ‘pure Python’ part will be important!

[descriptor-docs]: https://docs.python.org/3/reference/datamodel.html#implementing-descriptors
[itemgetter-src]: https://github.com/python/cpython/v3.5.2/blob/Lib/operator.py#l265


### Slots

This whole rabbithole started when I realized I had no idea how slots were implemented. I knew that they moved attribute storage from a dict to a packed representation, but that was it. This is where I had to poke around the CPython source to see what happens.

I'm not going to excerpt much source here, because it's all in C and so a little on the verbose side. It's also spread across a few files, which makes it harder to But here's a little summary of what declaring `__slots__` on your class does:

- allocates extra space for descriptors at the end of the type object
- creates a descriptor for each slot, and stores it in the class dict
- leaves the `tp_dictoffset` field of the type object as zero, which makes creating an instance not allocate a dict
- 

And when we look up 


## Putting it altogether

When we put all thi
