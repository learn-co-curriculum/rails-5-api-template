# Rails 5 API Template

## Objectives

* learn how to make a new rails 5 API applciation
* Setting up Active Model Serializers
* Creating a CRUD API

## Introduction

In this lesson we are going to be bootstrapping a Rails 5 API app using the Rails 5 API template. Make sure that you have Rails 5 installed (preferably version 5.1 or higher), if you need to install or upgrade check out the last lesson [here](https://github.com/learn-co-curriculum/intro-to-rails-and-react)

### Rails API Template

So what is Rails 5 API Template? It is a slimmed down version of Rails that doesn't include a template library, asset pipeline, ActionView, and some other excess bloat that is uncessary for creating API endpoints. 

You can find our more information [here](http://edgeguides.rubyonrails.org/api_app.html). Some developers feel that using Rails to build JSON APIs is a bit of an overkill, but Rails argues that being able to prototype and get up an app up and running quickly is most important. With Rails 5 API Template it succeeds at this in spades.

### Rails New

Bootstrapping a Rails 5 API Template application is just as easy as it is to create a traditional Rails app. Before we run the command let's talk about the app we are going to be building for the next few lessons. We are going to prototype a small application call Iron Starter that allows users to create Campaigns to raise money for a project or idea. The app is fairly simple and will include a __Campaign__ model that has many __Comments__. While basic, it gives us a good domain to sink our teeth into. 

Ok so let's get building. To generate a new project we need to run the `--api` flag after the name of the app we are generating.

```
rails new iron-starter-api --api
```

Next lets enter the the app directory and open it in the text editor. If we take a look at the file directory and we should notice that the `/app/assets` folder is missing and inside of our `/app/views` folder is just a basic `mailer.html.erb` file. As our app will only be serving JSON, we have no need of an `application.html.erb` file.

### Creating An Endpoint (Review)

In the Rails + JS section of the course we created endpoints for our JavaScript code to consume. We are going to be doing the same thing here, but we don't have to use the `respond_to` block and return either `.html` or `.json`. For this API we are only going to be returning JSON, so it allows us to simplify code a lot and focus on clean responses. 

Let's first build out a request for getting a list of Campaigns:

#### Step 1: Building the Model 

Since this is Rails, we will generate models the same way that we've always done. Our Campaign model will need attributes for `Title: String, Description: Text, Goal: Integer, Pledged: Integer`.

```
rails g model Campaign title:string description:text goal:integer pledged:integer
```

This should have created the following files: 

```ruby
invoke  active_record
    create    db/migrate/20170830192609_create_campaigns.rb
    create    app/models/campaign.rb
    invoke    test_unit
    create      test/models/campaign_test.rb
    create      test/fixtures/campaigns.yml
```

Note: If we go to the `/app/models/campaign.rb` file there should be a `Campaign` class that inherits from `ApplicationRecord` instead of `ActiveRecord::Base`. Rails 5 now uses `ApplicationRecord` as a wrapper to inherit from `ActiveRecord::Base`. This is defined in the `/app/models/application_record.rb` file. 

Next we should setup __ActiveModelSerializer__ to serialize our data in JSON. 

First we need to add it to the __Gemfile__:

```ruby 
# Gemfile

# ...

# Use ActiveModelSerializers for JSON serialization
gem 'active_model_serializers', '~> 0.10.0'

# ...

```

Next run `bundle install` and the generator to create our serialized file:

```bash
rails g serializer campaign
    create  app/serializers/campaign_serializer.rb
```

We should now have a __CampaignSerializer__ at /app/serializers/campaign/serializer.rb. We do need to update it to include all of the attributes though.

```ruby 
# /app/serializers/campaign/serializer.rb
class CampaignSerializer < ActiveModel::Serializer
  attributes :id, :title, :description, :goal, :pledged, :created_at, :updated_at
end
```

#### Step 2: Creating the endpoint

Now that we have created our __Campaign__ model, we need to create an endpoint to view campaigns, but first lets migrate our db migrations and build a couple of __Campaigns__ in Rails console. 

```ruby
rails db:migrate
rails console 
:001> Campaign.create(title: 'A Tale of Two Programming Languages', description: 'Raising money to write a book about the difference between JavaScript and Ruby', goal: 5000, pledged: 0)
:002> Campaign.create(title: 'Harry Potter Year 8: The Return of Professor Dumbledore', description: 'Dumbledore lives! In this new book Dumbledore comes back to life. Raising money for publishing rights.', goal: 10000, pledged: 5000)
:003> Campaign.create(title: 'Earthworm Jim 4', description: 'Help a struggling team of game developers create a fan service project for those who grew up playing Earthworm Jim', goal: 1000000, pledged: 10)
```

Now that we have some records we should be able to view them with an endpoint to `/campaigns`

To create our endpoint lets go to the `/config/routes.rb` file and change our code to look like this: 

```ruby
# /config/routes.rb
Rails.application.routes.draw do
  resources :campaigns, only: [:index]
end
```

Running `rails routes` in the terminal should now output this: 

```ruby
rails routes
   Prefix Verb URI Pattern          Controller#Action
campaigns GET  /campaigns(.:format) campaigns#index
```

Notice that it looking for a controller called `Campaigns` and action of `:index`. Let's generate that controller now. 

```bash
rails g controller Campaigns
```

That should have generated a `CampaignsController` at `/app/controllers/campaigns`. We now need to add our `:index` action. 

```ruby 
# /app/controllers/campaigns_controller.rb
class CampaignsController < ApplicationController

    def index 
        @campaigns = Campaign.all 
        render json: @campaigns, status: 200
    end
end
```

If we run a curl command, in the terminal, we should get a response of JSON with an array of campaigns. Make sure that the `rails server` is running.

```bash
curl http://localhost:3000/campaigns
[{"id":1,"title":"A Tale of Two Programming Languages","description":"Raising money to write a book about the difference between JavaScript and Ruby","goal":5000,"pledged":0,"created_at":"2017-08-30T19:32:06.816Z","updated_at":"2017-08-30T19:32:06.816Z"},{"id":2,"title":"Harry Potter Year 8: The Return of Professor Dumbledore","description":"Dumbledore lives! In this new book Dumbledore comes back to life. Raising money for publishing rights.","goal":10000,"pledged":5000,"created_at":"2017-08-30T19:34:32.078Z","updated_at":"2017-08-30T19:34:32.078Z"},{"id":3,"title":"Earthworm Jim 4","description":"Help a struggling team of game developers create a fan service project for those who grew up playing Earthworm Jim","goal":1000000,"pledged":10,"created_at":"2017-08-30T19:36:18.064Z","updated_at":"2017-08-30T19:36:18.064Z"}]
```

Or you can go to the browser at [http://localhost:3000/campaigns](http://localhost:3000/campaigns).

We've now created our first endpoint, let's now build an endpoint to create a Campaign. 

### POST /campaigns API Endpoint 

We will need to add a `:create` action to our routes file and our controller 

```ruby 
# /config/routes.rb
Rails.application.routes.draw do
  resources :campaigns, only: [:index, :create]
end

# /app/controllers/campaigns_controller.rb 
class CampaignsController < ApplicationController

    # ... previous code goes here

    def create
        @campaign = Campaign.create!(campaign_params) 
        render json: @campaign, status: 201
    end

    private 

        def campaign_params 
            params.require(:campaign).permit(:title, :description, :goal, :pledged) 
        end
end
```

Let's try out a post request with __curl__ and create a new campaign

```bash
curl -X POST -H 'Content-Type: application/json' -H 'Accept: application/json' http://localhost:3000/campaigns -d '{"campaign":{"title":"FlatOS","description":"Raising money to build a new OS system using Linux","goal":1000, "pledged":0}}' 

> {"id":4,"title":"FlatOS","description":"Raising money to build a new OS system using Linux","goal":1000,"pledged":0,"created_at":"2017-08-30T20:03:36.533Z","updated_at":"2017-08-30T20:03:36.533Z"}
```

This successfully created a new Campaign object and returned the JSON with a 201 message. 

### Refactoring Time

Now that we have a __GET & POST__ request working lets refactor some of our code. One thing you will notice is that we are using `create!`. When we use `create!` it will fail with either a __ActiveRecord::RecordInvalid__ or __ActionController::ParameterMissing__ error, if there is a problem creating a file, so we need to handle that. 

Right now we only have issues with __ActionController::ParameterMissing__, but we should add some validations to our model to make sure all of the fields are required so we can hit the __ActiveRecord::RecordInvalid__ error and protect our applications data. In our __Campaign__ model we should update it to look like this:

```ruby 
# /app/models/campaign.rb
class Campaign < ApplicationRecord
    validates :title, :description, :goal, :pledged, presence: true
end
```

To make sure we are hitting the error let's run our __curl__ command with bad data and verify that __ActiveRecord::RecordInvalid__ error is being hit. 

```bash
curl -X POST -H 'Content-Type: application/json' -H 'Accept: application/json' http://localhost:3000/campaigns -d '{"campaign":{"title":"","description":"","goal":"", "pledged":""}}' 
```

If you check out the __rails server log__ you should see this:

```ruby
Started POST "/campaigns" for 127.0.0.1 at 2017-08-30 16:17:29 -0400
Processing by CampaignsController#create as JSON
  Parameters: {"campaign"=>{"title"=>"", "description"=>"", "goal"=>"", "pledged"=>""}}
   (0.1ms)  begin transaction
   (0.2ms)  rollback transaction
Completed 422 Unprocessable Entity in 14ms (ActiveRecord: 1.0ms)

ActiveRecord::RecordInvalid (Validation failed: Title can't be blank, Description can't be blank, Goal can't be blank, Pledged can't be blank):
```

Success!! or rather unhandled success.

### Extending ActiveSupport::Concern

This is a good time to extend __ActiveSupport::Concern__ to rescue our code from these errors and show them as a JSON response. 

First we need to create the file

```bash 
touch app/controllers/concerns/exception_handler.rb
```

Inside of this file lets add the following code:

```ruby 
# /app/controllers/concerns/exception_handler.rb
module ExceptionHandler 
    extend ActiveSupport::Concern 

    included do  
        # handle unfound record errors
        rescue_from ActiveRecord::RecordNotFound do |entity| 
            render json: { message: entity.message }, status: :not_found
        end 

        # handle invalid record or missing parameter errors
        rescue_from ActiveRecord::RecordInvalid, ActionController::ParameterMissing do |entity| 
            render json: { message: entity.message }, status: :unprocessable_entity
        end
    end 
end
```

Only thing left to do is now add this to our __ApplicationController__

```ruby 
# /app/controllers/application_controller.rb
class ApplicationController < ActionController::API
    include ExceptionHandler
end
```

Now if we run our __curl__ command we should get a JSON object returned with an error message:

```bash
curl -X POST -H 'Content-Type: application/json' -H 'Accept: application/json' http://localhost:3000/campaigns -d '{"campaign":{"title":"","description":"","goal":"", "pledged":""}}' 
{"message":"Validation failed: Title can't be blank, Description can't be blank, Goal can't be blank, Pledged can't be blank"}[16:22:21] test-app
```

Yay!! Now it is truly a successful failure.

### Finishing up the routes and Controllers

To add the following CRUD routes lets update our controller and routes file. Our final code should look like. 

```ruby 
# /config/routes.rb
Rails.application.routes.draw do 
    resources :campaigns, only: [:index, :show, :create, :update, :destroy]
end


# /app/controllers/campaigns_controller.rb
class CampaignsController < ApplicationController
    before_action :set_campaign, only: [:show, :update, :destroy] 

    # GET /api/campaigns 
    def index 
        @campaigns = Campaign.all 
        render json: @campaigns, status: 200 
    end

    # GET /api/campaigns/:id
    def show 
        render json: @campaign, status: 200
    end

    # POST /api/campaigns
    def create 
        @campaign = Campaign.create!(campaign_params) 
        render json: @campaign, status: 201
    end

    # PUT /api/campaigns/:id
    def update 
        @campaign.update(campaign_params) 
        render json: @campaign, status: 200
    end

    # DELETE /api/campaigns/:id
    def destroy 
        @campaign.destroy
        head :no_content 
    end

    private 

        def campaign_params
            params.require(:campaign).permit(:title, :description, :goal, :pledged) 
        end

        def set_campaign  
            @campaign = Campaign.find(params[:id]) 
        end
end
```

### Namespacing an API

Since we are building out an api it is normally good practice to namespace our routes inside of an api route. To do this we can just add a small one liner to our routes file:

```ruby 
# /config/routes.rb
Rails.application.routes.draw do 
  namespace :api do
    resources :campaigns, only: [:index, :show, :create, :update, :destroy]
  end
end
``` 

Running our rails routes command should now return a list of routes nested in api:

```ruby 
rails routes
       Prefix Verb   URI Pattern                  Controller#Action
api_campaigns GET    /api/campaigns(.:format)     api/campaigns#index
              POST   /api/campaigns(.:format)     api/campaigns#create
 api_campaign GET    /api/campaigns/:id(.:format) api/campaigns#show
              PATCH  /api/campaigns/:id(.:format) api/campaigns#update
              PUT    /api/campaigns/:id(.:format) api/campaigns#update
              DELETE /api/campaigns/:id(.:format) api/campaigns#destroy
```

lets try our __curl__ command to GET /api/campaigns just to verify everything works. 

```bash
curl http://localhost:3000/api/campaigns
```

Oops! All kinds of errors. You should be something like: `{"status":404,"error":"Not Found","exception":"#\u003cActionController::RoutingError: uninitialized constant Api....}` The 404 error means it was unable to find the uninitialized constant Api. Since we have a namespace route we need to put our __CampaignsController__ inside of an API folder.

```bash
mkdir app/controllers/api
mv app/controllers/campaigns_controller.rb app/controllers/api
``` 

We also need to update the naming of our __CampaignsController__ class and our __ActiveSupport::Inflector.inflections__ intializer to know about the acronym capitalization about API instead of Api

```ruby 
# /app/controllers/api/campaigns_controller.rb
class API::CampaingsController < ApplicationController

# /config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.acronym 'API'
end
```

Restart the the Rails server (since we changed an initializer) and try the __curl__ command again to see the response. 

```bash
curl http://localhost:3000/api/campaigns
[{"id":1,"title":"A Tale of Two Programming Languages","description":"Raising money to write a book about the difference between JavaScript and Ruby","goal":5000,"pledged":0,"created_at":"2017-08-30T19:32:06.816Z","updated_at":"2017-08-30T19:32:06.816Z"},{"id":2,"title":"Harry Potter Year 8: The Return of Professor Dumbledore","description":"Dumbledore lives! In this new book Dumbledore comes back to life. Raising money for publishing rights.","goal":10000,"pledged":5000,"created_at":"2017-08-30T19:34:32.078Z","updated_at":"2017-08-30T19:34:32.078Z"},{"id":3,"title":"Earthworm Jim 4","description":"Help a struggling team of game developers create a fan service project for those who grew up playing Earthworm Jim","goal":1000000,"pledged":10,"created_at":"2017-08-30T19:36:18.064Z","updated_at":"2017-08-30T19:36:18.064Z"},{"id":4,"title":"FlatOS","description":"Raising money to build a new OS system using Linux","goal":1000,"pledged":0,"created_at":"2017-08-30T20:03:36.533Z","updated_at":"2017-08-30T20:03:36.533Z"}]
```

Everything is working again!!

### Summary

That was a lot. We just built a full CRUD API using Rails 5 template and namespaced our routes inside of an /api route. Give yourself a pat on the back and get ready for the wild world of CORS next. 

<p class='util--hide'>View <a href='https://learn.co/lessons/rails-5-api-template'>Rails 5 API Template</a> on Learn.co and start learning to code for free.</p>
