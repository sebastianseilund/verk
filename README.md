# Verk [![Build Status](https://travis-ci.org/edgurgel/verk.svg?branch=master)](https://travis-ci.org/edgurgel/verk) [![Hex pm](http://img.shields.io/hexpm/v/verk.svg?style=flat)](https://hex.pm/packages/verk) [![Coverage Status](https://coveralls.io/repos/edgurgel/verk/badge.svg?branch=master&service=github)](https://coveralls.io/github/edgurgel/verk?branch=master) [![hex.pm downloads](https://img.shields.io/hexpm/dt/verk.svg?style=flat)](https://hex.pm/packages/verk)

Verk is a job processing system backed by Redis. It uses the same job definition of Sidekiq/Resque.

The goal is to be able to isolate the execution of a queue of jobs as much as possible.

Every queue has its own supervision tree:

* A pool of workers;
* A `QueueManager` that interacts with Redis to get jobs and enqueue them back to be retried if necessary;
* A `WorkersManager` that will interact with the `QueueManager` and the pool to execute jobs.

Verk will hold one connection to Redis per queue plus one dedicated to the `ScheduleManager` and one general connection for other use cases like deleting a job from retry set or enqueuing new jobs.

The `ScheduleManager` fetches jobs from the `retry` set to be enqueued back to the original queue when it's ready to be retried.

It also has one GenStage producer called `Verk.EventProducer`.

The image below is an overview of Verk's supervision tree running with a queue named `default` having 5 workers.

