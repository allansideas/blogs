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
gem 'playhouse-sinatra', git: 'git://github.com/allansideas/playhouse-sinatra.git'


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

#require File.join(db_dir, 'seeds')
#DatabaseTasks.seed_loader = Wpww::SeedLoader

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

## All together now

We have a functioning back end app and a functioning front end app but
they are currently not talking to each other.  We will do a very basic
call from the front end to the back end, then in the next part of the
tutoral make some real calls to real data in the database.

We will use the $http service in the controller of our state to call one
of the out of the box api calls from the wpww backend.

*app/scripts/states/public.coffee*
```
angular.module('states.public', [])
.config(['$stateProvider', '$urlRouterProvider', ($stateProvider, $urlRouterProvider)->
  $stateProvider.state('test', 
    url: '/test'
    views:
      'main':
        template: '<strong>{{test_data}}</strong>'
        controller: (['$scope', '$state', '$http', ($scope, $state, $http)->
          $http({method: 'GET', url: 'http://localhost:9393/wpww/test'}).success (data, status, headers, config)->
            $scope.test_data = data
          .error (data, status, headers, config)->
            #can handle errors here.
        ]) #end controller
  )
])
```
Here we added the $http service to the controller dependencies, and
called our test url.  If the data successfully returned from the server
then it is assigned to the test_data scope variable, and is rendered in
the ```{{test_data}}``` part of the template.  You should now be seeing
Murray! in bold on the page at localhost:9000/#/test this is data
returned from the server Hurray!

In part two we will look at the wpww backend app in more depth and start
to hook it into our front end app in a more sensible manner.


#Part 2

Now we have a skeleton up and running we can start to plan how the
actual app will work.  I imagine a page with with a list of
people and how much they payed which anyone can add to, and a button that any of them can press which
calculates who pays who what.  I don't think anyone should have to sign up but the page should be semi private so we will use a hash as the url.

In this part we are going to slap someting ugly together with ugly html,
css, javascript functions, the whole works, and in the next part we will
look at what's wrong, and a few ways to clean everything up (otherwise
known as refactoring).

For something like this we need a group, with a description to hold the users and provide a
url, and users with a name, email address, and amount payed to provide
the data we need to run our calculation on.  Lets get started and create
the objects in our wpww backend.

Unfortunately as playhouse is in it's infancy we will have to manually
create our database migrations in the following format.
[year][month][day][hour][minute]_name.rb so in the wpww backend root
folder replace the date and time with your date and time ```touch db/migrate/201401051449_create_groups.rb```, then repeat for users.

*db/migrate/201401051449_create_groups.rb*
```
class CreateGroups < ActiveRecord::Migration
  def change
    create_table :groups do |t|
      t.string :description
      t.string :identifier
      t.timestamps
    end
  end
end
```

*db/migrate/201401051454_create_users.rb*
```
class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t|
      t.integer :group_id
      t.string :name
      t.string :email
      t.integer :amount_payed_cents

      t.timestamps
    end
  end
end
```

We then need to create the entities that relate to these database
tables.

touch lib/wpww/entities/group.rb && touch lib/wpww/entities/user.rb

*lib/wpww/entities/group.rb *
```
require 'active_record'

module Wpww
  class Group < ActiveRecord::Base
    has_many :users
  end
end
```

*lib/wpww/entities/user.rb *
```
require 'active_record'

module Wpww
  class User < ActiveRecord::Base
    belongs_to :group
  end
end
```
Lets set up our rakefile so we can run the migration.
```touch Rakefile```

*Rakefile*
```
require 'bundler'
Bundler.require

require 'rubygems'
require 'bundler/setup'
require 'active_record'

@root = File.dirname(__FILE__)
require 'tasks/active_record_tasks'
```
Now lets run the migration ```rake db:migrate```

So we have the entities, and the database set up, now we need to make
them accessible via the api.  Lets create some contexts.

```mkdir lib/wpww/contexts/groups lib/wpww/contexts/users```
```touch lib/wpww/contexts/groups/create.rb
lib/wpww/contexts/groups/show.rb lib/wpww/contexts/groups/destroy.rb
lib/wpww/contexts/users/create.rb lib/wpww/contexts/users/list.rb
lib/wpww/contexts/users/update.rb```

