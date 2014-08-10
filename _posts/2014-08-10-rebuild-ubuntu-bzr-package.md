---
layout: post
title: How to rebuild ubuntu memcached package from bzr source to lift up upstream version
---

Read Manual at [Ubuntu Packaging Guide](http://packaging.ubuntu.com/html/index.html)

### Install dev environment

[Getting set up instructions](http://packaging.ubuntu.com/html/getting-set-up.html)

_This step can be skipped partially for our VAGRANT distro._

There are a number of tools that will make your life as an Ubuntu developer much easier. You will encounter these tools later in this guide. To install most of the tools you will need run this command:

    $ sudo apt-get install gnupg pbuilder ubuntu-dev-tools bzr-builddeb apt-file

Note: Since Ubuntu 11.10 “Oneiric Ocelot” (or if you have Backports enabled on a currently supported release), the following command will install the above and other tools which are quite common in Ubuntu development:

    $ sudo apt-get install packaging-dev

Create your GPG key

    $ gpg --gen-key
    $ gpg --send-keys --keyserver keyserver.ubuntu.com <KEY ID>

Create your SSH key

    $ ssh-keygen -t rsa

Set up pbuilder

    $ pbuilder-dist <release> create

Upload your GPG key to Launchpad

To find about your GPG fingerprint, run:

    $ gpg --fingerprint email@address.com

and it will print out something like:

    pub   4096R/43CDE61D 2010-12-06
          Key fingerprint = 5C28 0144 FB08 91C0 2CF3  37AC 6F0B F90F 43CD E61D
    uid   Daniel Holbach <dh@mailempfang.de>
    sub   4096R/51FBE68C 2010-12-06

Then run this command to submit your key to Ubuntu keyserver:

    $ gpg --keyserver keyserver.ubuntu.com --send-keys 43CDE61D

where `43CDE61D` should be replaced by your key ID (which is in the first line of output of the previous command). Now you can import your key to Launchpad.

Upload your SSH key to Launchpad

Configure Bazaar

    $ bzr whoami "Bob Dobbs <subgenius@example.com>"
    $ bzr launchpad-login subgenius

Configure your shell¶

    export DEBFULLNAME="Bob Dobbs"
    export DEBEMAIL="subgenius@example.com"

Fix pbuilder-dist distribution bug under precise [https://bugs.launchpad.net/ubuntu/+source/ubuntu-dev-tools/+bug/1068390](https://bugs.launchpad.net/ubuntu/+source/ubuntu-dev-tools/+bug/1068390)

{% highlight bash %}
pbuilder-dist quantal update
Traceback (most recent call last):
  File "/usr/bin/pbuilder-dist", line 462, in <module>
    main()
  File "/usr/bin/pbuilder-dist", line 456, in main
    sys.exit(subprocess.call(app.get_command(args)))
  File "/usr/bin/pbuilder-dist", line 286, in get_command
    if self.target_distro == UbuntuDistroInfo().devel():
  File "/usr/lib/python2.7/dist-packages/distro_info.py", line 92, in devel
    raise DistroDataOutdated()
distro_info.DistroDataOutdated: Distribution data outdated
{% endhighlight %}

_WORKAROUND:_

In Precise, a quick and dirty fix is to edit the file `/usr/bin/pbuilder-dist` , commenting out lines 286 to 289 by prefixing each line with a #.

### Pull source

[http://packaging.ubuntu.com/html/udd-getting-the-source.html](http://packaging.ubuntu.com/html/udd-getting-the-source.html)

    $ bzr init-repo memcached
    $ cd memcached
    $ bzr branch ubuntu:precise/memcached memcached.dev
    $ cd memcached.dev

### Working on package

[http://packaging.ubuntu.com/html/udd-working.html](http://packaging.ubuntu.com/html/udd-working.html)

    $ bzr branch memcached.dev 1.4.18-0ubuntu1
    $ cd 1.4.18-0ubuntu1

Copy new source into dir. Fix debian/watch:

{% highlight bash %}
version=3
opts=\
downloadurlmangle=s|.*[?]name=(.*?)&.*|http://www.memcached.org/files/$1|,\
filenamemangle=s|[^/]+[?]name=(.*?)&.*|$1| \
http://www.memcached.org/files/memcached-([0-9.]+).tar.gz.*
{% endhighlight %}

Remove old patch debian/patches/60_fix_racey_test.patch and fix patches series file

### Commit changes

    $ bzr st
    $ bzr add ...
    $ dch -i
    $ bzr commit

A hook in bzr-builddeb will use the debian/changelog text as the commit message and set the tag to mark bug #12345 as fixed.

This only works with bzr-builddeb 2.7.5 and bzr 2.4, for older versions use debcommit.

### Rebuild package

    $ bzr builddeb -S
    $ pbuilder-dist precise build ../memcached_1.4.18-0ubuntu1.dsc