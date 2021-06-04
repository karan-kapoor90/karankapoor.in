---
layout: post
categories: [Tutorials, Security]
comments: true
excerpt: A simple guide to setup your own local Certificate Authority for development
title: Setting up a local Certificate Authority
tags: [mkcert, beginners, security, certificate]
description: When setting up platforms and systems, one often needs to supply signed certificates for domains, systems and services.
---

More often than not, systems, softwares, websites and services etc. that we create using technology, need to talk to each other. In doing so, security that helps services talk securely is super important. Insecure communication is by definition, a malpractice, meant only for when one is trying new things, developing or rapidly prototyping solutions. 

When we need services to talk to each other, a certificate authority, in the simplest form, becomes the middle actor that basically validates that both the parties communicating are who they say they are. 

The little `https`, we see in web addresses, is a sign that the website is a trusted resource and that your browser trusts the certificate confirming the identity of the website. This is because the certificate presented by the website to your browser is signed by a certificate authority that your browser trusts. How does my browser know which certificate authority to trust, you ask? Your browser comes pre-installed with identity of these certificate authority. 

So when you create your own Certificate Authority, you too can get your browser (and any other browser on anyone else's machine) to trust any websites signed by your CA, if you install the identity of this CA that you created, into the browser. 

So how do I create my own CA?


# Creating your own Certificate Authority

`mkcert` is a cool little utility that I like to install on my machine, which acts like a local CA. If you run a company, you too can install mkcert on a machine, and use it as described in the steps coming up ahead to get all machines inside your company to trust certificates signed by your CA. 


# Getting started

First you need to install mkcert. 

One can follow the steps here to setup mkcert as per your OS - https://github.com/FiloSottile/mkcert 

If you're on a Mac or a Linux, and you use Homebrew, then you could simply 

```bash
brew install mkcert
```

Once done, just install mkcert using the following

```bash
mkcert -install

# Output 

Created a new local CA at "/Users/karan/Library/Application Support/mkcert"
The local CA is now installed in the system trust store!
```

# Creating a certificate for your website

Now, you're one step closer to creating your own `google.com`. Just kidding. 

But if your website running on your let's say a domain like `mydomain.yo`, you can use mkcert as follows to create a domain certificate signed by your CA:

```bash
mkcert mydomain.yo 127.0.0.1

# here 127.0.0.1 is the IP address of your local machine on which your website is running.
```

# Using the cert with your website

Now your website, for example, would be served by a web-server typically. Meaning that all requests coming to your local machine, requesting for the web pages comes to your web server. Hence the web server needs to present the users with it's own certificate signed by the CA guaranteeing authenticity.

> At this point, only your machine - on which you setup mkcert and did the install knows the local CA you created as a valid CA. Other machines will still see the website's certificate as one signed by an unknown authority since they don't have the root CA certificate installed.

Add the website's certificate to the web server.

If you're using Nginx, the configuration would look something like this:

```
server {
    listen  443 ssl;
    server_name  mydomain.yo;
    root  /Users/karan/Sites/mydomain.yo/public;
    ssl_certificate     /Users/karan/Sites/mydomain.yo/mkcerts/mydomain.yo.pem;
    ssl_certificate_key /Users/karan/Sites/mydomain.yo/mkcerts/mydomain.yo-key.pem;
}
```

In case you're using Apache as the web server, the configuration would look like

```xml
<VirtualHost 127.0.0.1:443>
ServerAdmin webmaster@mydomain.yo
DocumentRoot /Users/karan/Sites/mydomain.yo/public
ServerName mydomain.yo
SSLEngine on
SSLCertificateFile /Users/karan/Sites/mydomain.yo/mkcerts/mydomain.yo.pem
SSLCertificateKeyFile /Users/karan/Sites/mydomain.yo/mkcerts/mydomain.yo-key.pem
</VirtualHost> 
```

And voila, your local website now has a certificate and will encrypt all messages coming to it, and the encryption will be recognized by your browser as well.


# Sharing the CA root certificate with other machines/ users

In order to get the location of the caroot certificate installed on your machine, use the following command:

```bash
mkcert -CAROOT
```

This will give you the location of the root CA certificates you created earlier. 

> Never share the private key of the CA root, as this allows others to sign and create their own certificates signed by the CA. Only share the public certificate.

From the location from the command above, export the `rootCA.pem` file, and share it with the other users. The users should then copy them into a location such as `/Users/karan/cacerts` and then set the `$CAROOT` environment variable on their machine. 

The user needs to then run `mkcert -install` following the installation steps from the beginning.


# An alternative solution

It's fairly painful to pass along the CA certificate and get loads of users to install it into their machines (and hopefully get it right), so a good alternative solution is to use `letsencrypt` at https://letsencrypt.org.

> Let’s Encrypt is a free, automated, and open certificate authority (CA), run for the public’s benefit. It is a service provided by the Internet Security Research Group (ISRG). We give people the digital certificates they need in order to enable HTTPS (SSL/TLS) for websites, for free, in the most user-friendly way we can. We do this because we want to create a more secure and privacy-respecting Web.

> Certificates that use letsencrypt as a CA typically have a 3 month validity, so be mindful of it - certs will need to be replaced after that.