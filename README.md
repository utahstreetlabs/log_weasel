# Log Weasel

Instrument Rails and Resque with shared transaction IDs so that you trace execution of a unit of work across instances.
This particularly handy if you're using a system like <a href="http://www.splunk.com">Splunk</a> to manage your log
files across many applications and application instances.

## Installation

Add log_weasel to your Gemfile:

<pre>
gem 'log_weasel'
</pre>

Use bundler to install it:

<pre>
bundle install
</pre>

## Rack

Log Weasel provides Rack middleware to create and destroy a transaction for every HTTP request. You can use it
in a any web framework that supports Rack (Rails, Sinatra,...)

### Rails 3

To see Log Weasel transaction IDs in your Rails logs, you need to add the Rack middleware and
either use the BufferedLogger provided or customize the formatting of your logger to include
<code>LogWeasel::Transaction.id</code>.

<pre>
YourApp::Application.configure do
  logger = LogWeasel::BufferedLogger.new "#{Rails.root}/log/#{Rails.env}.log"
  config.logger                   = logger
  config.action_controller.logger = logger
  config.active_record.logger     = logger

  config.middleware.insert_before Rails::Rack::Logger,
                                  LogWeasel::Middleware, :key => 'YOUR_APP'
end
</pre>

<code>:key</code> is an optional parameter that is useful in an environment where a unit of work may span multiple applications.

## Resque

To see Log Weasel transaction IDs in your Resque logs, you need to need to initialize Log Weasel
when you configure Resque, for example in a Rails initializer.

<pre>
LogWeasel::Resque.initialize! :key => 'YOUR_APP'
</pre>

Start your Resque worker with <code>VERBOSE=1</code> and you'll see transaction IDs in your Resque logs.

## Hoptoad

If you are using <a href="http://hoptoadapp.com">Hoptoad</a>, Log Weasel will add the parameter <code>log_weasel_id</code>
to Hoptoad errors so that you can track execution through your application stack that resulted in the error. No additional
configuration required.

## Example

In this example we have a Rails app pushing jobs to Resque and a Resque worker that run with the Rails environment loaded.

### HelloController

<pre>
class HelloController &lt; ApplicationController

  def index
    Resque.enqueue EchoJob, 'hello from HelloController'
    Rails.logger.info("HelloController#index: pushed EchoJob")
  end

end
</pre>

### EchoJob

<pre>
class EchoJob
  @queue = :default_queue

  def self.perform(args)
    Rails.logger.info("EchoJob.perform: #{args.inspect}")
  end
end
</pre>

Start Resque with:

<pre>
QUEUE=default_queue rake resque:work VERBOSE=1
</pre>

Requesting <code>http://localhost:3030/hello/index</code>, our development log shows:

<pre>
[2011-02-14 14:37:42] YOUR_APP-WEB-192587b585fa66b19638 48353 INFO

Started GET "/hello/index" for 127.0.0.1 at 2011-02-14 14:37:42 -0800
[2011-02-14 14:37:42] YOUR_APP-WEB-192587b585fa66b19638 48353 INFO   Processing by HelloController#index as HTML
[2011-02-14 14:37:42] YOUR_APP-WEB-192587b585fa66b19638 48353 INFO HelloController#index: pushed EchoJob
[2011-02-14 14:37:42] YOUR_APP-WEB-192587b585fa66b19638 48353 INFO Rendered hello/index.html.erb within layouts/application (1.8ms)
[2011-02-14 14:37:42] YOUR_APP-WEB-192587b585fa66b19638 48353 INFO Completed 200 OK in 14ms (Views: 6.4ms | ActiveRecord: 0.0ms)
[2011-02-14 14:37:45] YOUR_APP-WEB-192587b585fa66b19638 48461 INFO EchoJob.perform: "hello from HelloController"
</pre>

Fire up a Rails console and push a job directly with:

<pre>
> Resque.enqueue EchoJob, 'hi from Rails console'
</pre>

and our development log shows:

<pre>
[2011-02-14 14:37:10] YOUR_APP-RESQUE-a8e54bfb76718d09f8ed 48453 INFO EchoJob.perform: "hi from Rails console"
</pre>

Units of work initiated from Resque, for example if using a scheduler like
<a href="https://github.com/bvandenbos/resque-scheduler">resque-scheduler</a>,
will include 'RESQUE' in the transaction ID to indicate that the work started in Resque.

## Contributing

If you would like to contribute a fix or integrate Log Weasel transaction tracking into another frameworks
please fork the code, add the fix or feature in your local project and then send a pull request on github.
Please ensure that you include a test which verifies your changes.


## LICENSE

Copyright (c) 2011 Carbon Five. See LICENSE for details.