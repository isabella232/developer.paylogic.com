:title: Using Debian packages for Python deployments
:summary: At Paylogic we use Debian packages to deploy our Python applications. This article explains how we got started.
:category: DevOps
:author: Peter Odding
:date: 2013-05-14 00:28
:slug: articles/debian-packages
:tags: python, deployment, automation, debian, packaging
:email: peter@peterodding.com

.. contents:: Table of Contents
   :depth: 2

At Paylogic we use Debian packages to deploy our Python applications. This
article explains how we got started.

This is a large article but if you just want to get started building Debian
packages you can jump straight to the section on `getting started with Debian
packaging`_.

Disadvantages of Python packages
################################

Python has its own packaging infrastructure and there are a lot of people who
like it, but for us it doesn't come close to a full solution:

- PyPI_ and/or distribution websites go down regularly, usually at the exact
  time you need them to perform a live deployment :-)

- Python packages cannot and therefore do not declare their binary dependencies,
  because there is no portable way to do so (the packages names are different
  in every Linux distribution, let alone other operating systems)

- Python packages don't control init.d scripts, cron tabs, configuration files,
  etc. while we really do need to install and manage these files...

- What's with the whole Distutils_, Setuptools_, Distribute_, Distutils2_ and
  Distlib_ confusion?! Please for the love of god just merge the common
  subset, bless one tool and get this whole mess over with already! For more
  details about this subject see `Differences between distribute, distutils,
  setuptools and distutils2? <http://stackoverflow.com/questions/6344076/differences-between-distribute-distutils-setuptools-and-distutils2/14753678#14753678>`_

Virtual environments
====================

There's one more significant thing, which is that Python packages favor virtual
environments over system wide installations. Why do we say this?

- ``easy_install`` doesn't support removal of packages
- ``pip`` does support removal of packages but does not support anything like
  ``apt-get autoremove``

One of our problems with virtual environments is that we've seen them break in
various ways, for example because of security updates to the system-wide Python
installation (Google for `ImportError: cannot import name urandom`_). This is
one of the reasons why we prefer virtual machines over virtual environments to
isolate our production deployments.

Making sense of the Python packaging ecosystem
==============================================

As mentioned above the Python packaging ecosystem is a bit of a mess. For an
overview of the situation and some of the problems, refer to the following
external resources:

- `The Hitchhiker's Guide to Packaging: Current State of Packaging <http://guide.python-distribute.org/introduction.html#current-state-of-packaging>`_
- `Python Packaging: Hate, hate, hate everywhere <http://lucumr.pocoo.org/2012/6/22/hate-hate-hate-everywhere/>`_
- `setuptools versioning - wtf? <http://blog.workaround.org/setuptools-versioning-wtf>`_

I sometimes hear people call Debian package management complex. They certainly
have a point, but as a DevOps who wants to do their work properly, Debian and
Python are both complex, the difference is that Debian is (mostly) a pleasure
to work with... The Python packaging ecosystem is very fragmented and
underdocumented while in Debian there is almost always a canonical, documented
way to do things properly.

Advantages of Debian packages
#############################

Here are a couple of notable advantages of using Debian packages:

- They provide a controlled process for installing, removing, upgrading and
  downgrading packages (for example for doing new releases, but also rolling
  back existing releases)

- Dependencies on operating system packages are formalized as proper package
  dependencies instead of being written down in wiki pages, personal notes, or
  worse, not written down at all...

- The steps that should be executed in every environment where a package is
  deployed are formalized in pre/post installation/removal scripts

- The packages are built on a dedicated host so production machines don't need
  a build environment

There's also the fact that we get to use ``apt-get`` more and we (generally)
love ``apt-get`` :-)

.. _getting started with Debian packaging:

Getting started with Debian packaging
#####################################

Debian package management is a complex topic, however getting started requires
little upfront knowledge nor does it require a complex build environment. In
fact most of the tools you'll need are probably already installed if you're
running Debian or a derivative Linux distribution!

Creating your first Debian package
==================================

To create a simple Debian package we only need a single file and a single
command. Here's how you get started:

.. code-block:: sh

   # Create a directory to hold the files contained in the package.
   mkdir my-package && cd my-package

   # Create a directory with the package's control files.
   mkdir DEBIAN

   # Create the main control file with package metadata.
   cat > DEBIAN/control << EOF
   Package: name-of-package
   Version: 1.0
   Section: universe/web
   Priority: optional
   Architecture: all
   Installed-Size: 1
   Maintainer: $USER
   Description: Explanation of why name-of-package is so cool
   EOF

   # Build the package using the accepted naming scheme.
   NAME=$(awk '/^Package:/ {print $2}' DEBIAN/control)
   VERSION=$(awk '/^Version:/ {print $2}' DEBIAN/control)
   ARCH=$(awk '/^Architecture:/ {print $2}' DEBIAN/control)
   dpkg-deb --build . $NAME_${VERSION}_$ARCH.deb