We will start off with the creation and showing of groups.

We need to generate a unique url for each group on creation so let's add
that to a before_create hook in the entity.
*lib/wpww/entities/group.rb*
```
require 'active_record'

module Wpww
  class Group < ActiveRecord::Base
    before_create :generate_identifier
    has_many :users

    def generate_identifier
      self.identifier = SecureRandom.urlsafe_base64 64
      self.identifier.upcase
    end
  end
end
```
*lib/wpww/contexts/groups/create.rb*
```
require 'playhouse/context'

module Wpww
  module Groups
    class Create < Playhouse::Context
      actor :description, optional: true

      def perform
        data = Group.create!(actors)
        data
      end
    end
  end
end
```

And we need to tell playhouse to load all the groups contexts.
*lib/wpww/wpww_play*
```
require 'playhouse/support/files'
require 'playhouse/play'
require_all File.dirname(__FILE__), 'contexts/**/*.rb'


module Wpww
  class WpwwPlay < Playhouse::Play
    context Test
    contexts_for Groups #Here we say look at all the contexts in the module groups.

    def self.name
      'wpww'
    end
  end
end
```

If we now visit localhost:9393 we will see that there is a new api call
available to us.  

We will create the show context then try it out.

*lib/wpww/contexts/groups/show.rb*
```
require 'playhouse/context'
require 'wpww/entities/group'

module Wpww
  module Groups
    class Show < Playhouse::Context
      actor :identifier

      def perform
        data = Group.find_by_identifier(identifier)
        data
      end
    end
  end
end
```

Playhouse automatically generates GET and POST for each call listed in
localhost:9393, there are of course plans to make it automatically
generate nicer routes, and there is also a basic router in place which
we will touch on soon.  For the purposes of testing these two calls out
we will use GET from the browser location bar, but we know this is
wrong.

if we visit ```localhost:9393/wpww/create_groups?description=Horray!``` We should see some json representing the object that we just created.
If we then copy the identifier and visit
```localhost:9393/wpww/show_groups?identifier=paste_identifier_here```
then we should see the same json.

Lets create some nicer routes and then start wiring up the front end.

*config/routes.yml*
```
-
  get:
    route: 'groups/:identifier'
    command: show_groups
    params: '*identifier'
    description: Returns a specific identifier from the provided identifier
-
  post:
    route: 'groups/'
    command: create_groups
    params: '*description'
    description: Create a group
```

Lets try the get route out, visit ```localhost:9393/wpww/groups/paste_identifier_here``` and we should see the same json as before.

The command part of the routes referrs to the commands generated by
playouse which you see available when visiting localhost:9393 the routes themselves are up to you.


Okay, now we have some nicer routes let's do a messy pass at getting it
functioning in the browser, and look at ways to clean it up in part 3.

We will remove our test state then add a state for creating, and a state for showing a group. on a successful save we will redirect the user to the show state.

*app/scripts/states/public.coffee*
```
angular.module('states.public', [])
.config(['$stateProvider', '$urlRouterProvider', ($stateProvider, $urlRouterProvider)->
  $stateProvider.state('new_wpww',
    url: '/wpww/new'
    views:
      'main':
        template: '
          <h1>Create a new wpww</h1>
          <input ng-model="group.description" type="text"></div>
          <button ng-click="createGroup()">Create Group</button>
          '
        controller: (['$scope', '$state', '$http', ($scope, $state, $http)->
          $scope.group = {}

          $scope.createGroup = ()->
            $http.post('http://localhost:9393/wpww/groups', $scope.group).success (data, status, headers, config)->
              $state.go('wpww', identifier: data.identifier)
            .error (data, status, headers, config)->
              #can handle errors here.
        ]) #end controller
  ).state('wpww',
    url: '/wpww/:identifier'
    views:
      'main':
        template: '
          {{group}}
          '
        controller: (['$scope', '$state', '$http', ($scope, $state, $http)->
          $http.get("http://localhost:9393/wpww/groups/#{$state.params.identifier}").success (data, status, headers, config)->
            $scope.group = data
          .error (data, status, headers, config)->
            alert "Can't find group"
            #can handle errors here.
        ]) #end controller
  )


])
```

