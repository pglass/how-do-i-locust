# How do I Locust.

[Locust](https://locust.io/) generates load and does pretty much nothing else.
It comes with a web server with UI controls out of the box. It can also run in
a single-master, multiple-worker configuration in which the workers generate load
and the master aggregates reports from the workers.

You write a locust file, which is just regular python file

```python
# a descriptively named locust file
something.py
```

Put a task set in your locust file. Each task typically contains a particular
API operation.

```python
import locust
class MyTaskSet(locust.TaskSet):

    @locust.task
    def do_get(self):
        self.client.get('/resources/{id}')
```

Then define other test parameters, including the api endpoint to hit and the
wait time:

```python
class MyLocust(locust.HttpLocust):
    task_set = MyTaskSet

    min_wait = 900
    max_wait = 1100

    host = 'http://localhost:8000'
```

The min/max wait times control the amount of time each simulated user waits
between executing tasks. Each user will execute a task at random, wait a random
amount of time between `min_wait` and `max_wait`, and then repeat.

At this point, you can start locust's web server:

```bash
$ locust -f something.py
```

Then you can start the test in one of two ways:

1. Go to `localhost:8089` in your browser and type in numbers and click start.
2. POST `localhost:8089/swarm` with `{"locust_count": 3, "hatch_rate": 1}`

The locust count is the total number of users to spawn. The hatch rate is the
number of users to spawn per second, starting from zero when load generation
first begins. (The "hatching" period is the rampup period from when the test
first starts until the max number of users is reached.)

Each user does the following:

1. Pick one of the tasks from your locust file
2. Run the task (execute that task function)
3. Pick a random wait time between min_wait and max_wait (specified in your
locust file)
4. Wait that amount of time
5. Repeat from 1


### But how do I use workers?

Locust can be run in a single-master, multiple-worker configuration. The workers
do all the load generation, while the master controls and monitors. Every (by
default) 3 seconds, a worker sends a single report for all requests made *on
that worker* in the last 3 seconds. The master receives these reports from all
of its workers and consolidates them in real time.

The master controls the starting and stopping of *load generation* on the
workers. The master cannot start/stop the locust process running on the workers.
This means you need to create servers and start locust's processes yourself.

To start the master process:

```bash
$ locust -f something.py --master
```

You must start the master process before the workers. Then start the workers:

```bash
$ locust -f something.py --worker --master-host=<master-ip>
```

You should be able see the clients connect/disconnect in the master's logs.

### Random tips

##### Detecting whether you're the master or worker
It seems normal to use the same locust file on the master and worker (I've never
tried using a master locust file and a separate, different worker locust file.
Even if it works, you lose the ability to run your tests in non-distributed
mode.)

But sometimes I wanted to do certain things only on the master (like saving a
report to disk) and some things only on the worker (like doing some data prep
before starting load generation).

From what I can tell, locust doesn't provide a way to detect which mode you're
running in. What I did was check for the command line arguments:

```python
import sys

def is_master():
    return '--master' in sys.argv

def is_worker():
    return '--worker' in sys.argv
```

##### Event hooks
Locust has some nifty event hooks that let you execute a function when that
event is fired. Locust has [some built-in
events](http://docs.locust.io/en/latest/api.html#available-hooks), or you can
create your own:

```python
import locust.events

my_event = locust.events.EventHook()

def handler1(a, b): print "handled", a, b
def handler2(a, b): print "sum", a + b

my_event += handler1
my_event += handler2

# invokes each event handler in the order you inserted them
# note that you have to use keyworded arguments when you call .fire()
my_event.fire(a=1, b=2)
```

##### Pre-test actions
There are several different points at which you can do something before the
test starts. Some of these are kind of non-obvious, and I ran into situations
where a setup task was being run every time a new user was spawned (during
rampup) instead of just when load generation started. Anyway, these are the
points at which you can do something before/after your test:

1. *At process startup*: You can do whatever you like in the global scope of
your locust file, but this will only occur once for the entire time your
locust process is running. I read config files and do other "framework" setup,
like registering event handlers or preparing integration with another service.

```python
import locust
# read the config file once, at the start of the locust process
CONF = get_config()
# only register this event handler once
locust.events.locust_start_hatching += get_auth_token

class MyTasks(locust.TaskSet):
    ...
```

2. *At test start*: You click the start button to start load generation You can
hook into this by attaching event handlers to the
`locust.events.locust_start_hatching` event:

```python
def do_thing():
    # do a thing
locust.events.locust_start_hatching += do_thing
```

This will call `do_thing` exactly once when you press the start button. If
you're running locust in master/worker mode, then
`locust.events.locust_start_hatching` fires only on workers, and
`locust.events.master_start_hatching` fires only on the master.

3. *At user spawn time*: You can do per-user setup in two places. Locust will
call the `on_start` method when a Locust user is started:

```python
class MyTasks(locust.TaskSet):
    def on_start(self):
        # Each locust user gets a different id
        self.random_id = str(uuid.uuid4())
```

Locust creates a new instance of your TaskSet class once per user, so you can
also do setup in the class constructor (I think this is less preferred):

```python
class MyTasks(locust.TaskSet):
    def __init__(self, *args, **kwargs):
        super(MyTasks, self).__init__(*args, **kwargs)
        # Each locust user gets a different id
        self.random_id = str(uuid.uuid4())
```

##### Grouping requests in the report
By default, locust will use `/resources/{uuid}` as the name that shows up in
the summary report. But if your the url has an id in it, you'll get a separate
entry in locust's report for each id.

Instead, you can provide an explicit `name` argument for Locust to use in its
report:

```python
@locust.task
def do_get(self):
    self.client.get('/resources/{id}', name='/resources/UUID')
```

##### Manually marking requests as success/failure
Normally, locust automatically records a success or failure by looking at the
http status code. You can use a `with` block to override this:

```python
with self.client.post('/things', catch_response=True) as post_resp:
    if post_resp.json()['status'] == 'ACTIVE':
        post_resp.success()
    elif post_resp.json()['status'] == 'ERROR':
        post_resp.failure("Saw ERROR status")
    else:
        post_resp.failure("Unknown error")
```

##### Achieving precise request rates
You can compute the approximate request rate using the average wait time and
the total number of users. This will be inaccurate though, because it doesn't
take into account the time each user spends executing a task. This is a problem
when you have long running tasks (e.g. if you need to poll).

For example, consider a long running task like this:

```python
@task
def do_sleep(self):
    gevent.sleep(60)
```

A locust user will enter this task and sleep for 60 seconds. If you have a
small number of users, chances are they'll all eventually get stuck sleeping in
this task, unable to continue executing other tasks, so your request rates will
plummet. (Due to reasons, you should actually use gevent.sleep() instead of
time.sleep(). See the next section for more on this.)

Instead, you want the task to take as short a time as possible. To do this, you
can spawn a new greenlet that sleeps in the background:

```python
def _do_async_thing_handler(self):
    gevent.sleep(60)

@task
def do_async_thing(self):
    gevent.spawn(self._do_async_thing_handler)
```

This accomplishes two things:

1. Your task (do_async_thing) returns effectively immediately, and the locust
user can go on to do other things. This means your actual request rate
should much be closer to what you expect.
2. Your task's functionality (_do_async_thing_handler) continues running in the
background and will terminate on its own.

##### How do I sleep

With gevent, you should use `gevent.sleep` instead of `time.sleep` to avoid
"blocking the world", or causing all your task functions to block/sleep. Why?
Because gevent operates entirely within a single OS thread. Calling
`time.sleep` actually sleeps the thread, which means gevent cannot continue
executing your task functions.

*However*, Locust runs gevent's [monkey
patching](http://www.gevent.org/intro.html#monkey-patching), during which
gevent replaces certain functions from the Python standard library with
gevent-friendly versions. During this monkey patching step, `time.sleep` is
replaced with `gevent.sleep`. After monkey patching occurs, `time.sleep` _is_
the `gevent.sleep` function and you have nothing to worry about.

But, how do I not block the world? Either,

- Call `gevent.sleep` explicitly within your Locust task functions and other
Locust-related code, OR
- Ensure gevent is monkey patched, so that `time.sleep` can be used safely.
Locust runs monkey patching for you, but you must have performed an `import
locust` _prior_ to using `time.sleep`. (You can also call
`gevent.monkey.patch_all()` explicitly yourself, if you need to).

Monkey patching gevent is a must when you are using an external library that
can't be modified to use `gevent.sleep`.

##### Asynchronous polling

You can use a similar pattern as above to poll for a status. One thing to
watch out for is having too many greenlets running in the background on a single
machine (you can lower the polling frequency or run in distributed mode to
help manage this).

First, I added functions to report the result of asynchronous operations to Locust.
These use the native [request_success](
https://docs.locust.io/en/stable/api.html#locust.event.Events.request_success)
and [request_failure](
https://docs.locust.io/en/stable/api.html#locust.event.Events.request_failure)
events.

```python
def async_success(name, start_time, resp):
    locust.events.request_success.fire(
        request_type=resp.request.method,
        name=name,
        response_time=int((time.monotonic() - start_time) * 1000),
        response_length=len(resp.content),
    )

def async_failure(name, start_time, resp, message):
    locust.events.request_failure.fire(
        request_type=resp.request.method,
        name=name,
        response_time=int((time.monotonic() - start_time) * 1000),
        exception=Exception(message),
    )
```

Then, I implemented polling logic in an async "handler" function and called
the `async_success`/`async_failure` functions above when polling is complete.
These calculate the total elapsed time for the async operation, and report
the result to Locust. I also added the usual `@task` function to spawn the async
handler function in the background.

```python
def _do_async_thing_handler(self, timeout=600):
    post_resp = self.client.post('/things')
    if not post_resp.ok:
        return
    id = post_resp.json()['id']

    # Now poll for an ACTIVE status
    start_time = time.monotonic()
    end_time = start_time + timeout
    while time.monotonic() < end_time:
        r = self.client.get('/things/' + id)
        if r.ok and r.json()['status'] == 'ACTIVE':
            async_success('POST /things/ID - async', start_time, post_resp)
            return
        elif r.ok and r.json()['status'] == 'ERROR':
            async_failure('POST /things/ID - async', start_time, post_resp,
                          'Failed - saw ERROR status')
            return

        # IMPORTANT: Sleep must be monkey-patched by gevent (typical), or else
        # use gevent.sleep to avoid blocking the world.
        time.sleep(1)
    async_failure('POST /things/ID - async', start_time, post_resp,
                  'Failed - timed out after %s seconds' % timeout)

@task
def do_async_thing(self):
    gevent.spawn(self._do_async_thing_handler)
```

The key things here are:

1. Spawning a new greenlet in your `@task` function so that other Locust tasks
are not blocked by the long-running polling loop. This ensures that the `do_async_thing`
task is executed at a predictable (request) rate, that will not be affected by
the time spent in the polling loop.
2. Calling the `async_success`/`async_failure` functions to compute the duration
until the async operation was finished. These used the `request_success`/`request_failure`
event hooks to report the result to Locust, which is included in the request metrics.

(Another issue here: When I stop load generation, any greenlets running in the background
will continue running - until the greenlets finish running, or until the process is
terminated. I tracked my greenlets in a list and used an event hook to kill them all when
the test is stopped.)
