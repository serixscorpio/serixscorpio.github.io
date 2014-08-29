---
layout: post
title: Figaro to Rails
---
Figaro can be used within the context of Rails.  For the longest time I have taken it for granted that it 'just works' without digging deeper into how various gems are made to work seamlessly with Rails.  This time, we will start exploring.  Figaro, an environment variable management tool, knows about the values defined in `config/application.yml` within a Rails application.  But how does Figaro know about the presence of this file in Rails?  What if I want to put my custom environment variables into a separate file instead of using `config/application.yml`?  How can that be accomplished?

To begin with, we look at `spec/rails_spec.rb` in Figaro.

```ruby
require "spec_helper"

describe Figaro::Rails do
  before do
    run_simple(<<-CMD)
      rails new example \
        --skip-gemfile \
        --skip-bundle \
        --skip-keeps \
        --skip-sprockets \
        --skip-javascript \
        --skip-test-unit \
        --no-rc \
        --quiet
      CMD
    cd("example")
  end

  describe "initialization" do
    before do
      write_file("config/application.yml", "foo: bar")
    end

    it "loads application.yml" do
      run_simple("rails runner 'puts Figaro.env.foo'")

      assert_partial_output("bar", all_stdout)
    end
    ...
  end
end
```

Notice that before each test, a new, barebone rails application called `example` is created.  Also, a `config/application.yml` is created with `foo: bar` in it.  The test executes `puts Figaro.env.foo` in rails and expects the output to contain `bar`.  `bar` would only appear on standard output (stdout) if the `application.yml` is somehow read and turns `foo` into an environment variable with value `bar`.

Let us take a couple of steps back and consider why Rails understand `Figaro.env.foo`. At the top of `spec/rails_spec.rb`, we require `spec_helper`:

```ruby
require "bundler"
Bundler.setup

if ENV["COVERAGE"]
  require "codeclimate-test-reporter"
  CodeClimate::TestReporter.start
end

require "figaro"

Bundler.require(:test)

Dir[File.expand_path("../support/*.rb", __FILE__)].each { |f| require f }

```

Notice the `require "figaro"`, which leads us to `lib/figaro.rb`:

```ruby
require "figaro/error"
require "figaro/env"
require "figaro/application"

module Figaro
  ...
end

require "figaro/rails"
```

This is the top level file that includes the core of the Figaro gem itself, but at the moment, we are more interested in the last line `require "figaro/rails"`.  Inside `figaro/rails`:

```ruby
begin
  require "rails"
rescue LoadError
else
  require "figaro/rails/application"
  require "figaro/rails/railtie"

  Figaro.adapter = Figaro::Rails::Application
end
```

We see `require "rails"`. This require will run successfully because the Gemfile of the Figaro codebase includes `rails`.  This is also why we can run `rails new <application name>` in `spec/rails_spec.rb`.  Then we move onto the `else` block.  Here, `figaro/rails/application` and `figaro/rails/railtie` are the primary integration point between Figaro and Rails.

`figaro/rails/application.rb` implements a subclass `Figaro::Rails::Application` which inherits from base class `Figaro::Application`.  The subclass provides implementation for `default_path` and `default_environment`.  In both methods, the implementation relies on the presence of `::Rails`.  The default path is derived as `<rails root>/config/application.yml`; the default environment is the current environment of the rails app.

`figaro/rails/railtie.rb` implements a subclass `Figaro::Rails::Railtie` which inherits from base class `::Rails::Railtie`.  Railtie allows non-rails code to [interact with the Rails frameworks during or after boot](http://edgeapi.rubyonrails.org/classes/Rails/Railtie.html). In this case, the `config` method returns the instance of `Railtie::Configuration` that belongs to the application being initialized.  Specifically `config` has a method called `before_configuration`.  This method's block is the first configurable block to run, and is called before any initializers are run.  So essentially `Figaro.load` is called as early as possible during rails startup.  It runs before the application configuration block inside `application.rb` is run.

`Figaro.load` then kicks off a chain of calls in `lib/figaro.rb`.  Eventually, we end up with an instance of `Figaro::Rails::Application`. The method `load` is then invoked to load the `<rails root>/config/application.yml` file.
