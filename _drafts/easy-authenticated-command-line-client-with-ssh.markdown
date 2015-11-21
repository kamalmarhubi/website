---
layout: post
title: Easy authenticated command line API with SSH
---

I came across nifty way to build an authenticated command line API with just
SSH. Makes use of SSH forced commands, which allow limiting a user to a single
command.

You can set up a forced command for a specific public key by prepending its line with `command="some-custom-command"`. The line would looke like:

~~~
command="some-custom-command" ssh-rsa LOOOONNNNGGGSTTTRINNGGFORTHEKEY
~~~

When someone connects using that key, instead of a shell, `some-custom-command` will run. This will also ignore any command they specify, instead putting it in the `SSH_ORIGINAL_COMMAND` environment variable. This gives us:

client running example:

shyamalan$ spoilers create bruce-willis
shyamalan$ spoilers publish bruce-willis

lucas$ spoilers create darth-vader
lucas$ spoilers publish darth-vader

rowling$ spoilers publish snape

You want this to be authenticated so that only JK Rowling can work with her spoilers &c

Setup:

Server with sshd. Create a dedicated user for your command:

~~~
# adduser \
  --system \
  --disabled-password \
  --shell=/bin/sh \
  --group \
  spoilers
~~~


--system makes ita system user, --disabled-password disables password-based login, allowing other methods (like SSH), shell overrides the default of disallowing any login 

k


The main thing is that the `authorized_keys` file allows specifying a command to be run instead of giving the user a shell. The configuration looks like this:

Here's how you can implement an authenticated API using this without needing any HTTP servers or OAUTH or setting HTTP basic auth with `.netrc` or anything else

# Create a dedicated account on the server



Cost:
- each user command will make

`authorized_keys` get scanned on every login attempt, which might eventually get slow.
Ways forward:
- Go's ssh library
- libssh


