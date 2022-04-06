# Sidekiq-Cron

[![Gem Version](https://badge.fury.io/rb/sidekiq-cron.svg)](https://badge.fury.io/rb/sidekiq-cron)
[![Build Status](https://github.com/ondrejbartas/sidekiq-cron/workflows/CI/badge.svg?branch=master)](https://github.com/ondrejbartas/sidekiq-cron/actions)
[![Coverage Status](https://coveralls.io/repos/github/ondrejbartas/sidekiq-cron/badge.svg?branch=master)](https://coveralls.io/github/ondrejbartas/sidekiq-cron?branch=master)
[![Join the chat at https://gitter.im/ondrejbartas/sidekiq-cron](https://badges.gitter.im/ondrejbartas/sidekiq-cron.svg)](https://gitter.im/ondrejbartas/sidekiq-cron?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

> A scheduling add-on for [Sidekiq](https://sidekiq.org/)

🎬 [Introduction video about Sidekiq-Cron by Drifting Ruby](https://www.driftingruby.com/episodes/periodic-tasks-with-sidekiq-cron)

Sidekiq-Cron runs a thread alongside Sidekiq workers to schedule jobs at specified times (using cron notation `* * * * *` parsed by [Fugit](https://github.com/floraison/fugit).

Checks for new jobs to schedule every 30 seconds and doesn't schedule the same job multiple times when more than one Sidekiq worker is running.

Scheduling jobs are added only when at least one Sidekiq process is running, but it is safe to use Sidekiq-Cron in environments where multiple Sidekiq processes or nodes are running.

If you want to know how scheduling work, check out [under the hood](#under-the-hood).

Works with ActiveJob (Rails 4.2+).

You don't need Sidekiq PRO, you can use this gem with plain Sidekiq.

## Upgrade from < 0.6 to 1.0

Please be aware that Sidekiq-Cron < 1.0 was relying on rufus-scheduler < 3.5. Using those older versions with rufus-scheduler >= 3.5 ends up with jobs failing on creation. Sidekiq-Cron 1.0 includes a patch that switches from rufus-scheduler to rufus-scheduler's core dependency, fugit.

## Changelog

Before upgrading to a new version, please read our [Changelog](Changes.md).

## Installation

### Requirements

- Redis 2.8 or greater is required (Redis 3.0.3 or greater is recommended for large scale use)
- Sidekiq 4.2 or greater is required (for Sidekiq < 4 use version sidekiq-cron 0.3.1)

Install the gem:

```
$ gem install sidekiq-cron
```

Or add to your `Gemfile` and run `bundle install`:

```ruby
gem "sidekiq-cron", "~> 1.3"
```

**NOTE** If you are not using Rails, you need to add `require 'sidekiq-cron'` somewhere after `require 'sidekiq'`.

## Getting Started

**Job properties:**

```ruby
{
 'name' => 'name_of_job', # must be uniq!
 'cron' => '1 * * * *',  # execute at 1 minute of every hour, ex: 12:01, 13:01, 14:01, 15:01, ... (HH:MM)
 'class' => 'MyClass',
 # OPTIONAL
 'queue' => 'name of queue',
 'args' => '[Array or Hash] of arguments which will be passed to perform method',
 'date_as_argument' => true, # add the time of execution as last argument of the perform method
 'active_job' => true,  # enqueue job through rails 4.2+ active job interface
 'queue_name_prefix' => 'prefix', # rails 4.2+ active job queue with prefix
 'queue_name_delimiter' => '.',  # rails 4.2+ active job queue with custom delimiter
 'description' => 'A sentence describing what work this job performs.'
}
```

### Time, cron and Sidekiq-Cron

For testing your cron notation you can use [crontab.guru](https://crontab.guru).

Sidekiq-Cron uses [Fugit](https://github.com/floraison/fugit) to parse the cronline.

If using Rails, this is evaluated against the timezone configured in Rails, otherwise the default is UTC.

If you want to have your jobs enqueued based on a different time zone you can specify a timezone in the cronline,
like this `'0 22 * * 1-5 America/Chicago'`.

See [rufus-scheduler documentation](https://github.com/jmettraux/rufus-scheduler#a-note-about-timezones) for more information.

**NOTE** Rufus scheduler is using Fugit under the hood, so documentation for Rufus Scheduler can help you also.

### What objects/classes can be scheduled

#### Sidekiq Worker

In this example, we are using `HardWorker` which looks like:

```ruby
class HardWorker
  include Sidekiq::Worker

  def perform(*args)
    # do something
  end
end
```

#### Active Job Worker

You can schedule `ExampleJob` which looks like:

```ruby
class ExampleJob < ActiveJob::Base
  queue_as :default

  def perform(*args)
    # Do something
  end
end
```

For Active jobs you can use `symbolize_args: true` in `Sidekiq::Cron::Job.create` or in Hash configuration,
which will ensure that arguments you are passing to it will be symbolized when passed back to `perform` method in worker.

#### Adding Cron job

```ruby
class HardWorker
  include Sidekiq::Worker

  def perform(name, count)
    # do something
  end
end

Sidekiq::Cron::Job.create(name: 'Hard worker - every 5min', cron: '*/5 * * * *', class: 'HardWorker') # execute at every 5 minutes, ex: 12:05, 12:10, 12:15...etc
# => true
```

`create` method will return only true/false if job was saved or not.

```ruby
job = Sidekiq::Cron::Job.new(name: 'Hard worker - every 5min', cron: '*/5 * * * *', class: 'HardWorker')

if job.valid?
  job.save
else
  puts job.errors
end

# or simple
unless job.save
  puts job.errors # will return array of errors
end
```

Load more jobs from hash:

```ruby
hash = {
  'name_of_job' => {
    'class' => 'MyClass',
    'cron'  => '1 * * * *',
    'args'  => '(OPTIONAL) [Array or Hash]'
  },
  'My super iber cool job' => {
    'class' => 'SecondClass',
    'cron'  => '*/5 * * * *'
  }
}

Sidekiq::Cron::Job.load_from_hash hash
```

Load more jobs from array:

```ruby
array = [
  {
    'name'  => 'name_of_job',
    'class' => 'MyClass',
    'cron'  => '1 * * * *',
    'args'  => '(OPTIONAL) [Array or Hash]'
  },
  {
    'name'  => 'Cool Job for Second Class',
    'class' => 'SecondClass',
    'cron'  => '*/5 * * * *'
  }
]

Sidekiq::Cron::Job.load_from_array array
```

Bang-suffixed methods will remove jobs that are not present in the given hash/array, update jobs that have the same names, and create new ones when the names are previously unknown.

```ruby
Sidekiq::Cron::Job#load_from_hash! hash
Sidekiq::Cron::Job#load_from_array! array
```

Or from YAML (same notation as Resque-scheduler):

```yaml
# config/schedule.yml

my_first_job:
  cron: "*/5 * * * *"
  class: "HardWorker"
  queue: hard_worker

second_job:
  cron: "*/30 * * * *" # execute at every 30 minutes
  class: "HardWorker"
  queue: hard_worker_long
  args:
    hard: "stuff"
```

```ruby
# config/initializers/sidekiq.rb
schedule_file = "config/schedule.yml"

if File.exist?(schedule_file) && Sidekiq.server?
  Sidekiq::Cron::Job.load_from_hash YAML.load_file(schedule_file)
end
```

From version 3.x it is better not to use separate initializer of schedule instead add `config.on(:startup)` to your Sidekiq configuration:

```ruby
Sidekiq.configure_server do |config|
  config.on(:startup) do
    schedule_file = "config/schedule.yml"

    if File.exist?(schedule_file)
      Sidekiq::Cron::Job.load_from_hash YAML.load_file(schedule_file)
    end
  end
end
```

Or you can use for loading jobs from yml file [sidekiq-cron-tasks](https://github.com/coverhound/sidekiq-cron-tasks) which will add rake task `bundle exec rake sidekiq_cron:load` to your rails application.

### Finding jobs

```ruby
# return array of all jobs
Sidekiq::Cron::Job.all

# return one job by its unique name - case sensitive
Sidekiq::Cron::Job.find "Job Name"

# return one job by its unique name - you can use hash with 'name' key
Sidekiq::Cron::Job.find name: "Job Name"

# if job can't be found nil is returned
```

### Destroy jobs

```ruby
# destroy all jobs
Sidekiq::Cron::Job.destroy_all!

# destroy job by its name
Sidekiq::Cron::Job.destroy "Job Name"

# destroy found job
Sidekiq::Cron::Job.find('Job name').destroy
```

### Work with job

```ruby
job = Sidekiq::Cron::Job.find('Job name')

# disable cron scheduling
job.disable!

# enable cron scheduling
job.enable!

# get status of job:
job.status
# => enabled/disabled

# enqueue job right now!
job.enque!
```

### How to start scheduling?

Just start Sidekiq workers by running:

```
$ sidekiq
```

### Web UI for Cron Jobs

If you are using Sidekiq's web UI and you would like to add cron jobs too to this web UI,
add `require 'sidekiq/cron/web'` after `require 'sidekiq/web'`.

With this, you will get:

![Web UI](examples/web-cron-ui.png)

### Forking Processes or problem with `NotImplementedError`

If you're using a forking web server like Unicorn you may run into an issue where the Redis connection is used
before the process forks, causing the following exception to occur:

```
Redis::InheritedError: Tried to use a connection from a child process without reconnecting. You need to reconnect to Redis after forking.
```

To avoid this, wrap your job creation in the call to `Sidekiq.configure_server`:

```ruby
Sidekiq.configure_server do |config|
  config.on(:startup) do
    schedule_file = "config/schedule.yml"

    if File.exist?(schedule_file)
      Sidekiq::Cron::Job.load_from_hash YAML.load_file(schedule_file)
    end
  end
end
```

**NOTE** This API is only available in Sidekiq 3.x.

## Under the hood

When you start the Sidekiq process, it starts one thread with `Sidekiq::Poller` instance, which perform the adding of scheduled jobs to queues, retries etc.

Sidekiq-Cron adds itself into this start procedure and starts another thread with `Sidekiq::Cron::Poller` which checks all enabled Sidekiq cron jobs every 30 seconds, if they should be added to queue (their cronline matches time of check).

Sidekiq-Cron is checking jobs to be enqueued every 30s by default, you can change it by setting:

```ruby
Sidekiq.options[:average_scheduled_poll_interval] = 10
```

Sidekiq-Cron is safe to use with multiple Sidekiq processes or nodes. It uses a Redis sorted set to determine that only the first process who asks can enqueue scheduled jobs into the queue.

## Contributing

**Thanks to all [contributors](https://github.com/ondrejbartas/sidekiq-cron/graphs/contributors), you’re awesome and this wouldn’t be possible without you!**

* Check out the latest master to make sure the feature hasn't been implemented or the bug hasn't been fixed yet.
* Check out the issue tracker to make sure someone already hasn't requested it and/or contributed it.
* Fork the project.
* Start a feature/bugfix branch.
* Commit and push until you are happy with your contribution.
* Make sure to add tests for it. This is important so we don't break it in a future version unintentionally.

### Testing

You can execute the test suite by running:

```
$ bundle exec rake test
```

## License

Copyright (c) 2013 Ondrej Bartas. See [LICENSE](LICENSE.txt) for further details.
