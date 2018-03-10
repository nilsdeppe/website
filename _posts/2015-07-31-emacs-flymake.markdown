---
layout: post
title:  "Emacs and Flymake Syntax-Checking"
date:   2015-07-31 15:00:38
categories: general programming C++ emacs IDE python C
comments: True
permalink: posts/emacs-flymake
shortUrl: http://goo.gl/5ZH8ZR
is_post: true
---
[Emacs][3] is a great tool for software development. If you're not using
a more advanced environment than Notepad and a Terminal it may be time
for you to look at options. Two popular ones are [Emacs][3]
and [Vim][4]. Depending
on who you ask you will get widely varying responses about which is
["better"][5]. [Eclipse][6], [Visual Studio][7]
and [Xcode][8] are all popular options as well. Now to get
to the point. Compiling code takes time and having a built-in syntax check
is an excellent way to reduce time spent compiling. For [Emacs][3] there
is [Flymake][9] and [Flycheck][10]. As you can probably guess,
I will discuss [Flymake][9].

[Flymake][9] and [Flycheck][10] are both options for syntax
checking in [Emacs][3]. I use Flymake and am very happy with it
so far. It is packaged with Emacs 24 but is rather dated now.
Flycheck is newer and
as far as I know still maintained. Notice that so far I haven't
specified which language we will be syntax checking? This is because
[Flymake][9] supports a few by default and more via some extensions.
[Flycheck][10] supports 30, so if Flymake doesn't support your language
of choice you should stop reading now and learn to use Flycheck.
I program primarily in C++ and also frequently in python,
both of which Flymake supports.

Let's get started with python. If you already have Flymake installed,
then install `flymake-easy` and `flymake-python-pyflakes` using
`M-x list-packages`. You will also need to install `pyflakes` using
`pip install pyflakes`. In case you didn't know, `pip` is essentially
a module/package manager for python. Now open your Emacs init file
(generally `~/.emacs`) and add the following lines of code to the end.

{% highlight lisp %}
;; Load Flymake
(require 'flymake)
;; Load Flymake-python
(require 'flymake-python-pyflakes)
;; Activate it each time python mode is turned on
(add-hook 'python-mode-hook 'flymake-python-pyflakes-load)
{% endhighlight %}

If Emacs is having issues with finding Flymake you will need to add the
appropriate folders to the `load-path`. In my case I had to add the
following *before* the above `require` lines.

{% highlight lisp %}
(add-to-list 'load-path "~/.emacs.d/elpa/flymake-python-pyflakes-0.9/")
(add-to-list 'load-path "~/.emacs.d/elpa/flymake-easy-0.10/")
{% endhighlight %}

If you are familiar with python I recommend opening up a file and testing
it out to make sure everything works correctly. If it does we can move
on to syntax checking C++.

The above setup will almost work for C++ code. Before
delving into setting up Makefiles for your projects I will share my
Flymake settings with you. I like to have the option to go to the next
and previous error. To include this add the following to your init file:

{% highlight lisp %}
;; Control-c n    next error
(global-set-key (kbd "C-c n") 'flymake-goto-next-error)
;; Control-c p    previous error
(global-set-key (kbd "C-c p") 'flymake-goto-prev-error)
{% endhighlight %}

To run Flymake when a file is opened add

{% highlight lisp %}
;; Run flymake when we open a file
(add-hook 'find-file-hook 'flymake-find-file-hook)
{% endhighlight %}

This can be annoying for projects where you do not have Flymake
setup because you will receive a warning on load. I also like having
Flymake underline the line with an error or warning. This means
adding

{% highlight lisp %}
;; Underline warnings and errors from Flymake
(custom-set-faces
 '(flymake-errline ((((class color)) (:underline "red"))))
 '(flymake-warnline ((((class color)) (:underline "yellow")))))
{% endhighlight %}

By default Flymake shows the error in a tooltip when you hoover
your mouse over the line. To instead have the error display in
the mini buffer when your cursor is on the line add:

{% highlight lisp %}
;; Display error and warning messages in minibuffer.
(custom-set-variables
 '(help-at-pt-timer-delay 0.5)
 '(help-at-pt-display-when-idle '(flymake-overlay)))
{% endhighlight %}

We can now move on to the fun part: adding the syntax-check
to the Makefile of the project. I'm assuming you are familiar with
how Makefiles work. If not, give it a quick
[Google][1] to get the basics and then
we can carry on. Once you have your Makefile set up you will want
to add the following:

{% highlight bash %}
check-syntax:
	g++ -o nul -S ${CHK_SOURCES}
{% endhighlight %}

You can change the compiler you want to use. On my system `g++`
defaults to [clang][2]. If you're compiling with MPI you'll want to
use `mpic++` or whichever MPI wrapper you're using. You can add
flags too. For example, if you want C++11 support, specify
the `-std=c++11` flag.

The above will work if you have a simple build system. For
more complex build systems where the paths of header files are referenced
from a code "home" directory it will fail. I have managed to
successfully deal with this by using the following syntax
check instead.

{% highlight bash %}
# Used for flymake in eMacs
$(CODE_HOME)=/path/to/code/home
check-syntax:
	g++ -I$(CODE_HOME) -std=c++11 -s -o nul -S $(CHK_SOURCES)
{% endhighlight %}

Note that `$(CODE_HOME)` needs to be specified giving the
absolute path to where your header files are referenced from.

I hope you found this useful! Any suggestions or questions are
welcome in the comments below.

[1]: https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=makefile%20tutorial
[2]: http://clang.llvm.org/
[3]: https://www.gnu.org/software/emacs/emacs.html
[4]: http://www.vim.org/
[5]: http://unix.stackexchange.com/questions/986/what-are-the-pros-and-cons-of-vim-and-emacs
[6]: https://eclipse.org/
[7]: https://www.visualstudio.com/en-us/visual-studio-homepage-vs.aspx
[8]: https://developer.apple.com/xcode/
[9]: http://www.emacswiki.org/emacs/FlyMake
[10]: https://github.com/flycheck/flycheck
