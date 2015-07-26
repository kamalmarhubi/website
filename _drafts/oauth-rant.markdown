---
title: oauth rant
---

Security things on the internet sometimes make me sadangry. OAuth and the
improper use thereof is on my mind at the moment. Let's start off with a very
quick WEI (what even is) of OAuth, beginning with the abstract from the RFC:

> The OAuth 2.0 authorization framework enables a third-party application to
> obtain limited access to an HTTP service, either on behalf of a resource owner
> by orchestrating an approval interaction between the resource owner and the
> HTTP service.

RFC http://tools.ietf.org/html/rfc6749

There are three players when using OAuth: the HTTP service, the resource owner,
and the third-party application. Using mobile Twitter apps as an example,
- Twitter is the HTTP service;
- you are the resource owner; and
- your mobile Twitter app is the third-party application.

The kinds of ‘limited access’ available are up to the HTTP service—in the case
of Twitter, they include things like ‘post tweets’, ‘read direct messages’,
‘follow users’, and more. The ‘approval interaction’ is usually clicking an
‘Authorize’ button.

The big thing missing in that paragraph is any mention of giving the resource
owner's credentials—your Twitter account name and password—to the third-party
application. In fact, Twitter's OAuth confirmation page makes it clear that you
will not be sharing your password with the third-party application.

So, reading that abstract again,

> The OAuth 2.0 authorization framework enables a mobile Twitter app to be able
> to post tweets and read direct messages on Twitter on your behalf by asking
> you to click ‘Authorize’ on a confirmation page on Twitter's website.

If you have a Twitter account, you've probably done this at some point. I keep
trying new Twitter clients, so I have done it many times!

## Doing OAuth wrong

Sticking with Twitter mobile apps, let's look at what setting up an account
looks like in TODO(someapp):

A mobile app has full control over everything it shows you, and has full access
to all input you give it. From a security perspective, it's safest to assume
everything it shows you is a lie, and that it's secretly logging all input. This
matters especially if you're entering your login credentials to another service.

Here's a quick diagram of a typical Android Twitter app's authorization flow,
with lies and secret logging widgets highlighted:

You're probably used to logging in at that screen and clicking the big green
‘Authorize’ button. So used to it, it's not even clear how to do it better. But
here's another picture, this time showng the same screen in the mobile web
browser:






github's hub tool requires you to type your credentials into a prompt it presents

a third party is authorized to perform some. the aim is to both grant the
authorization, but also limit it to a specific set of actions (called scopes in
OAUTH parlance). By entering credentials into a screen controlled by thethird
party, this is all lost. A malicious application can simply present
somethign that looks "legit", have the user enter their credentials, and
bam now they have full access to the account.

Some examples:

Third party Twitter apps on Android are incredibly bad for this. In
fact, the first party Twitter app does this as well. 


user: you
relying party: ThirdPartyTwitterApp
identity provider: :
how an oauth flow goes:


