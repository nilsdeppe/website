---
layout: post
title:  "CLion as C++ IDE With Charm++ Projects"
date:   2017-01-14 11:00:00
categories: emacs CLion charm charm++ charmc clang c++ cpp ide
comments: True
permalink: posts/clion-ide-charm
shortUrl: http://goo.gl/tSQ1WY
is_post: true
---

A while ago I wrote a [post]({% post_url 2016-07-17-emacs-as-ide %})
on using Emacs as a C++ IDE. That setup seems
to work really well for small to medium sized projects that don't use
certain libraries. However, I've found that the code completion is not
always as consistent and good as I would like. After listening to a
[CppCast][cppcast] interview with Anastasia Kazakova I decided to try
[CLion][clion] out as an IDE. Being a loyal Emacs user I was quite
skeptical initially but
always longed for a really well-developed and smooth workflow,
something that CLion promised.

After experimenting with CLion for maybe 30 minutes I wanted to give
it a test-drive on a large project I'm working on that leverages
Charm++ for parallelization. Charm++ is actually part of the reason
the Emacs setup doesn't work as well as I'd like. It adds a lot of
code that Emacs tries to do code completion on and either I haven't
found the right settings or the plugins just are not quite up to the
task. As it often happens when starting to use a new IDE with an
existing large project, nothing worked out of the box. I quickly
managed to get
the code to compile. Playing around with more toy code allowed me to
better understand how CLion's CMake options worked and soon
thereafter I was compiling the project with CLion. Great, right?

Unfortunately, even though CLion had no troubles compiling the
project it failed with strange "CMake Project" errors. Specifically,
`charmc`, the Charm++ compiler wrapper, would fail with the error
`file with unrecognized extension ...`. This
was very puzzling since I could build the project just
fine. A Google search yielded nothing useful so I decided to solve the
problem myself. Searching through the `charmc` script I found the line
that gives the error and then started experimenting with what changes
would result in successful code compilation and also successful
code completion in CLion. Ultimately I found there were two changes
needed
to `charmc`.  Note that the funny indentation in the following is
because of the tabs in the `charmc` script. First I had to replace the
lines (approximately line 1350)
{% highlight shell %}
	*.C|*.cc|*.cxx|*.cpp|*.c++|*.cu)
		Do $CMK_CXX -I$CHARMINC $CMK_SYSINC $OPTS_CPP_INTERNAL $OPTS_CPP $OPTS_CXX -c $FILE $DESTO
{% endhighlight %}
with
{% highlight shell %}
	*.C|*.cc|*.cxx|*.cpp|*.c++|*.cu)
		# Do $CMK_CXX -I$CHARMINC $CMK_SYSINC $OPTS_CPP_INTERNAL $OPTS_CPP $OPTS_CXX -c $FILE $DESTO
		exec $CMK_CXX -I$CHARMINC $CMK_SYSINC $OPTS_CPP_INTERNAL $OPTS_CPP $OPTS_CXX -c $FILE $DESTO
		exit 0
{% endhighlight %}
This was needed for dealing with code completion and CLion not finding
some of the header files.
Next, to deal with the actual "file extension unrecognized" errors
I changed the lines (approximately line 1380)
{% highlight shell %}
	*)
		Abort "file with unrecognized extension $FILE"
	esac
{% endhighlight %}
to
{% highlight shell %}
	*)
		exec g++ -I$CHARMINC $CMK_SYSINC $OPTS_CPP_INTERNAL $OPTS_CPP $OPTS_CXX "$@"
		exit 0
		# Abort "file with unrecognized extension $FILE"
	esac
{% endhighlight %}
These changes appear to be sufficient for projects that only use
C++. However, if your project also compiles C or Fortran code you may
need to change more of the `charmc` script. Hopefully the above will
guide you on your way to making those changes.

Next I had to get CLion's code completion and analysis toolsto find
the Charm++ headers. For this you will need to explicitly add the
`-I/path/to/charm/include` flag to the `CMAKE_CXX_FLAGS` variable.
This can be done by adding the following to your CMake setup,
{% highlight cmake %}
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -I/path/to/charm/include")
{% endhighlight %}
Now we are most of the way to where we want to be, but unfortunately
CLion still will not recognize C++11, even
if you have set the flag `-std=c++11` (at least using Clang on
macOS). To deal with this I added
{% highlight cmake %}
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -stdlib=libc++")
{% endhighlight %}
to the CMake file. Note that if you do not have `libc++` installed on
your system you can omit the `-stdlib=libc++` flag. On macOS `libc++`
is always installed.
Finally, on macOS it was also necessary to add
{% highlight cmake %}
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mmacosx-version-min=10.7")
{% endhighlight %}
to the CMake file.

With these changes I was able to successfully compile the code base
and also have CLion's code completion and code analysis tools work
correctly. At some point I may submit patches to the `charmc` script
back to the Charm++ project, but for now this is enough to allow me to
have the workflow I've always wanted.

[cppcast]: http://cppcast.com/2016/06/anastasia-kazakova/
[clion]: https://www.jetbrains.com/clion/specials/clion/clion.html?&gclid=Cj0KEQjw_9-9BRCqpZeZhLeOg68BEiQAOviWAs-q4ChSK0G-TnqehxOk13WbDplaR8PoLSEh7W_oWKoaAm_v8P8HAQ&gclsrc=aw.ds.ds&dclid=CNTekdOw0M4CFcZBNwodTJsMWw

[cppcon]: https://www.youtube.com/watch?v=5FQwQ0QWBTU
[brew]: http://brew.sh/
[Bear]: https://github.com/rizsotto/Bear
[RTags]: https://github.com/Andersbakken/rtags
[LLVM]: http://llvm.org/
[ClangFormat]: http://clang.llvm.org/docs/ClangFormat.html
[ECCA]: https://github.com/Golevka/emacs-clang-complete-async
[aaronbedra]: http://aaronbedra.com/emacs.d/
[flycheck]: http://www.flycheck.org/en/latest/
[flymake]: http://www.emacswiki.org/emacs/FlyMake
[helm]: https://github.com/emacs-helm/helm
[helm-ctest]: https://github.com/danlamanna/helm-ctest
[ExtraSnippets]: https://github.com/AndreaCrotti/yasnippet-snippets
[Flyspell]: https://www.emacswiki.org/emacs/FlySpell
[Magit]: https://github.com/magit/magit
[init]: /assets/init.el
