# Understanding workflow progress with Parsl monitoring

Ben Clifford
For parslfest 2021.
20 mins

the title of this talk is about monitoring - parsl has a monitoring component -
but I want to look at that through the lens of what is happening inside parsl,
rather than specific tools to look at monitoring data - to focus on
understanding what data is there, rather than how to look at it.

parsl can update an SQL database with what is happening in a workflow.
parsl-visualise can show you some data from that, but while working with users,
it turned out that lots of people are comfortable with making their own
database queries (for example, using SQL or loading into pandas (TODO: check
what tomg uses - i think pandas, not sure?)

In this talk I'm not going to try to convince you one way or another *how*
you should be accessing this data. Instead I want to talk about what happens
inside parsl when it is executing tasks, and how those processes are mapped
into rows in the database.

* note that I'm talking about the `master` version of parsl - so what will
become the next parsl release.

* things i can mention in passing / of note / new features - retry "fuel" for tries
(is that merged?)

* checkpointing: what does the hashsum mean? especially:
- how does it correlate tasks across runs - putting all stuff into one db means we can do stuff like that. and `hashsum` is the way to do it at task level
- but - the hashsum can't be known until all of the inputs are ready - so a task can exist that will eventually get a hash sum but if it still has incomplete dependencies, then the hashsum will not be populated yet. This can be a bit confusing if the dependencies are being used for ordering of effectful/external computations, rather than for calculating parameters, and maybe its something we should look at in future: you know in your head that two tasks are the same, but parsl doesn't yet.
- emphasise how task_ids don't align across runs - even though for some particular workflow patterns that might be the case.

## Initialising parsl

```python
import TODO

config = # TODO
  Config( .... whatever ... MonitoringHub())

parsl.load(config_with_monitoring)
```

creates a row in the `workflow` table. important thing for querying is that there is a unique workflow ID which we will use a lot. It doesn't contain any "meaning" - you just get a new one each time you run parsl.load.

So from the perspective of the monitoring db a "workflow" corresponds one-to-one to a parsl load (with monitoring configured).

By default, the monitoring database is created in the current directory. It is an sqlite file.

Here's the SQL schema - along with example values. Basically summary information about

```sql
$ sqlite3 monitoring.db
sqlite> .schema workflow
CREATE TABLE workflow (       
	run_id TEXT NOT NULL,    -- 8eb6a5d3-0275-4d2d-b93d-1db9812dab25
	workflow_name TEXT,      -- myworkflow.py
	workflow_version TEXT,   -- 2021-10-15 09:29:51
	time_began DATETIME NOT NULL,  -- 2021-10-15 09:29:51.256551
	time_completed DATETIME,       -- 2021-10-15 09:31:30.251855
	host TEXT NOT NULL,            -- laptop.cqx.ltd.uk
	user TEXT NOT NULL,            -- benc
	rundir TEXT NOT NULL,          -- /home/benc/parsl/src/parsl/runinfo/103
	tasks_failed_count INTEGER NOT NULL, -- 35
	tasks_completed_count INTEGER NOT NULL, -- 238
	PRIMARY KEY (run_id)
);
```

## Invoking an app

```python
@python_app
def hello():
  return "hi"

hello().result()
```

I can invoke an app. That result in what is called a `task` inside Parsl to
somehow run the app and return the result. We get a row in the `task`
table to describe this:

```sql
sqlite> .schema task
CREATE TABLE task (
	task_id INTEGER NOT NULL, -- 50
	run_id TEXT NOT NULL,  -- 8eb6a5d3-0275-4d2d-b93d-1db9812dab25

	task_func_name TEXT NOT NULL, -- hello

	task_depends TEXT, -- 32,33

	task_inputs TEXT, 
	task_outputs TEXT, 
	task_stdin TEXT, -- /path/to/stdin
	task_stdout TEXT,  -- /path/to/stdout
	task_stderr TEXT,  -- /path/to/stderr
	task_time_invoked DATETIME, -- 2021-10-15 09:31:17.133745
	task_time_returned DATETIME, -- 2021-10-15 09:31:27.337326

    -- memoization/caching/checkpointing
	task_memoize TEXT NOT NULL, -- 0 or 1
	task_hashsum TEXT, -- 5bb0ccc7c134a9cdf1a9248422003596

    -- retries
	task_fail_count INTEGER NOT NULL, -- 0
	task_fail_cost FLOAT NOT NULL, -- 0.0
	PRIMARY KEY (task_id, run_id)
);
```

The task_id is identifies the task within a particular workflow run, so with a db with multiple workflow runs in it, use a combination of the task_id and run_id.

There's some straightforward information about the task: timing, dependencies, paths for inputs/outputs.

There's also some information related to checkpoint and to retries. I'll come to these in the next sections.

## Trying

I've invoked an app, which has created a task.

In the simple case, parsl will try to run the code for that app to get a result. But that's not the only thing that might happen. If the app fails, it might try again - there can be several tries. Or it might turn out that the task is checkpointing, so parsl doesn't even need to try once.

This behaviour gives us the `try` table:

```sql
sqlite> .schema try
CREATE TABLE try (
	try_id INTEGER NOT NULL,  -- 0
	task_id INTEGER NOT NULL,  -- 33
	run_id TEXT NOT NULL,  -- 8eb6a5d3-0275-4d2d-b93d-1db9812dab25
	block_id TEXT, -- 3
	hostname TEXT, -- myexecutor.cqx.ltd.uk
	task_executor TEXT NOT NULL,  -- htex_example
	task_try_time_launched DATETIME,  -- 2021-10-15 09:31:29.624186
	task_try_time_running DATETIME, -- 2021-10-15 09:31:29.816768
	task_try_time_returned DATETIME, -- 2021-10-15 09:31:29.926946
	task_fail_history TEXT, -- TypeError('write() argument must be str, not int')
	task_joins TEXT, -- 20
	PRIMARY KEY (try_id, task_id, run_id)
);
```

For every task, there will be 0 or more try rows: 0 if the task was checkpointed, 1 if it ran without failures, and more than 1 if retries had to happen. The `try_id` combines with `task_id` and `run_id` to uniquely identify this try in the database.

In the case of a task that needs to be retried, there will be several `try` rows. Earlier tries will have failed, and so the `task_fail_history` field should contain a Python exception string in all but the final try - in the example above, a TypeError with an associated message.

The `task` table has two columns related to retries:

```sql
sqlite> .schema task
CREATE TABLE task (
	task_id INTEGER NOT NULL, -- 50
    -- ...
	task_fail_count INTEGER NOT NULL, -- 0
	task_fail_cost FLOAT NOT NULL, -- 0.0
    -- ...
);
```

The task fail count is the number of times that the task failed. In the common case, you might say "try each task three times", and this `task_fail_count` will be where you see that limit being counted up to.

Parsl retry handling (in `master`) is a little bit more generic than that: a workflow can specify different costs for different kinds of errors: for example, a particular application error might be a definitive failure that shouldn't be retried at all, but another error might be likely to be transient and so several retries should happen.

This is implemented by supply a `retry_handler` which, given a description of a failing task, returns the cost of that error. This then accumulates in the `task_fail_cost` column. By default, the cost is 1.0, and so the fail cost and the fail count will be the same. It is when this cost gets too much (rather than when the retry count gets too much) that a task is abandoned.


## Checkpointing, caching and memoization

That covers if there are one or more tries. But parsl can also do checkpointing (which is also referred to with subtle differences in meaning as caching and memoization). What that means is when you invoke an app and launch a task, parsl knows that some equivalent task already succeeded, and it can return the result from that, rather than making any tries at all.

In that case there won't be any try rows for this task - because parsl didn't need to try.

Sometimes it's useful to track down the try where the work actually was done - for example, to analyse the run times of every app invocation in a workflow, even when there have been checkpoints and restarts.

Parsl checkpointing has the notion of task equivalence: if two tasks are equivalent, then we can treat the results as equivalent. This equivalence is expressed through the `task_hashsum` field of the task table.


```sql
sqlite> .schema task
CREATE TABLE task (
	task_id INTEGER NOT NULL, -- 50
	run_id TEXT NOT NULL,  -- 8eb6a5d3-0275-4d2d-b93d-1db9812dab25
    -- ...
	task_hashsum TEXT, -- 5bb0ccc7c134a9cdf1a9248422003596
    -- ...
 );
```

This is a hash of all of the input parameters: if I invoke an app twice with the same input parameters, then the two resulting tasks will have the same hashsum.

This is one of the few places where it makes sense to make queries across workflow runs. If I see a task was not tried due to memoisation in a particular run, I can query across all runs to find all the other tasks with the same task_hashsum, and find any tries associated with all of those.

## status

Alongside the `try` table, tasks also get rows in the `status` table:

```sql
sqlite> .schema status
CREATE TABLE status (
	task_id INTEGER NOT NULL, 
	task_status_name TEXT NOT NULL, 
	timestamp DATETIME NOT NULL, 
	run_id TEXT NOT NULL, 
	try_id INTEGER NOT NULL, 
	PRIMARY KEY (task_id, run_id, task_status_name, timestamp), 
	FOREIGN KEY(task_id) REFERENCES task (task_id), 
	FOREIGN KEY(run_id) REFERENCES workflow (run_id)
);
```

For each task, this shows state transitions as task execution progresses. For example, here is a task that is attempted several times, failing each time and eventually being abandoned.

```
2021-10-15 09:14:10.078314    pending
2021-10-15 09:14:10.085996    launched
2021-10-15 09:14:10.132284    running
2021-10-15 09:14:10.256535    fail_retryable
2021-10-15 09:14:10.258465    pending
2021-10-15 09:14:10.265687    launched
2021-10-15 09:14:10.335339    running
2021-10-15 09:14:10.462653    fail_retryable
2021-10-15 09:14:10.464572    pending
2021-10-15 09:14:10.472266    launched
2021-10-15 09:14:10.536143    running
2021-10-15 09:14:10.665704    failed
```

I'll walk through these states:

- when an app is invoked, it is `pending`. It might not be possible to try to execute that app right away, because there might be uncompleted dependencies.
- when all those dependencies are completed (and only then), parsl can compute the `task_hashsum` and decide if it should try to execute the app. In this example, it decided to try to execute the app, by passing the app to an executor and marking the task as `launched`. This means that the core of parsl has handed over responsibility of execution to an executor, which probably has its own queues, and its own policies. For example, when Work Queue tries to arrange tasks using resource descriptions, it can only do that with tasks that are `launched` - it has no idea about `pending` tasks.
- eventually that task is run by the executor (perhaps after many hours of waiting in a queue), and the task is marked as `running`.
- In this example, the task then fails. Parsl computes the retry cost, and in this case, it decides that the task can be retried. it is marked as `fail_retryable` to record that fact, and then goes back into the same pending/launched/running/fail sequence.
- The third time round of this, after computing the retry cost, Parsl decides that this task shouldn't be retried any more - so the task goes to a final `failed` state, indicating that parsl is done with this task.

Here's a simpler example of a regular one-try task execution:

```
2021-10-15 09:12:36.809534    pending
2021-10-15 09:12:36.818164    launched
2021-10-15 09:12:36.892969    running
2021-10-15 09:12:37.011490    exec_done
```

The difference here is that instead of ending up `failed`, the final state is `exec_done`: the task is done successfully, by execution.

Here's a really short state sequence when a task doesn't need executing because it has been checkpointed:

```
2021-10-15 09:12:46.708270|pending
2021-10-15 09:12:46.717982|memo_done
```

Almost nothing happens: as soon as Parsl can check the parameters for checkpointing, the task goes to a final state of `memo_done`: the task is successfully done, because of memoization rather than because execution was tried.

## summary so far

Those four tables: workflow, task, try and status form a tree of information about app invocations.

Next I'll talk about some other tables which contain information about the execution environment.

## blocks

Some executors in parsl follow the pattern of a pool of workers running on many nodes (for example, on a large HPC system) - for example the `HighThroughputExecutor` and the `WorkQueueExecutor`.
