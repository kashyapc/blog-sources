---
layout: post
title: "Hosting an Octopress instance on OpenShift"
date: 2014-01-30 14:41:48 +0530
comments: true
categories: 
- OpenShift
- Octopress
- jekyll
---

[Reference] 

The below description involves full STDOUT of all commands.

Create a Ruby OpenShift application
-----------------------------------

Install the OpenShift client RPM:

    $ sudo yum install rubygem-rhc -y  


Create a Ruby application:

    $ rhc app-create octotest ruby-1.9
    Application Options
    -------------------
      Domain:     testjekyll
      Cartridges: ruby-1.9
      Gear Size:  default
      Scaling:    no
    
    Creating application 'octotest' ... done
    
    
    Waiting for your DNS name to be available ... done
    
    Cloning into 'octotest'...
    The authenticity of host 'octotest-testjekyll.rhcloud.com (50.19.143.76)' can't be established.
    RSA key fingerprint is cf:ee:77:cb:0e:fc:02:d7:72:7e:ae:80:c0:90:88:a7.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added 'octotest-testjekyll.rhcloud.com,50.19.143.76' (RSA) to the list of known hosts.
    
    Your application 'octotest' is now available.
    
      URL:        http://octotest-testjekyll.rhcloud.com/
      SSH to:     52e3b7494382ece770000032@octotest-testjekyll.rhcloud.com
      Git remote: ssh://52e3b7494382ece770000032@octotest-testjekyll.rhcloud.com/~/git/octotest.git/
      Cloned to:  /var/tmp/test/octotest
    
    Run 'rhc show-app octotest' for more details about your app.


Enumerate the details of the app:

    $ rhc show-app octotest
    octotest @ http://octotest-testjekyll.rhcloud.com/ (uuid: 52e3b7494382ece770000032)
    -----------------------------------------------------------------------------------
      Domain:     testjekyll
      Created:    6:38 PM
      Gears:      1 (defaults to small)
      Git URL:    ssh://52e3b7494382ece770000032@octotest-testjekyll.rhcloud.com/~/git/octotest.git/
      SSH:        52e3b7494382ece770000032@octotest-testjekyll.rhcloud.com
      Deployment: auto (on git push)
    
      ruby-1.9 (Ruby 1.9)
      -------------------
        Gears: 1 small


Configure Octopress
-------------------

Clone the Octopress repsitory, and install 'bundler':

    $ git clone git://github.com/imathis/octopress.git octopress 

    $ gem install bundler
    Successfully installed bundler-1.5.2
    Parsing documentation for bundler-1.5.2
    Done installing documentation for bundler after 2 seconds
    1 gem installed
    

Edit the Gemfile as below (update the rake to local version & add a few
Gems):

    $ cd octopress

    $ rake --version
    rake, version 10.0.4

    $ git diff Gemfile
    diff --git a/Gemfile b/Gemfile
    index cd8ce57..16dc5ac 100644
    --- a/Gemfile
    +++ b/Gemfile
    @@ -1,7 +1,7 @@
     source "https://rubygems.org"
     
     group :development do
    -  gem 'rake', '~> 0.9'
    +  gem 'rake', '~> 10.0.4'
       gem 'jekyll', '~> 0.12'
       gem 'rdiscount', '~> 2.0.7'
       gem 'pygments.rb', '~> 0.3.4'
    @@ -15,6 +15,9 @@ group :development do
       gem 'stringex', '~> 1.4.0'
       gem 'liquid', '~> 2.3.0'
       gem 'directory_watcher', '1.4.1'
    +  gem 'json'
    +  gem 'execjs'
    +  gem 'therubyracer'
     end
     
     gem 'sinatra', '~> 1.4.2'
    $ 


Install and setup Octopress 
---------------------------

