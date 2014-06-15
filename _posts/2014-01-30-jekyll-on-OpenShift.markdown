---
layout: post
title: "Hosting a Jekyll instance on OpenShift"
date: 2014-01-30 14:40:48 +0530
comments: true
categories: 
- OpenShift
- jekyll
---

Create a Jekyll OpenShift application
-------------------------------------

Install the RPMs

    $ sudo yum install rubygem-rhc


Create the Jekyll application:

    $ rhc app create jekyll https://raw.github.com/openshift-cartridges/openshift-jekyll-cartridge/master/metadata/manifest.yml 
    The cartridge 'https://raw.github.com/openshift-cartridges/openshift-jekyll-cartridge/master/metadata/manifest.yml' will be downloaded and installed
    
    Application Options
    -------------------
      Domain:     testjekyll
      Cartridges: https://raw.github.com/openshift-cartridges/openshift-jekyll-cartridge/master/metadata/manifest.yml
      Gear Size:  default
      Scaling:    no
    
    Creating application 'jekyll' ... done
    
    
    Waiting for your DNS name to be available ... done
    
    Cloning into 'jekyll'...
    The authenticity of host 'jekyll-testjekyll.rhcloud.com (54.226.238.63)' can't be established.
    RSA key fingerprint is cf:ee:77:cb:0e:fc:02:d7:72:7e:ae:80:c0:90:88:a7.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added 'jekyll-testjekyll.rhcloud.com,54.226.238.63' (RSA) to the list of known hosts.
    
    Your application 'jekyll' is now available.
    
      URL:        http://jekyll-testjekyll.rhcloud.com/
      SSH to:     52e118694382ec02180007e7@jekyll-testjekyll.rhcloud.com
      Git remote: ssh://52e118694382ec02180007e7@jekyll-testjekyll.rhcloud.com/~/git/jekyll.git/
      Cloned to:  /home/kashyap/jekyll
    
    Run 'rhc show-app jekyll' for more details about your app.


Clone the repository:

    $ cd /export/openshift-tinker
    $ git clone ssh://52e118694382ec02180007e7@jekyll-testjekyll.rhcloud.com/~/git/jekyll.git/
    Cloning into 'jekyll'...
    remote: Counting objects: 16, done.
    remote: Compressing objects: 100% (15/15), done.
    remote: Total 16 (delta 0), reused 16 (delta 0)
    Receiving objects: 100% (16/16), 6.36 KiB | 0 bytes/s, done.


Move into the repository and install json and jekyll:

    $ cd jekyll

    $ gem install json jekyll
    Fetching: json-1.8.1.gem (100%)
    Building native extensions.  This could take a while...
    Successfully installed json-1.8.1
    Parsing documentation for json-1.8.1
    Installing ri documentation for json-1.8.1
    Done installing documentation for json after 0 seconds
    Fetching: jekyll-1.4.3.gem (100%)
    Successfully installed jekyll-1.4.3
    Parsing documentation for jekyll-1.4.3
    Installing ri documentation for jekyll-1.4.3
    Done installing documentation for jekyll after 1 seconds
    2 gems installed


Serve locally, to test:

    $ jekyll serve --watch
    Configuration file: /export/openshift-tinker/jekyll/_config.yml
                Source: /export/openshift-tinker/jekyll
           Destination: /export/openshift-tinker/jekyll/_site
          Generating... done.
     Auto-regeneration: enabled
        Server address: http://0.0.0.0:4000
      Server running... press ctrl-c to stop.


The default Jekyll cartrdige file does not list Jekyll blog posts, so
add a simple index.html file:

    $ cat index.html
    ---
    layout: default
    title: Your New Jekyll Site
    ---
    
    <div id="home">
      <h1>Blog Posts</h1>
      <ul class="posts">
        {% for post in site.posts %}
          <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a></li>
        {% endfor %}
      </ul>
    </div>