![Supervision Tree](http://i.imgur.com/ATsNAvJ.png)

Feature set:

* Retry mechanism with exponential backoff
* Dynamic addition/removal of queues
* Reliable job processing (RPOPLPUSH and Lua scripts to the rescue)
* Error and event tracking

## Installation

First, add Verk to your `mix.exs` dependencies:

```elixir
def deps do
  [{:verk, "~> 1.0"}]
end
```

and run `$ mix deps.get`. Add `:verk` to your applications list if your Elixir version is 1.3 or lower:

```elixir
def application do
  [applications: [:verk]]
end
```

Add `Verk.Supervisor` to your supervision tree:

```elixir
defmodule Example.App do
  use Application

  def start(_type, _args) do
    import Supervisor.Spec
    tree = [supervisor(Verk.Supervisor, [])]
    opts = [name: Simple.Sup, strategy: :one_for_one]
    Supervisor.start_link(tree, opts)
  end
end
```

Finally we need to configure how Verk will process jobs.

## Configuration

Example configuration for Verk having 2 queues: `default` and `priority`

The queue `default` will have a maximum of 25 jobs being processed at a time and `priority` just 10.

```elixir
config :verk, queues: [default: 25, priority: 10],
              max_retry_count: 10,
              poll_interval: 5000,
              start_job_log_level: :info,
              done_job_log_level: :info,
              fail_job_log_level: :info,
              node_id: "1",
              redis_url: "redis://127.0.0.1:6379"
```

Verk supports the convention `{:system, "ENV_NAME", default}` for reading environment configuration at runtime using [Confex](https://hexdocs.pm/confex/readme.html):

```elixir
config :verk, queues: [default: 25, priority: 10],
              max_retry_count: 10,
              poll_interval: {:system, :integer, "VERK_POLL_INTERVAL", 5000},
              start_job_log_level: :info,
              done_job_log_level: :info,
              fail_job_log_level: :info,
              node_id: "1",
              redis_url: {:system, "VERK_REDIS_URL", "redis://127.0.0.1:6379"}
```

Now Verk is ready to start processing jobs! :tada:

## Workers

A job is defined by a module and arguments:

```elixir
defmodule ExampleWorker do
  def perform(arg1, arg2) do
    arg1 + arg2
  end
end
```

This job can be enqueued using `Verk.enqueue/1`:

```elixir
Verk.enqueue(%Verk.Job{queue: :default, class: "ExampleWorker", args: [1,2], max_retry_count: 5})
```

This job can also be scheduled using `Verk.schedule/2`:

 ```elixir
 perform_at = Timex.shift(Timex.now, seconds: 30)
 Verk.schedule(%Verk.Job{queue: :default, class: "ExampleWorker", args: [1,2]}, perform_at)
 ```

### Retry at

A job can define the function `retry_at/2` for custom retry time delay:

```elixir
defmodule ExampleWorker do
  def perform(arg1, arg2) do
    arg1 + arg2
  end

  def retry_at(failed_at, retry_count) do
    failed_at + retry_count
  end
end
```

In this example, the first retry will be scheduled a second later,
the second retry will be scheduled two seconds later, and so on.

If `retry_at/2` is not defined the default exponential backoff is used.

### Keys in arguments

By default, Verk will decode keys in arguments to binary strings.
You can change this behavior for jobs enqueued by Verk with the following configuration:
```elixir
config :verk, :args_keys, value
```

The following values are valid:

* `:strings` (default) - decodes keys as binary strings
* `:atoms` - keys are converted to atoms using `String.to_atom/1`
* `:atoms!` - keys are converted to atoms using `String.to_existing_atom/1`


### On Heroku

Heroku provides [an experimental environment variable](https://devcenter.heroku.com/articles/dynos#local-environment-variables) named after the type and number of the dyno. _It is possible that two dynos with the same name could overlap for a short time during a dyno restart._

```elixir
config :verk, queues: [default: 25, priority: 10],
              max_retry_count: 10,
              poll_interval: {:system, :integer, "VERK_POLL_INTERVAL", 5000},
              start_job_log_level: :info,
              done_job_log_level: :info,
              fail_job_log_level: :info,
              node_id: {:system, "DYNO", "job.1"},
              redis_url: {:system, "VERK_REDIS_URL", "redis://127.0.0.1:6379"}
```

## Queues

It's possible to dynamically add and remove queues from Verk.

```elixir
Verk.add_queue(:new, 10) # Adds a queue named `new` with 10 workers
```

```elixir
Verk.remove_queue(:new) # Terminate and delete the queue named `new`
```

## Reliability

Verk's goal is to never have a job that exists only in memory. It uses Redis as the single source of truth to retry and track jobs that were being processed if some crash happened.

Verk will re-enqueue jobs if the application crashed while jobs were running. It will also retry jobs that failed keeping track of the errors that happened.

The jobs that will run on top of Verk should be idempotent as they may run more than once.

## Error tracking

One can track when jobs start and finish or fail. This can be useful to build metrics around the jobs. The `QueueStats` handler does some kind of metrics using these events: https://github.com/edgurgel/verk/blob/master/lib/verk/queue_stats.ex

Verk has an Event Manager that notifies the following events:

* `Verk.Events.JobStarted`
* `Verk.Events.JobFinished`
* `Verk.Events.JobFailed`
* `Verk.Events.QueueRunning`
* `Verk.Events.QueuePausing`
* `Verk.Events.QueuePaused`

One can define an error tracking handler like this:

```elixir
defmodule TrackingErrorHandler do
  use GenStage

  def start_link() do
    GenStage.start_link(__MODULE__, :ok)
  end

  def init(_) do
    filter = fn event -> event.__struct__ == Verk.Events.JobFailed end
    {:consumer, :state, subscribe_to: [{Verk.EventProducer, selector: filter}]}
  end

  def handle_events(events, _from, state) do
    Enum.each(events, &handle_event/1)
    {:noreply, [], state}
  end

  defp handle_event(%Verk.Events.JobFailed{job: job, failed_at: failed_at, stacktrace: trace}) do
    MyTrackingExceptionSystem.track(stacktrace: trace, name: job.class)
  end
end
```

Notice the selector to get just the type JobFailed. If no selector is set every event is sent.

Then adding the consumer to your supervision tree:

  ```elixir
  defmodule Example.App do
    use Application

    def start(_type, _args) do
      import Supervisor.Spec
      tree = [supervisor(Verk.Supervisor, []),
              worker(TrackingErrorHandler, [])]
      opts = [name: Simple.Sup, strategy: :one_for_one]
      Supervisor.start_link(tree, opts)
    end
  end
  ```

## Dashboard ?

Check [Verk Web](https://github.com/edgurgel/verk_web)!

![Dashboard](http://i.imgur.com/LsDKIVT.png)

## Sponsorship

Initial development sponsored by [Carnival.io](http://carnival.io)
