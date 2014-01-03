#Playhouse angular app - part 1 - setting up

In this 3 part series we will build a small app using playhouse, and
playhouse sinatra to serve an api, and a separate yeoman based angular
app to serve the front end.

The app is a basic who owes who what calculator to use after going away with a
bunch of friends.  The problem it solves is that often everyone pays different
amounts at different times (e.g. I'll get the gas this time), then when it
comes to figuring out who owes who what at the end of the trip someone
has a spreadsheet nightmare to deal with, and needs to email everyone
and pester them.

In this first part (of what I hope will be a 3 part series) we will
cover the basic setup of the playhouse api, and the yeoman front end.

##Setting up the playhouse api.

Requirements.

1. a version of ruby installed
2. bundler

First we need to create a Gemfile and populate it with the base set of
gems we need to get up and running.

###Gemfile
```
source "http://rubygems.org"

gem 'rake'

gem 'sqlite3'
gem 'activerecord', '~>4.0.1'

gem 'playhouse', git: 'git://github.com/enspiral/playhouse.git'
gem 'playhouse-sinatra', git: 'git://github.com/enspiral/playhouse-sinatra.git'


gem 'rack-cors', :require => 'rack/cors'
```

We can now run bundle install.