Install dependencies through `bundle`:

    $ bundle install
    Resolving dependencies...
    Using rake (10.0.4)
    Using RedCloth (4.2.9)
    Using chunky_png (1.2.5)
    Using fast-stemmer (1.0.1)
    Using classifier (1.3.3)
    Using fssm (0.2.9)
    Using sass (3.2.9)
    Using compass (0.12.2)
    Using directory_watcher (1.4.1)
    Using execjs (2.0.2)
    Using haml (3.1.7)
    Using kramdown (0.13.8)
    Using liquid (2.3.0)
    Using syntax (1.0.0)
    Using maruku (0.6.1)
    Using posix-spawn (0.3.6)
    Using yajl-ruby (1.1.0)
    Using pygments.rb (0.3.4)
    Using jekyll (0.12.0)
    Using json (1.8.1)
    Using libv8 (3.16.14.3)
    Using rack (1.5.2)
    Using rack-protection (1.5.0)
    Using rb-fsevent (0.9.1)
    Using rdiscount (2.0.7.3)
    Using ref (1.0.5)
    Using rubypants (0.2.0)
    Using sass-globbing (1.0.0)
    Using tilt (1.3.7)
    Using sinatra (1.4.2)
    Using stringex (1.4.0)
    Using therubyracer (0.12.0)
    Using bundler (1.5.2)
    Your bundle is complete!
    Use `bundle show [gemname]` to see where a bundled gem is installed.


Install it and traverse one level above the Octopress directory:

    $ rake install
    ## Copying classic theme into ./source and ./sass
    mkdir -p source
    cp -r .themes/classic/source/. source
    mkdir -p sass
    cp -r .themes/classic/sass/. sass
    mkdir -p source/_posts
    mkdir -p public

    $ cd ..


Create a different git repository outside the Octopress directory, copy
the `config.ru` and `Gemfile` into `_deployment/ directory`:

    $ mkdir _deployment && cd _deployment
    
    $ cp ../octopress/config.ru .
    
    $ cp ../octopress/Gemfile .
    
    $ bundle install
    Fetching gem metadata from https://rubygems.org/.......
    Fetching additional metadata from https://rubygems.org/..
    Resolving dependencies...
    Using rake (10.0.4)
    Using RedCloth (4.2.9)
    Using chunky_png (1.2.9)
    Using fast-stemmer (1.0.2)
    Using classifier (1.3.4)
    Using fssm (0.2.10)
    Using sass (3.2.14)
    Using compass (0.12.2)
    Using directory_watcher (1.4.1)
    Using execjs (2.0.2)
    Using haml (3.1.8)
    Using kramdown (0.14.2)
    Using liquid (2.3.0)
    Using maruku (0.7.1)
    Using posix-spawn (0.3.8)
    Using yajl-ruby (1.1.0)
    Using pygments.rb (0.3.7)
    Using jekyll (0.12.1)
    Using json (1.8.1)
    Using libv8 (3.16.14.3)
    Using rack (1.5.2)
    Using rack-protection (1.5.2)
    Using rb-fsevent (0.9.4)
    Using rdiscount (2.0.7.3)
    Using ref (1.0.5)
    Using rubypants (0.2.0)
    Using sass-globbing (1.0.0)
    Using tilt (1.4.1)
    Using sinatra (1.4.4)
    Using stringex (1.4.0)
    Using therubyracer (0.12.0)
    Using bundler (1.5.2)
    Your bundle is complete!
    Use `bundle show [gemname]` to see where a bundled gem is installed.

    $ mkdir public/
    
    $ git init .
    Initialized empty Git repository in /var/tmp/test/_deployment/.git/
    
    $ git remote add openshift ssh://52e3b7494382ece770000032@octotest-testjekyll.rhcloud.com/~/git/octotest.git/

    $ git remote --verbose
    openshift   ssh://52e3b7494382ece770000032@octotest-testjekyll.rhcloud.com/~/git/octotest.git/ (fetch)
    openshift   ssh://52e3b7494382ece770000032@octotest-testjekyll.rhcloud.com/~/git/octotest.git/ (push)
    
    $ git add .
    
    $ git commit -m "Initial deployment"
    [master (root-commit) 3e1b644] Initial deployment
     3 files changed, 124 insertions(+)
     create mode 100644 Gemfile
     create mode 100644 Gemfile.lock
     create mode 100644 config.ru
    
    $ cd ..
    
    $ mv _deployment/ octopress/
 