Add another blog post [Hello Universe],  in the posts directory, commit
and push:

    $ git add index.html

    $ git commit index.html -m "Add a proper Jekyll index.html file"

    $ git add _posts/2014-01-23-hellouniverse.markdown

    $ git commit _posts/2014-01-23-hellouniverse.markdown -m "Add another simple Mardown blog post"

    $ git push
  
    Counting objects: 8, done.
    Delta compression using up to 4 threads.
    Compressing objects: 100% (6/6), done.
    Writing objects: 100% (6/6), 989 bytes | 0 bytes/s, done.
    Total 6 (delta 2), reused 0 (delta 0)
    remote: Stopping jekyll cart
    remote: Sending SIGTERM to jekyll:95399 ...
    remote: Building git ref 'master', commit df37147
    remote: Preparing build for deployment
    remote: Deployment id is 00a5680a
    remote: Activating deployment
    remote: Starting jekyll cart
    remote: Executing bundle install
    remote: Fetching gem metadata from https://rubygems.org/........
    remote: Fetching gem metadata from https://rubygems.org/..
    remote: Resolving dependencies...
    remote: Using blankslate (2.1.2.4) 
    remote: Using fast-stemmer (1.0.2) 
    remote: Using classifier (1.3.4) 
    remote: Using colorator (0.1) 
    remote: Using highline (1.6.20) 
    remote: Using commander (4.1.5) 
    remote: Using ffi (1.9.3) 
    remote: Using liquid (2.5.5) 
    remote: Using rb-fsevent (0.9.4) 
    remote: Using rb-inotify (0.9.3) 
    remote: Using rb-kqueue (0.2.0) 
    remote: Using listen (1.3.1) 
    remote: Using maruku (0.7.1) 
    remote: Using posix-spawn (0.3.8) 
    remote: Using yajl-ruby (1.1.0) 
    remote: Using pygments.rb (0.5.4) 
    remote: Using redcarpet (2.3.0) 
    remote: Using safe_yaml (0.9.7) 
    remote: Using parslet (1.5.0) 
    remote: Using toml (0.1.0) 
    remote: Using jekyll (1.4.3) 
    remote: Using json (1.8.1) 
    remote: Using bundler (1.3.5) 
    remote: Your bundle is complete!
    remote: Use `bundle show [gemname]` to see where a bundled gem is
    installed.
    remote: Starting Jekyll server
    remote: Found 127.13.28.129:8080 listening port
    remote: Result: success
    remote: Activation status: success
    remote: Deployment completed with status: success
    To
    ssh://52e118694382ec02180007e7@jekyll-testjekyll.rhcloud.com/~/git/jekyll.git/
       50bcec4..df37147  master -> master


List the application:

    $ rhc apps
    jekyll @ http://jekyll-testjekyll.rhcloud.com/ (uuid: 52e118694382ec02180007e7)
    -------------------------------------------------------------------------------
      Domain:     testjekyll
      Created:    6:56 PM
      Gears:      1 (defaults to small)
      Git URL:    ssh://52e118694382ec02180007e7@jekyll-testjekyll.rhcloud.com/~/git/jekyll.git/
      SSH:        52e118694382ec02180007e7@jekyll-testjekyll.rhcloud.com
      Deployment: auto (on git push)
    
      developercorey-jekyll-1.3.1 (Jekyll static web page and blog server)
      --------------------------------------------------------------------
        From:  https://raw.github.com/openshift-cartridges/openshift-jekyll-cartridge/master/metadata/manifest.yml
        Gears: 1 small


Browse the [Jekyll application on OpenShift]


Jekyll w/o OpenShift
--------------------

    $ gem install jekyll
    $ jekyll new testwebsite 
    $ cd testwebsite
    $ jekyll serve
    # Now browse to http://localhost:4000
 

Some commands
-------------

An arbitrary list of handy commands:

    $ rhc help commands
    $ rhc setup # Helps to run
    $ rhc account
    $ rhc cartridge-list
    $ rhc server
    $ rhc apps
    $ rhc app show
    $ rhc app show manifestyml
    $ rhc show-app --gears -a manifestyml 
    $ rhc ssh  -a manifestyml
    # Create an application from the command-line
    $ rhc app create jekyll https://raw.github.com/openshift-cartridges/openshift-jekyll-cartridge/master/metadata/manifest.yml


[Hello Universe]:http://kashyapc.fedorapeople.org/openshift-tinker/2014-01-23-hellouniverse.markdown
[Jekyll application on OpenShift]:http://jekyll-testjekyll.rhcloud.com/

