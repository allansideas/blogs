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

*Gemfile*
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

Next we need to create the basic directory structure and config files for our app.

The basic directory structure looks like this:
```
-config
--database.yml
-db
--migrate
--schema.rb
--seeds.rb
-lib
--wpww
---composers
---contexts
---entities
---roles
---wpww_play.rb
--support
--tasks
--active_record_tasks.rb
--wpww_core.rb
Gemfile
config.ru
wpww_sinatra.rb
```

```
touch wpww_sinatra.rb config.ru && mkdir config db lib && cd db && mkdir migrate && cd ../lib && touch wpww_core.rb && mkdir wpww support tasks && cd wpww && touch wpww_play.rb && mkdir composers contexts entities roles && cd .. && cd ..
```

The guts of the app live under the app name in lib. and there are four main
config files at this stage: wpww_sinatra.rb, wpww_core.rb, wpww_play.rb,
and database.yml.  There is also a tasks file for activerecord
migrations and seeds and a config.ru file for rackup.

wpww_sinatra, and wpww_core set up the external dependencies of the app,
and wpww_play sets up the internals (the bits we create).

*wpww_sinatra.rb*
```
$LOAD_PATH << File.expand_path(File.join(File.dirname(__FILE__), 'lib'))
require 'yaml'
require 'sinatra/base'
require 'playhouse/sinatra'
require 'wpww/wpww_play'

class WpwwWeb < Sinatra::Base
  register Playhouse::Sinatra
  set :root,  File.expand_path(File.join(File.dirname(__FILE__)))

  #we will get to this later.
  routes = YAML.load_file('config/routes.yml')

  add_play Wpww::WpwwPlay, routes

  run! if app_file == $0
end
```

*wpww_core.rb*
```
require 'active_record'
require 'sqlite3'
require 'active_record/connection_adapters/sqlite3_adapter'
require 'wpww/wpww_play'

module Wpww
  #Here we can set app wide config.
  ROOT_PATH = File.expand_path(File.join(File.dirname(__FILE__), '..'))
end
```

*wpww_play.rb*
```
require 'playhouse/support/files'
require 'playhouse/play'
require_all File.dirname(__FILE__), 'contexts/**/*.rb'


module Wpww
  class WpwwPlay < Playhouse::Play
    context TODO
    contexts_for TODO

    def self.name
      'wpww'
    end
  end
end
```

*config.ru*
```
$LOAD_PATH << '.'
#config.ru
require 'rubygems'
require 'json'
require 'sinatra'
require 'rack'
require 'rack/cors'

#This is currently completely insecure.
use Rack::Cors do |config|
  config.allow do |allow|
    allow.origins '*'
    allow.resource '*',
        :methods => [:get, :post, :put, :patch, :delete, :options],
        :headers => :any,
        :max_age => 0
  end
end

require 'wpww_sinatra'
set :root, Pathname(__FILE__).dirname
set :environment, ENV['RACK_ENV']
set :run, false
run WpwwWeb
```

*active_record_tasks.rb*
```
include ActiveRecord::Tasks

db_dir = File.join(@root, 'db')
config_dir = File.join(@root, 'config')

ENV['ENV'] = 'test' if ENV['TRAVIS']
DatabaseTasks.env = ENV['ENV'] || 'development'
DatabaseTasks.root = @root
DatabaseTasks.db_dir = db_dir
DatabaseTasks.database_configuration = YAML.load(File.read(File.join(config_dir, 'database.yml')))
DatabaseTasks.migrations_paths = File.join(db_dir, 'migrate')

require File.join(db_dir, 'seeds')
DatabaseTasks.seed_loader = Wpww::SeedLoader

task :environment do
  ActiveRecord::Base.configurations = DatabaseTasks.database_configuration
  ActiveRecord::Base.establish_connection DatabaseTasks.env
end

load 'active_record/railties/databases.rake'
```

*database.yml*
```
development:
  database: db/development.sqlite
  adapter: sqlite3

test:
  database: db/test.sqlite
  adapter: sqlite3
```

Now if we run rackup from our root dir, we will see that the app can't
load the context TODO.  Lets create a test context to have a hurray
moment after all our hard work setting up.

*lib/wpww/contexts/test.rb*
```
require 'playhouse/context'

module Wpww
  class Test < Playhouse::Context
    def perform
      "Hurray!"
    end
  end
end
```

Now when we run rackup we should see that the server is listening on
localhost:9292

If we visit http://localhost:9292/wpww/test in our browser then we should see Hurray!.  Hurray! we have a functioning playhouse sinatra api.

Sidenote: Note that if you try changing the Hurrah! text in test.rb and refresh
the page, it will still say hurrah. rackup doesn't automatically reload files
when they change, to remedy with this add ```gem shotgun``` to the Gemfile, and
use ```shotgun``` instead of ```rackup``` shotgun uses localhost:9393 by
default so visit localhost:9393/wpww/test, then try changing the text in
test.rb and refreshing the page.

##Setting up the yeoman angular app.

Requirements:

1. node
2. yeoman (yo)
3. generator-angular
4. Ruby
4. Compass

I like to use a different rbenv/gemset because deployment will differ
from the api.

After you have installed the above run ```yo angular --minsafe
--coffee```.  When asked include twitter bootstrap, and then select yes
for scss version.  I chose to only install ng-sanitize from the list of
aditional modules, and will be using ui-router.

Then run ```sudo npm install``` and  ```bower install```

And run ```grunt server``` to start the client app server

You should see a blank page.  Now that it's all working, lets add
angular-ui-router.

```
bower install angular-ui-router -save
```
The -save flag saves it to your package.json file.

Then in index.html add

```
<script src="bower_components/angular-ui-router/release/angular-ui-router.js"></script>
```
below
```
<!-- build:js scripts/modules.js -->
<script src="bower_components/angular-sanitize/angular-sanitize.js"></script>
```
and a ui-view just inside the body tag
```
<div ui-view="main"></div>
```

Then we need to alter our main angular app config.
*app/scripts/app.coffee*
```
angular.module('wpwwClientApp', [
  'ngSanitize'
  'ui.router'
])
  .config ['$urlRouterProvider', ($urlRouterProvider) ->
    $urlRouterProvider.otherwise('/')
  ]
```

And lets add a state to test it out.  ```mkdir app/scripts/states```

*app/views/states/public.coffee*
```
angular.module('states.public', [])
.config(['$stateProvider', '$urlRouterProvider', ($stateProvider, $urlRouterProvider)->
  $stateProvider.state('test', 
    url: '/test'
    views:
      'main':
        template: 'Hurrah!'
        controller: (['$scope', '$state', ($scope, $state)->
          console.log $scope, $state
        ]) #end controller
  )
])
```

Then add the dependency to app.coffee
*app.coffee*
```
angular.module('wpwwClientApp', [
  'ngSanitize'
  'ui.router'
  'states.public' <--- here
])
---
```

And add the file to index.html so it is compiled. 

*index.html*
```
---
<!-- build:js({.tmp,app}) scripts/scripts.js -->
<script src="scripts/app.js"></script>
<script src="scripts/states/public.js"></script>
<!-- endbuild -->
---
```

Now when we run grunt server, and visit http://127.0.0.1:9000/#/test
then we see Hurrah!.
