[gem]: https://rubygems.org/gems/sidekiq-batch
[travis]: https://travis-ci.org/breamware/sidekiq-batch
[codeclimate]: https://codeclimate.com/github/breamware/sidekiq-batch

# Sidekiq::Batch

[![Join the chat at https://gitter.im/breamware/sidekiq-batch](https://badges.gitter.im/breamware/sidekiq-batch.svg)](https://gitter.im/breamware/sidekiq-batch?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[![Gem Version](https://badge.fury.io/rb/sidekiq-batch.svg)][gem]
[![Build Status](https://travis-ci.org/breamware/sidekiq-batch.svg?branch=master)][travis]
[![Code Climate](https://codeclimate.com/github/breamware/sidekiq-batch/badges/gpa.svg)][codeclimate]
[![Code Climate](https://codeclimate.com/github/breamware/sidekiq-batch/badges/coverage.svg)][codeclimate]
[![Code Climate](https://codeclimate.com/github/breamware/sidekiq-batch/badges/issue_count.svg)][codeclimate]

Simple Sidekiq Batch Job implementation.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'sidekiq-batch'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install sidekiq-batch

## Usage

Sidekiq Batch is MOSTLY a drop-in replacement for the API from Sidekiq PRO. See https://github.com/mperham/sidekiq/wiki/Batches for usage.

## Caveats/Gotchas

Consider the following workflow:

  * Batch Z created
  * Worker A queued in batch Z
  * Worker A starts Worker B in batch Z
  * Worker B completes *before* worker A does
  * Worker A completes

In the standard configuration, the `on(:success)` and `on(:complete)` callbacks will be triggered when Worker B completes.
This configuration is the default, simply for legacy reasons. This gem adds the following option to the sidekiq.yml options:

```yaml
:batch_push_interval: 0
```

When this value is *absent* (aka legacy), Worker A will only push the increment of batch jobs (aka Worker B) *when it completes*

When this value is set to `0`, Worker A will increment the count as soon as `WorkerB.perform_async` is called

When this value is a positive number, Worker A will wait a maximum of value-seconds before pushing the increment to redis, or until it's done, whichever comes first.

This comes into play if Worker A is queueing thousands of WorkerB jobs, or has some other reason for WorkerB to complete beforehand.

If you are queueing many WorkerB jobs, it is recommended to set this value to something like `3` to avoid thousands of calls to redis, and call WorkerB like so:
```ruby
WorkerB.perform_in(4.seconds, some, args)
```
this will ensure that the batch callback does not get triggered until WorkerA *and* the last WorkerB job complete.

If WorkerA is just slow for whatever reason, setting to `0` will update the batch status immediately so that the callbacks don't fire.


## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/breamware/sidekiq-batch.


## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).


## Changes in this fork

Copied from this Pull Request: https://github.com/breamware/sidekiq-batch/pull/47/files
There are multiple bugs reported on how the complete/success callback is called on batches.
This version fixes (tries to fix) 2 bugs (unwanted behaviours): 
1. The complete callback is called sometimes to early, before batch completion 
2. The total number of pending jobs from the batch sometimes differ from the initial jobs enqueued initially 

Both issues are caused (in my case) when the jobs from the batch are enqueuing jobs that are not related to the batch. 
Also this is reproducible only when concurrency is > 1

#### How the fix works:  
We consider that only the "Allowed" classes can be added to the batch queue.
To enable this behaviour, the following constants should be set in an initializer:
```
Sidekiq::Batch::Extension::KnownBatchBaseKlass::ENABLED = true
Sidekiq::Batch::Extension::KnownBatchBaseKlass::ALLOWED = [MyBaseBatchWorker].freeze
```
By enabling this behaviour, jobs that are enqueued by jobs from our batch, are not added to the batch also. Only the ones that have as an ancestor a base class defined by us.
