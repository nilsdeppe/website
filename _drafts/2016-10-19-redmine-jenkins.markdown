---
layout: post
title:  "Setting up Redmine Alongside Jenkins"
date:   2016-08-20 11:00:00
categories: jenkins redmine wiki issue tracker
comments: True
permalink: posts/redmine-alongside-jenkins
shortUrl: http://goo.gl/tSQ1WY
is_post: true
---

Port already used gives:

Job for nginx.service failed because the control process exited with error code. See "systemct
l status nginx.service" and "journalctl -xe" for details.

Check services using ports:

`sudo lsof -i -P | grep -i "80"`

e.g. return: `aolserver 1361 www-data    4u  IPv4  20332      0t0  TCP localhost:80
(LISTEN)`

htmlentities error with version 3.2.1:

redmine@blofeld:~/redmine$ bundle exec rake generate_secret_token
/usr/local/rvm/gems/ruby-2.2.3/gems/htmlentities-4.3.1/lib/htmlentities/mappings/expanded.rb:4
65: warning: duplicated key at line 466 ignored: "inodot"

this issue is solved be removing the gem (which will warn about dependencies):
gem uninstall htmlentities -v '4.3.1'
and installing
gem install htmlentities -v '4.3.4'
Then you must update /home/redmine/redmine/Gemfile.lock to use
htmlentities v 4.3.4 instead of v 4.3.1 and restart nginx.

http://www.redmine.org/plugins/redmine_multiprojects_issue INSTALL
REQUIRED PLUGIN FIRST!!

bundle exec rake redmine:plugins NAME=redmine_multiprojects_issue RAILS_ENV=production

Theme: https://www.redminecrm.com/pages/circle-theme

Slack integration: https://github.com/planio-gmbh/redmine-slack

Redmine plugins: http://plan.io/blog/post/142349004143/redmine-plugins-the-top-9-planio-approved-plugins


https://github.com/Undev/redmine_issue_checklist

Useful links:

https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+on+Ubuntu

https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-with-ssl-as-a-reverse-proxy-for-jenkins

https://gist.github.com/stevegrunwell/6286357

http://blog.bigpixel.ro/2011/12/syncronizing-redmine-bitbucket-repositories/

https://bitbucket.org/divcad/redmine_bitbucket

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
