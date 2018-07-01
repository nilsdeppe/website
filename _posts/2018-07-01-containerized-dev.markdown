---
layout: post
title:  "Containerized Development With Singularity"
date:   2018-07-01 12:00:00
categories: clang c++ cpp
comments: True
permalink: posts/containerized-dev
shortUrl: https://goo.gl/fRm6i3
is_post: true
---
I've been wanting to try using containers for managing different development
environments for a while now. [Docker](https://www.docker.com/) is a well known
container software, but I've found it not be well-suited for
development. [Singularity](http://singularity.lbl.gov/) is an alternative that
is designed for high performance computing and not microservices. It integrates
more readily with the host OS, while being more secure than
Docker. Interestingly, in some applications Singularity containers actually
[perform better than bare-metal](https://arxiv.org/abs/1709.10140)(the native
OS). In this post I will outline what I've done to use Singularity for a
container-based development environment.

Singularity containers use [recipes](http://singularity.lbl.gov/docs-recipes),
which can derive off of a variety of
different sources, such as Docker containers or Singularity
containers. Singularity recipes are analogous to Docker files. Setting up a
development environment requires being aware of a few subtleties:
- You can share the host init file for your shell (`~/.bashrc`, for example) by
  launching the Singularity container using `singularity exec ubuntu.simg
  /bin/bash`. However, since the configuration in the container is different
  from the host you may need to tweak your init files to work in both
  environments.
- Having a modules (see [LMod](https://lmod.readthedocs.io/en/latest/)) system
  both inside and outside the container can lead to difficulty. You need to run
  `module purge` before starting the Singularity container.
- You can run a different OS in the container than the host under almost all
  circumstances. The only exception is when you need to install software that is
  compatible with the host kernel since that may not be available from the
  container OS's package manager. The Linux
  [perf](https://perf.wiki.kernel.org/index.php/Main_Page) utility is the
  example I encountered.
- The locale inside the container must be set to the locale of the host OS.
  Without setting the locale correctly certain fonts like
  [Powerline](https://github.com/powerline/fonts) won't work and perl gives
  warnings about the incorrect locale settings. Unfortunately, setting the
  locale is OS dependent, but I will show how to do it for Ubuntu and Arch
  Linux.

#### Sharing init files
To enable my `~/.zshrc` to work both inside and outside the container I have the
container set the environment variable `IN_CONTAINER=true`. In Singularity
recipes this is achieved using:

{% highlight shell linenos %}
%environment
    export IN_CONTAINER=true
{% endhighlight %}

Inside my `~/.zshrc` I have the following function that returns true if
evaluated inside a container:

{% highlight shell linenos %}
# Check if we are running inside a container
in_container() {
    if [[ $IN_CONTAINER == "true" ]]; then
        return 0
    fi
    return 1
}
{% endhighlight %}

I then use the `in_container` function throughout my `~/.zshrc` file to control
exactly what is executed inside the container and outside.

#### Using modules
In order to use modules both inside and outside the container I set
up an alias that purges the modules, drops into a container, and reloads the
modules after I exit the container. Here is what I do:

{% highlight shell linenos %}
if ! in_container; then
    # Load modules for host OS
    load_modules() {
        module load \
               blaze-3.2-gcc-7.2.0-mnkglif \
               brigand-master-gcc-6.3.1-pcaobur \
               libxsmm-1.8.1-gcc-6.3.1-mevsuzx \
               openblas-0.2.19-gcc-6.3.1-if6giqo \
               gsl-2.3-gcc-6.3.1-hsntrnj \
               yaml-cpp-master-gcc-6.3.1-gmc3k4n \
               catch-2.1.0-gcc-7.3.0-do5wfi3 \
               kvasir-mpl-develop-gcc-7.2.0-hr2lgqe
    }
    load_modules

    # Drop into a Singularity dev environment, reset modules after
    container() {
        if [ "$1" = "arch" ]; then
            module purge && \
                singularity exec \
                            ~/Research/singularity/arch.simg \
                            /bin/zsh && \
                load_modules
        elif [ "$1" = "ubuntu" ]; then
            module purge && \
                singularity exec \
                            ~/Research/singularity/ubuntu.simg \
                            /bin/zsh && \
                load_modules
        fi
    }
fi
{% endhighlight %}

The `container` function can be called to drop into a Singularity container of
your choice. In my case they all load Zsh as the default shell, but you can
change this back to bash if you prefer. Since the system modules can interfere
with the modules in the container we first purge the system modules, then launch
the container, and after exit reload the host modules.

#### Dealing with kernel dependencies
For my work I really care about performance of code, and so optimization is
something I spend a fair amount of my time doing. This means I need to be able
to use Linux perf to see how software and hardware are interacting. Since
you cannot use perf version that is older than your kernel version, using perf
might require you to use the same OS inside and outside the container. For
example, I cannot use perf if I use an Ubuntu container since there is no
perf available that is compatible with the kernel version. Using an Arch Linux container
allows me to easily install a recent version of perf that is compatible with my
kernel. The good news is that for most people the OS won't matter, and if it
does the solution is fairly straightforward: use the same OS as your host.

#### Setting the locale/Powerline fonts support
If the locale inside the Singularity container is not set correctly then fonts
like [Powerline](https://github.com/powerline/fonts) won't work. I use the
[Agnoster Zsh theme](https://github.com/agnoster/agnoster-zsh-theme), which uses
Powerline fonts. Setting the locale is OS dependent so you'll need to figure out
how to do it for the OS you're using in the container. Here is what is necessary
for an Ubuntu recipe:

{% highlight shell linenos %}
%post
    # In order to get locales working properly inside a Singularity container
    # we need to do the following:
    apt-get update && apt-get install -y \
            locales language-pack-fi language-pack-en
            export LANGUAGE=en_US.UTF-8 && \
            export LANG=en_US.UTF-8 && \
            export LC_ALL=en_US.UTF-8 && \
            locale-gen en_US.UTF-8 && \
            dpkg-reconfigure locales
{% endhighlight %}

and what is necessary for an Arch Linux recipe:

{% highlight shell linenos %}
%post
    # Use US english UTF-8 locale
    sed -i 's/#en_US.UTF-8/en_US.UTF-8/g' /etc/locale.gen
    locale-gen en_US.UTF-8
    pacman -Sy --noconfirm zsh
{% endhighlight %}

#### Tweaking recipe files and incremental builds
One nice feature of Singularity containers (similar to Docker containers) is
that incremental builds will use the existing image to run
the recipe rather than an empty one. Let me give an example. Say I'm building
the recipe called `Ubuntu.sh`, then I would run `sudo singularity build
ubuntu.simg Ubuntu.sh`. If I want to add a new package using `apt-get` I
can comment out all the commands under the `%post` section that built the
existing image and only have the new commands. Then when I run `sudo singularity
build ubuntu.simg Ubuntu.sh`, the additional command will be run and the new
image is much faster to build.

Specifically, say the example recipe file below is built.

{% highlight shell linenos %}
Bootstrap: docker
From: ubuntu:18.04

%post
    apt-get -y update
    apt-get -y install wget
{% endhighlight %}

If the line `apt-get -y install emacs` is added to the file like:

{% highlight shell linenos %}
Bootstrap: docker
From: ubuntu:18.04

%post
    apt-get -y update
    apt-get -y install wget
    apt-get -y install emacs
{% endhighlight %}

When the image is rebuilt only the new command will be executed and the result
added to the image.

#### Summary
In this post I describe some of the troubles with using Singularity containers
for daily development and how to overcome them. Specifically, I address how to
use modules, deal with kernel dependencies, and set the locale to get Powerline
fonts to work inside Singularity. I'm sharing the container recipes that I use
for development on [GitHub](https://github.com/nilsdeppe/MyEnvironment). Overall
Singularity containers are a nice way to develop software, especially if the
development is shared by many people and there is a long list of
dependencies. For the project I work on mainly,
[SpECTRE](https://github.com/sxs-collaboration/spectre), we've been using
Docker containers for nearly a year now and just adopted Singularity as an
alternative to make local development easier.

I hope you find this post helpful and please feel free to ask any questions in
the comments below.