Publishing to OpenShift
-----------------------

Create a post and generate the static site, preview it locally:

    $ cd octopress/

    $ git add _deployment/

    $ rake new_post["Hello Universe"]
    mkdir -p source/_posts
    Creating new post: source/_posts/2014-01-25-hello-universe.markdown
    kashyap@octopress$ rake generate
    ## Generating Site with Jekyll
    directory source/stylesheets/ 
       create source/stylesheets/screen.css 
    Configuration from /var/tmp/test/octopress/_config.yml
    Building site: source -> public
    Successfully generated site: source -> public

    $ rake generate
    ## Generating Site with Jekyll
    directory source/stylesheets/ 
       create source/stylesheets/screen.css 
    Configuration from /var/tmp/test/octopress/_config.yml
    Building site: source -> public
    Successfully generated site: source -> public


Preview it locally and browse to http://localhost:4000:

    $ rake preview
    Starting to watch source with Jekyll and Compass. Starting Rack on port
    4000
    Configuration from
    /export/openshift-tinker/test-octopress/octopress/_config.yml
    Auto-regenerating enabled: source -> public
    [2014-01-30 15:16:15] regeneration: 96 files changed
    >>> Change detected at 15:16:15 to: screen.scss
    [2014-01-30 15:16:16] INFO  WEBrick 1.3.1
    [2014-01-30 15:16:16] INFO  ruby 2.0.0 (2013-11-22) [x86_64-linux]
    [2014-01-30 15:16:16] INFO  WEBrick::HTTPServer#start: pid=818 port=4000
    identical public/stylesheets/screen.css 
    
    Dear developers making use of FSSM in your projects,
    FSSM is essentially dead at this point. Further development will
    be taking place in the new shared guard/listen project. Please
    let us know if you need help transitioning! ^_^b
    - Travis Tilley
    
    >>> Compass is polling for changes. Press Ctrl-C to Stop.


