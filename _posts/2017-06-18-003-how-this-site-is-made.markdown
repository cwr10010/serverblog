---
layout: post
title:  "How this site is made"
date:   2017-06-18 15:09:00 +0200
categories: webserver jekyll
---
This is the third posting on this blog already. But how do I actually tame this thing? To make it clear, this is no Wordpress. The web server delivers static files. No PHP, Python, Ruby or what-so-ever involved on serverside. But I don't like writing plain HTML and I am no web designer.

Some weeks ago my fiancier came across some static web site generators. One of them which is [Jekyll][jekyll]{:target="_blank"} is involved here. Jekyll is based on Ruby which caused some doubts in me weather this tool might be of use or not. I have my troubles with this language. But as using Jekyll as a tool in the first place I gave it a try. And what to say? It is pretty nice! You do write files as markdown or many other markups like textile and even org-mode.

So let us install this.

{% highlight sh %}
$ sudo gem install jekyll bundler
{% endhighlight %}

Next we create a new  project:
{% highlight sh %}
$ jekyll new my_project
{% endhighlight %}

This creates a project skeleton we can work on. To dive deeper I refer to the [documentation][jekyll-doc]{:target="_blank"} of jekyll. Just for starters you can add new blog posts just by adding files with the name pattern `<yyyy>-<mm>-<dd>-name.markupext` to the `_post` directory

As I am a developer and admin I want the result to be deployed safely but also automatically onto my server. Said and done, right? Not quiet. To have a little playground for boring weekends I decided to store the project on github and use [Travis-CI][travis-ci]{:target="_blank"} as my continuous integration system of choice.

To connect my github account was pretty easy, actually it was already for some other projects I did last year. And all you need to do to make Travis-CI work is to add a `.travis.yml` to the root of your project that contains just the language:

{% highlight yaml %}
language: ruby
{% endhighlight %}

Hey you might think thats all? Not yet. Travis-CI runds bundler which expects a Rakefile in the project to build that thing. Rakefiles are those ruby makefiles, fully ruby scriptable. So I created one. And as I wanted it to build the jekyll project I let rake (ruby make, got it?) exec a shell command. So here I am fiddling with this language I do not really like. But does it get better? Sure it does. Travis-CI has a new feature in which you can tell it to execute shell commands. Whait what? No Rakefile? Great! So here we go. We tell Travis-CI in the config file the following:

{% highlight yaml %}
script: bundle exec jekyll build
{% endhighlight %}

To make this even sweeter we now can see when the build fails. Via rake the shell process just died and did not tell bundler that it failed. There is rake in between, remember? Now we call jekyll directly via bundler. So now we have the possibility to build the website automatically. But how can I deploy that on my server? The first thing that comes into the mind of a unix admin is rsync. But I do not want the build server to access my server in an insecure fashion. Rsync offers the possibility to send the data via a ssh tunnel. But this requires either username and password or a public/private keypair. The first is out of question. I will never give a valid username and password to a system that I don't know.

So we need to do that with the key pair. But for that I need a private key file on the build server, right? And this has to be there on any build server when I run the upload command. The only way to do that is to provide the key file in the github repo. To protect that file from being read Travis-CI has another nifty feature. You can encrypt files and decrypt them later on the build server. All you need on your system is the key pair and a tool from the Travis-CI guys.

First let us generate the keyfiles
{% highlight sh %}
$ ssh-keygen -t rsa -b 4096 -C 'build@travis-ci.org' -f ./deploy_rsa
{% endhighlight %}

And now let us encrypt that. First we need the `travis` tool:
{% highlight sh %}
$ sudo gem install travis
{% endhighlight %}

Second we login to travis. Travis-CI will ask you for your github credentials:
{% highlight sh %}
$ travis login --auto
{% endhighlight %}

And finally we encrypt the **private** keyfile with this tool.
{% highlight sh %}
$ travis encrypt-file deploy_rsa --add
{% endhighlight %}

This alters the .travis.yml so the file will be decrypted on build time. But we want to change something here. First we need to tell Travis-CI to do the deployment:

{% highlight yaml %}
deploy:
  provider: script
  skip_cleanup: true
  script: rsync -r --delete-after --quiet $TRAVIS_BUILD_DIR/_site <username>@<domain>:/path/to/site
  on:
    branch: master
{% endhighlight %}

And now we want the private keyfile decrypted right before we deploy. So add the following, best before the `deploy` section:

{% highlight yaml %}
before_deploy:
- openssl aes-256-cbc -K $encrypted_<...>_key -iv $encrypted_<...>_iv -in deploy_rsa.enc -out /tmp/deploy_rsa -d
- eval "$(ssh-agent -s)"
- chmod 600 /tmp/deploy_rsa
- ssh-add /tmp/deploy_rsa
{% endhighlight %}

What do we do here? First we decrypt the keyfile with command line in the before_build section. But we write the output to the /tmp directory to prevent jekyll to ever publish this file to the site. The key file is private and should always be. Second we load the ssh agent with its current keys and load the decrypted keyfile. As we set up the decryption before deployment we should remove the section before_build now.

With this setup we can connect to our server - almost. We just need to add the public keyfile to the authorized_keys file of the user on our server. This should make the system work. We now can deploy to the web server. But wait! The Travis-CI build agent wants us to interactively acknowledge the fingerprint of our server. This is impossible for us! So how do we do? Easiest is to deactivate the check. So we do. But keep in mind, you should never do that if you transmit private data. You open gates for men in the middle attacs! Here it is just some static files that will later be available via the web anyway.

{% highlight yaml %}
deploy:
 script: rsync -e "ssh -o StrictHostKeyChecking=no" -r --delete-after --quiet $TRAVIS_BUILD_DIR/_site <username>@<domain>:/path/to/site
 {% endhighlight %}

Finally we should delete the private key from the tmp folder and remove the key from the ssh agent. Both should stay there as short as possible.
{% highlight yaml %}
after_deploy:
- ssh-add -d
- rm /tmp/deploy_rsa
 {% endhighlight %}

 Last but not least we need to configure our web server. I use Nginx for that. The config file is very simple as we only serve static files.

 {% highlight nginx %}
server {
        listen XXX.XXX.XXX.XXX:80;
        root /path/to/site/_site
        index index.html;
        server_name domain.name;
        location / {
                try_files $uri $uri/ =404;
        }
}
{% endhighlight %}

That's it!

_FIN_.

[jekyll]: https://jekyllrb.com
[jekyll-doc]: https://jekyllrb.com/docs/home/
[travis-ci]: https://travis-ci.org
[travis-doc]: https://docs.travis-ci.com