Now we want to be able to add users to the group so we need to create
the contexts for create, and list (by group) on the backend.  We also
need to create the routes.

*lib/wpww/contexts/users/create.rb*
```
require 'playhouse/context'

module Wpww
  module Users
    class Create < Playhouse::Context
      actor :name
      actor :email, optional: true
      actor :amount_payed_cents

      def perform
        data = User.create!(actors)
        data
      end
    end
  end
end
```

*lib/wpww/contexts/users/list.rb*
```
require 'playhouse/context'
require 'wpww/entities/group'

module Wpww
  module Users
    class List < Playhouse::Context
      actor :group, repository: Group

      def perform
        data = group.users
        data
      end
    end
  end
end
```

Then we add Users to our play

*lib/wpww/wpww_play.rb*
```
------
module Wpww
  class WpwwPlay < Playhouse::Play
    context Test
    contexts_for Groups
    contexts_for Users

    def self.name
      'wpww'
    end
  end
end
------
```

Now open localhost:9393 and check out the new commands and add them to
our routes.

```
------
#user routes
-
  post:
    route: 'groups/:group_id/users'
    command: create_users
    params: '*group_id, *name, *amount_payed_cents, email'
    description: Create user in a group

-
  get:
    route: 'groups/:group_id/users'
    command: list_users
    params: '*group_id'
    description: Returns a list of users for a groulist of users for a group
```

And now we add the front end functionality for adding users to groups.

*app/scripts/states/public.coffee*
```
------
  ).state('wpww',
    url: '/wpww/:identifier'
    views:
      'main':
        template: '
          <h2>{{group.description}}</h2>
          <ul>
            <li ng-repeat="user in group.users">
            {{user.name}} : ${{user.amount_payed_cents/100}}
            </li>
          </ul>
          <form>
            <p>name*</p>
            <input type="text" ng-model="_user.name" />
            <p>email</p>
            <input type="email" ng-model="_user.email" />
            <p>amount payed in cents</p>
            <input type="number" ng-model="_user.amount_payed_cents" />
            <button ng-click="addUser()">Add someone</button>
          </form>
          '
        controller: (['$scope', '$state', '$http', ($scope, $state, $http)->
          $scope.group = {}
          $scope._user = {}

          $http.get("http://localhost:9393/wpww/groups/#{$state.params.identifier}").success (data, status, headers, config)->
            $scope.group = data
            $http.get("http://localhost:9393/wpww/groups/#{$scope.group.id}/users").success (data, status, headers, config)->
              $scope.group.users = data
              $scope._user = {}
          .error (data, status, headers, config)->
            alert "Can't find group"
            
          $scope.addUser = ()->
            $http.post("http://localhost:9393/wpww/groups/#{$scope.group.id}/users", $scope._user).success (data, status, headers, config)->
              $scope.group.users.push data
            .error (data, status, headers, config)->
              #can handle errors here.
        ]) #end controller
  )
])
```

Now that we can add users and amounts lets create an ugly but functional
wpww calculator.  We should aim to have people paying as few different
people as they can.  And the end result is an array of people who owe
other people money, with what they owe.  We are going to do a messy
draft of this in the controller for now and cover cleaning it up in part 3

we will use some lodash utilities so add lodash to the head in your
index.html outside of any build blocks from yeoman

```<script src="//cdnjs.cloudflare.com/ajax/libs/lodash.js/2.4.1/lodash.min.js"></script>```

The controller for the state wpww should now look like this.  It's huge
and ugly, thank goodness there is a part 3 about cleaning things up.