Copy the static website to `_deployment/ directory`:

    $ cp -R public/* _deployment/public/

    $ cd _deployment/

    $ git add .

    $ git commit -m "New blog post"


Push it to the remote OpenShift repository:

    $ git push openshift +master
    Counting objects: 87, done.
    Delta compression using up to 4 threads.
    Compressing objects: 100% (79/79), done.
    Writing objects: 100% (87/87), 188.61 KiB | 0 bytes/s, done.
    Total 87 (delta 3), reused 0 (delta 0)
    remote: Stopping Ruby cartridge
    remote: [Sat Jan 25 09:47:42 2014] [warn] PassEnv variable SHELL was undefined
    remote: [Sat Jan 25 09:47:42 2014] [warn] PassEnv variable USER was undefined
    remote: [Sat Jan 25 09:47:42 2014] [warn] PassEnv variable LOGNAME was undefined
    remote: Building git ref 'master', commit 64e51f1
    remote: Building Ruby cartridge
    remote: Bundling RubyGems based on Gemfile/Gemfile.lock to repo/vendor/bundle with 'bundle install --deployment'
    remote: Fetching gem metadata from https://rubygems.org/.......
    remote: Fetching gem metadata from https://rubygems.org/..
    remote: Installing rake (10.0.4) 
    remote: Installing RedCloth (4.2.9) with native extensions 
    remote: .........
    remote: ..
    remote: 
    remote: Installing chunky_png (1.2.9) 
    remote: Installing fast-stemmer (1.0.2) with native extensions 
    remote: .......
    remote: ..
    remote: 
    remote: Installing classifier (1.3.4) 
    remote: Installing fssm (0.2.10) 
    remote: Installing sass (3.2.14) 
    remote: Installing compass (0.12.2) 
    remote: Installing directory_watcher (1.4.1) 
    remote: Installing execjs (2.0.2) 
    remote: Installing haml (3.1.8) 
    remote: Installing kramdown (0.14.2) 
    remote: Installing liquid (2.3.0) 
    remote: Installing maruku (0.7.1) 
    remote: Installing posix-spawn (0.3.8) with native extensions 
    remote: ........
    remote: ..
    remote: 
    remote: Installing yajl-ruby (1.1.0) with native extensions 
    remote: .........................
    remote: ..
    remote: 
    remote: Installing pygments.rb (0.3.7) 
    remote: Installing jekyll (0.12.1) 
    remote: Installing json (1.8.1) with native extensions 
    remote: ....
    remote: ..
    remote: 
    remote: ...
    remote: ..
    remote: 
    remote: Installing libv8 (3.16.14.3) 
    remote: Installing rack (1.5.2) 
    remote: Installing rack-protection (1.5.2) 
    remote: Installing rb-fsevent (0.9.4) 
    remote: Installing rdiscount (2.0.7.3) with native extensions ..
    remote: .........................................................................................
    remote: ..
    remote: 
    remote: Installing ref (1.0.5) 
    remote: Installing rubypants (0.2.0) 
    remote: Installing sass-globbing (1.0.0) 
    remote: Installing tilt (1.4.1) 
    remote: Installing sinatra (1.4.4) 
    remote: Installing stringex (1.4.0) 
    remote: Installing therubyracer (0.12.0) with native extensions 
    remote: .....................................................................................................................................
    remote: ..
    remote: 
    remote: Using bundler (1.1.4) 
    remote: Your bundle is complete! It was installed into ./vendor/bundle
    remote: Preparing build for deployment
    remote: Deployment id is 1795eeb0
    remote: Activating deployment
    remote: Starting Ruby cartridge
    remote: Result: success
    remote: Activation status: success
    remote: Deployment completed with status: success
    To ssh://52e3b7494382ece770000032@octotest-testjekyll.rhcloud.com/~/git/octotest.git/
     + 26e8a20...64e51f1 master -> master (forced update)


Traverse to [Octopress application on OpenShift].


Notes on making further changes
-------------------------------

Make some changes:

    kashyap@octopress$ git diff _config.yml
    diff --git a/_config.yml b/_config.yml
    index 74d35ba..3dd4dfd 100644
    --- a/_config.yml
    +++ b/_config.yml
    @@ -2,10 +2,10 @@
     #      Main Configs       #
     # ----------------------- #

    -url: http://yoursite.com
    -title: My Octopress Blog
    -subtitle: A blogging framework for hackers.
    -author: Your Name
    +url: http://localhost
    +title: Writings Open Source software
    +subtitle: Linux, Virtualization, and more.
    +author: Kashyap Chamarthy
     simple_search: http://google.com/search
     description:


Generate the static website, preview it locally:

    kashyap@octopress$ rake generate
    ## Generating Site with Jekyll
    identical source/stylesheets/screen.css
    Configuration from /var/tmp/test/octopress/_config.yml
    Building site: source -> public
    Successfully generated site: source -> public

    kashyap@octopress$ rake preview


Remove the files from  `_deployment/public/*`

    kashyap@octopress$ rm -rf _deployment/public/*


Copy the generated static files to `_deployment directory`:

    kashyap@octopress$ cp -R public/* _deployment/public/


Traverse to `_deployment directory`; add, commit, push:

    kashyap@octopress$ cd _deployment/

    kashyap@_deployment$ git add .

    kashyap@_deployment$ git commit -m "Minor edit to _config.yml "
    [master 6da8f94] Minor edit to _config.yml
     8 files changed, 55 insertions(+), 55 deletions(-)
    kashyap@_deployment$

    kashyap@_deployment$ git push openshift +master


Reload the [Octopress application on Openshift] to reflect the new
changes:


[Reference]:http://www.shellco.de/deploying-octopress-on-openshift/
[Octopress application on OpenShift]:http://octotest-testjekyll.rhcloud.com/
