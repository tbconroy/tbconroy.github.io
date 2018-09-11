---
layout: post
title:  "Creating Data with FactoryBot for Rails Cypress Tests"
date:   2018-04-07 19:06:55 -0400
categories:
  - cypress
  - testing
  - rails
  - ruby
  - javascript
  - factory bot
---
I have been working with the wonderful and relatively new [Cypress Test Runner](https://www.cypress.io/) to do integration testing for a Rails app with an extensive React/Redux frontend. It has been great. You can read about why it makes sense to use Cypress for testing Rails in this [blog post](https://www.simplethread.com/unblock-rails-ui-testing-cypress/).

Cypress doesn't have any knowledge of the internals of whatever you are testing. So setting up and tearing down data for testing a Rails app can be a little tricky especially if you are used to having [FactoryBot](https://github.com/thoughtbot/factory_bot) (formerly FactoryGirl) to do so. One suggested way to do this is by creating a endpoint in your app to which you can post data to from within your Cypress tests to and create whatever data is needed.

Here is quick solution I came up with that does exactly this.

## Setting it up

First setup the route for the `/factories` endpoint in `config/routes.rb`:
```ruby
if Rails.env.test?
  resources :factories, only: :create
end
```
Note that this route should only be available in your testing environment. It should NEVER be available in production, or probably any other environment, for obvious reasons.

Next create the `FactoriesController` that allows one to post data to the `\factories` endpoint above.

```ruby
require 'factory_bot_rails'

class FactoriesController < ApplicationController
  respond_to :json
  rescue_from Exception, with: :show_errors

  def create
    render json: FactoryBot.create(factory, *traits, attributes).to_json, status: 201
  end

  private

  def traits
    if params[:traits].present?
      params[:traits].map { |_key, trait| trait.to_sym }
    end
  end

  def factory
    params[:factory].to_sym
  end

  def attributes
    params.except(:factory, :traits, :controller, :action, :number).symbolize_keys
  end

  def show_errors(exception)
    error = {
      error: "#{exception.class}: #{exception.to_s}",
      backtrace: exception.backtrace.join("\n")
    }

    render json: error, status: 400
  end
end
```

This controller expects a json request body with the following structure:
```json
{
  // the name of the factory
  "factory":"user",
  // optional factory traits
  "traits":["admin","has_posts"],
  // any attributes for the factory you want to set, possible examples listed below
  "name":"tbconroy",
  "email":"tbconroy@example.com"
}
```
When posted to the `/factories` endpoint it would the equivalent of doing this with FactoryBot:
```ruby
FactoryBot.create(:user, :admin, :has_posts, name: 'tbconroy', email: 'tbconroy@example.com')
```
Of course you would have to have a `user` factory defined. For more information on FactoryBot on how to set it up in Rails I suggest reading the documentation.

When you successfully create a model instance using this endpoint it will return a serialized version of that instance's attributes so that you can see exactly what is created in Cypress.

If something goes wrong the response will include the error raised with a Rails stack trace to help debug any issues.

## Usage with Cypress

I created a helper for use in Cypress, called a [command](https://docs.cypress.io/api/cypress-api/custom-commands.html), that posts to the endpoint defined above.
```javascript
Cypress.Commands.add('factoryBotCreate', (args) => {
  cy.request({
    url: '/factories',
    method: 'post',
    form: true,
    failOnStatusCode: true,
    body: args
  })
})
```
Because `failOnStatusCode` option is set to `true` for `cy.request` if the data fails to create on the Rails side the test will fail and output the factory error and stack trace in the test runner to help with debugging.

This command can then be used in your Cypress test to create the same `user` from before:
```javascript
before(() =>{
  cy.factoryBotCreate({
    factory: 'user',
    traits: ['admin','has_posts'],
    name: 'tbconroy',
    email: 'tbconroy@example.com'
  })
})
```

## Going further
Cleaning your database after each test is important for testing consistency. Another action could be defined in the `FactoriesController` to handle this. One solution would be to use the [DatabaseCleaner gem](https://github.com/DatabaseCleaner/database_cleaner) and the truncate cleaning strategy to wipe the entire database. You would obviously have to be sure not to inadvertently expose this endpoint in production.
