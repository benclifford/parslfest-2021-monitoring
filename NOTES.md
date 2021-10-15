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
into records in the database.

* note that I'm talking about the `master` version of parsl - so what will
become the next parsl release.

* things i can mention in passing / of note / new features - retry "fuel" for tries
(is that merged?)

* checkpointing: what does the hashsum mean? especially:
- how does it correlate tasks across runs - putting all stuff into one db means we can do stuff like that. and `hashsum` is the way to do it at task level
- but - the hashsum can't be known until all of the inputs are ready - so a task can exist that will eventually get a hash sum but if it still has incomplete dependencies, then the hashsum will not be populated yet. This can be a bit confusing if the dependencies are being used for ordering of effectful/external computations, rather than for calculating parameters, and maybe its something we should look at in future: you know in your head that two tasks are the same, but parsl doesn't yet.

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
  return "hello"

hello().result()
```

I can invoke an app. That result in what is called a `task`.