*app/scripts/states/public.coffee*
```
------
controller: (['$scope', '$state', '$http', ($scope, $state, $http)->
  $scope.group = {}
  $scope._user = {}
  $scope.owing_results = []

  $http.get("http://localhost:9393/wpww/groups/#{$state.params.identifier}").success (data, status, headers, config)->
    $scope.group = data
    $http.get("http://localhost:9393/wpww/groups/#{$scope.group.id}/users").success (data, status, headers, config)->
      $scope.group.users = data
      $scope._user = {}
  .error (data, status, headers, config)->
    alert "Can't find group"
    
  $scope.addUser = ()->
    $http.post("http://localhost:9393/wpww/groups/#{$scope.group.id}/users", $scope._user).success (data, status, headers, config)->
      $scope.group.users.push data
    .error (data, status, headers, config)->
      #can handle errors here.

  $scope.totalSpend = ()->
    total = 0
    for user in $scope.group.users
      total = total + user.amount_payed_cents
    total


  $scope.indexOfOwerInOwings = (ower)->
    index = undefined
    for o, i in $scope.owing_results
      if o.ower.id == ower.id
        index = i
    index

  $scope.addOwing = (ower, user, amount)->
    #if the ower is already in the owers array, find the index
    ower_in_owings_index = $scope.indexOfOwerInOwings(ower)
    #if they are then add the user to thier owings array,
    #otherwise create the owing with one user.
    if ower_in_owings_index?
      $scope.owing_results[ower_in_owings_index].owings.push {user: user, amount: amount}
    else
      owing = {}
      owing.ower = ower
      owing.owings = []
      owing.owings.push {user: user, amount: amount}
      $scope.owing_results.push owing

  $scope.calculateWhoPaysWhat = ()->
    total = $scope.totalSpend()
    num_users = $scope.group.users.length

    #total amount spent by all / number of people = the amount
    #payed by each person if they had all split everything
    #evenly along the way
    even_split = total / num_users

    for user in $scope.group.users
      #get the negative(owing) or positive(owed) distance from the even split
      user.from_even = user.amount_payed_cents - even_split
     
    #sort the users 
    users_order_least_owed = _.sortBy $scope.group.users, "from_even", _.values
    users_order_most_owed = users_order_least_owed.reverse()

    #calculate who owes a given user
    getOwers = (user)->
      for ower, i in users_order_least_owed
        #ignore the user we are asking for owers from ignore
        #them if they are owed money, or they have had all their
        #owers figured out (from_even == 0)
        unless ower.id == user.id or ower.from_even >= 0 or user.from_even == 0
          #if the ower owes more than the total remaining owed
          #to the user then owe the rest of what that user is
          #owed and set the remainder they owe after, otherwise
          #owe the full amount of what they owe to the given
          #user.
          if (user.from_even - ower.from_even * -1) < 0
            remainder_after = user.from_even - ower.from_even * -1
            rest_of_owed = (ower.from_even * -1) - remainder_after * -1
            #building the owers array
            $scope.addOwing(ower, user, rest_of_owed)
            #setting the user and ower new from_even vals
            user.from_even -= rest_of_owed
            ower.from_even = remainder_after
          else
            #building the owers array
            $scope.addOwing(ower, user, ower.from_even * -1)
            #setting the user and ower new from_even vals
            user.from_even -= (ower.from_even * -1)
            ower.from_even = 0

    for owed_user, i in users_order_most_owed
      if owed_user.from_even == 0
        users_order_most_owed.splice i, 1
        return
      else
        getOwers(owed_user)

]) #end controller
----------
```

We also modify the template for the state to output our results when we
click a button.

```
<h2>{{group.description}}</h2>
<ul>
  <li ng-repeat="user in group.users">
  {{user.name}} : ${{user.amount_payed_cents}}
  </li>
</ul>
<form>
  <p>name*</p>
  <input type="text" ng-model="_user.name" />
  <p>email</p>
  <input type="email" ng-model="_user.email" />
  <p>amount payed in cents</p>
  <input type="number" ng-model="_user.amount_payed_cents" />
  <button ng-click="addUser()">Add someone</button>
</form>
<button ng-click="calculateWhoPaysWhat()">Calculate who pays what.</button>
<div ng-repeat="ower in owing_results track by $index">
  <strong>{{ower.ower.name}}</strong>
  must pay
  <br />
  <p ng-repeat="o in ower.owings">
    {{o.user.name}}
    {{o.amount}}
  </p>
</div>
```

Hurrah! frankenstein lives.  It isn't very pretty but it works.  In the
next part we look at how we can turn our frankenstein into somthing that
more closely resembles i-robot, or the wonderful hero/heroin who comes
to save the day.