Assuming you're on a Debian/Ubuntu system, the above commands should be enough
to build a simple package. Any files in the working directory (excluding the
special ``DEBIAN`` directory) will be included in the package as if the
directory containing the ``DEBIAN`` package is the root of the file system.

The resulting ``*.deb`` file can be installed using ``dpkg -i $filename``,
however this won't automatically install dependencies, instead ``dpkg`` will
error out when dependencies are missing... When this happens you can run
``apt-get install -f`` to install the dependencies. After that you can rerun
the ``dpkg`` command; it should now succeed. Read on if you're interested in a
more streamlined experience.

Creating a Debian package repository
====================================

As mentioned earlier ``dpkg`` doesn't automatically pull in dependencies. If
you use ``apt-get`` it will do what you expect however ``apt-get`` does not
support installation of local ``*.deb`` archives; it needs a repository. In
other words, once you start using dependencies you will want to setup a Debian
package repository for your packages! Here's how you get started:

.. code-block:: sh

   # Create repository layout, copy packages.
   mkdir -p repo/binary
   cp *.deb repo/binary
   cd repo

   # Create list of packages.
   rm -f Packages Packages.gz Release Release.gpg  # cleanup after previous run
   dpkg-scanpackages -m . | sed 's@: \./@: @' > Packages

   # Create compressed copy of list.
   cat Packages | gzip > Packages.gz

   # Generate release file.
   rm -f Release Release.gpg  # cleanup after previous run
   LANG= apt-ftparchive release . > Release.tmp
   mv Release.tmp Release

   # Sign release file.
   rm -f Release.gpg  # cleanup after previous run
   gpg -abs -o Release.gpg Release

There are several gotcha's in the above piece of shell script:

- We cleanup generated files from previous runs because their presence
  corrupts the generated files

- We modify the output of ``dpkg-scanpackages`` to change for example
  ``./test_1.0_all.deb`` to ``test_1.0_all.deb`` (for some reason ``apt-get``
  doesn't like ``Packages`` files with leading ``./`` fragments)

- We clear the ``$LANG`` environment variable so that we are sure the
  ``Release`` file is properly formatted regardless of the value of ``$LANG``

- You need to have a private GPG key to sign the ``Release`` file; if you don't
  have one yet you'll need to create one using the command ``gpg --gen-key``
  (you may find this `GPG quick start`_ useful)

Hosting the package repository
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After running the above commands, the directory `repo/` can be served using a
regular web server (e.g. Apache_ or Nginx_). No specific configuration is
required because the repository contains only static files.

Using the package repository
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The package repository can be registered in a Debian/Ubuntu system by creating
the file ``/etc/apt/sources.list.d/example.sources.list`` with the following
contents::

   deb http://server-address ./

Installing the GPG key
~~~~~~~~~~~~~~~~~~~~~~

Before the package repository can be used from remote machines, the GPG key
used to sign the Release file has to be installed on the remote machines.
Assuming you have SSH and sudo access to the server where you generated the GPG
key `and` the one where you want to install the GPG key, the following command
will install the GPG key:

.. code-block:: sh

   ssh build-server sudo -i gpg --armor --export | ssh target-host sudo apt-key add -

Wrapping up
###########

That's it really, at least to get started. Now consider how easy it is to write
some Python scripts that automatically build these packages for you based on
the contents of one or more version control systems and suddenly you're looking
at a viable deployment strategy!

There are even people who build Python `virtual environments`_ and ship those
in Debian packages. It may sound revolting at first, but give it a moment to
sink in; it has its advantages :-)

In future articles we'll dive into more advanced topics like pre/post
installation/removal scripts, dpkg triggers and generation of configuration
files. Stay tuned!

.. External references:
.. _`ImportError: cannot import name urandom`: https://www.google.com/search?q=ImportError%3A%20cannot%20import%20name%20urandom
.. _Apache: http://httpd.apache.org/
.. _Distlib: https://pypi.python.org/pypi/distlib
.. _Distribute: https://pypi.python.org/pypi/distribute
.. _Distutils2: https://pypi.python.org/pypi/Distutils2
.. _Distutils: http://docs.python.org/2/library/distutils.html
.. _GPG quick start: http://www.madboa.com/geek/gpg-quickstart/
.. _Nginx: http://nginx.org/
.. _PyPi: https://pypi.python.org
.. _Setuptools: https://pypi.python.org/pypi/setuptools
.. _virtual environments: http://www.virtualenv.org/en/latest/
