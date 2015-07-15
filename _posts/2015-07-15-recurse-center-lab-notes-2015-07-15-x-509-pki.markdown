---
title: "Recurse Center lab notes 2015-07-15: X.509 PKI"
date: 2015-07-15
permalink: /blog/2015-07-15/recurse-center-lab-notes
---

I spent today working towards my Kubernetes cluster. On the way, I decided I
wanted to have client certificates to control access to the API server.

Here's an overview of where I'm going:

- generate a root certificate and key, ideally stored offline and in a hardware
  security module (HSM)
- use that to sign an intermediate root certificate
- use the intermediate root to sign any client or server certificates

I'm just tinkering for now, and so I'm storing the root certificate on my
laptop. I'm using easy-rsa to generate the setup. It's a collection of
scripts from the OpenVPN project to help set up a CA to use for VPN
authentication.

I'm roughly following these guides, and doing peripheral reading up on what I'm
actually doing:

- [https://openvpn.net/index.php/open-source/documentation/miscellaneous/77-rsa-key-management.html][guide1]
- [https://wiki.archlinux.org/index.php/Create_a_Public_Key_Infrastructure_Using_the_easy-rsa_Scripts][guide2]

[guide1]: https://openvpn.net/index.php/open-source/documentation/miscellaneous/77-rsa-key-management.html
[guide2]: https://wiki.archlinux.org/index.php/Create_a_Public_Key_Infrastructure_Using_the_easy-rsa_Scripts

A fun part is I get to name my CA. I went with ‘Kamal Marhubi CA’, but I'm
probably going to redo it. I also have to get a better handle on all these `O`,
`CN`, `C`, and other fields that show up in an X.509 certificate, and what's
allowed there. [RFC 5280][rfc-5280] is my friend here.

[rfc-5280]: https://tools.ietf.org/html/rfc5280

Once done, I'll have to keep running these scipts whenever I need new
certificates. That's just fine for my small use case! I'll need a server
certificate for each of the API server nodes, client certificates for each
worker node, and a client certificate for me to connect to the API server.

If you want to get fancy, CloudFlare have [a good post on setting up fancier
PKI][cf-post] using their [CFSSL] project. It includes an API server that can
issue certificates automatically.  This would be useful if you wanted your
services to be authenticated, but also to be able to spin new instances up and
down dynamically.

[cf-post]: https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/
[cfssl]: https://blog.cloudflare.com/how-to-build-your-own-public-key-infrastructure/

The CloudFlare post talks about using multiple intermediate roots for different
services. The example is that an API server only trusts certificates signed by
the DB root CA, and vice versa. A comment there mentions using [Extended Key
Usage (EKU)][eku] instead. This is a standard way to designate the type of use
a certificate is valid for. This requires an OID. For external use, you'd have
to register one. But in this use case, you can use aUUID based OID. Some more
info for my future self: [one], [two].

[eku]: https://tools.ietf.org/html/rfc5280#section-4.2.1.12
[one]: http://www.itu.int/en/ITU-T/asn1/Pages/UUID/uuids.aspx
[two]: http://www.oid-info.com/get/2.25

For maximal fanciness, you want to be using HSMs and TPMs where possible. Both
are special hardware that stores keys, and is able to do cryptographic
operations without the key ever leaving the device. You probably have a TPM in
the computer you're using right now, though it might be disabled. I enabled
mine in the BIOS setup, and have ‘taken ownership’ of it. As in, I literally
ran a command called `tpm_takeownership`. I'm going to see if I can get it
added as a security device in Firefox. That way, when the time comes, I'll be
able to store my client certificates in there. Or something!
