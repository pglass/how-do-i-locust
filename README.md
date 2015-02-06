# How do I Locust.

Locust generates load and does pretty much nothing else. It comes with a web
server with UI controls out of the box. It can also run in a single-master,
multiple-slave configuration in which the slaves generate load and the master
aggregates reports from the slaves.

You write a locust file, which is just regular python file

    # a descriptively named locust file
    something.py

Put a task set in your locust file. Each task typically contains a particular
API operation.

    import locust
    class MyTaskSet(locust.TaskSet):

        @locust.task
        def do_get(self):
            self.client.get('/resources/{id}')

Then define other test parameters, including the api endpoint to hit and the wait time:

    class MyLocust(locust.HttpLocust):
        task_set = MyTaskSet

        min_wait = 900
        max_wait = 1100

        host = 'http://localhost:8000'

The min/max wait times control the amount of time each simulated user waits 
between executing tasks. Each user will execute a task at random, wait a random 
amount of time between `min_wait` and `max_wait`, and then repeat.

At this point, you can start locust's web server:

    locust -f something.py

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
3. Pick a random wait time between min_wait and max_wait (specified in your locust file)
4. Wait that amount of time
5. Repeat from 1


### But how do I slavery??

Locust can be run in a single-master, multiple-slave configuration. The slaves
do all the load generation, while the master controls and monitors. Every (by default)
3 seconds, a slave sends a single report for all requests made *on that slave* 
in the last 3 seconds. The master receives these reports from all of its slaves 
and consolidates them in real time.

The master controls the starting and stopping of *load generation* on the slaves.
The master cannot start/stop the locust process running on the slaves. This
means you need to create servers and start locust's processes yourself.

To start the master process:

    locust -f something.py --master

You must start the master process before the slaves. Then start the slaves:

    locust -f something.py --slave --master-host=<master-ip>

You should be able see the clients connect/disconnect in the master's logs.

### Random tips


##### Detecting whether you're the master or slave
It seems normal to use the same locust file an the master and slave (I've never
tried using a master locust file and a separat, different slave locust file. Even 
if it works, you lose the ability to run your tests in non-distributed mode.)

But sometimes I wanted to do certain things only on the master (like saving
a report to disk) and some things only on the slave (like doing some data prep
before starting load generation).

From what I can tell, locust doesn't provide a way to detect which mode you're
running in. What I did was check for the command line arguments:

    import sys
    
    def is_master():
        return '--master' in sys.argv

    def is_slave():
        return '--slave' in sys.argv
        
##### Grouping requests in the report
By default, locust will use `/resources/{uuid}` as the name that shows up
in the summary report. But if your the url has an id in it, you'll get a separate entry
in locust's report for each id. 

Instead, you can provide an explicit `name` argument for Locust to use in its report:

    @locust.task
    def do_get(self):
        self.client.get('/resources/{id}', name='/resources/UUID')

##### Manually marking requests as success/failure
Normally, locust automatically records a success or failure by looking at the
http status code. You can use a `with` block to override this:

    with self.client.post('/things', catch_response=True) as post_resp:
        if post_resp.json()['status'] == 'ACTIVE':
            post_resp.success()
        elif post_resp.json()['status'] == 'ERROR':
            post_resp.failure("Saw ERROR status")
        else:
            post_resp.failure("Unknown error")

##### Achieving precise request rates
You can compute the approximate request rate using the average wait time and
the total number of users. This will be inaccurate though, because it doesn't
take into account the time each user spends executing a task. This is a problem
when you have long running tasks (e.g. if you need to poll).

For example, consider a long running task like this:

    @task
    def do_sleep(self):
        time.sleep(60)

A locust user will enter this task and sleep for 60 seconds. If you have a small
number of users, chances are they'll all eventually get stuck sleeping in this task, 
unable to continue executing other tasks, so your request rates will
plummet. (Due to reasons, you should actually use gevent.sleep() instead of 
time.sleep(). See the gevent docs for more.)

Instead, you want the task to take as short a time as possible. To do this, you
can spawn a new greenlet that sleeps in the background:

    def _do_async_thing_handler(self):
        gevent.sleep(60)

    @task
    def do_async_thing(self):
        gevent.spawn(self._do_async_thing_handler)

This accomplishes two things:

1. Your task (do_async_thing) returns effectively immediately, and the
locust user can go on to do other things. This means your actual request
rate should much be closer to what you expect.
2. Your task's functionality (_do_async_thing_handler) continues running in
the background and will terminate on its own.

##### Polling for things asynchronously
You can use a similar pattern as above to poll for an active status. The only
thing to watch out for here is having too many greenlets running in the
background on a single box (you can lower the polling frequency or run in
distributed mode to solve this):

    def _do_async_thing_handler(self):
        # using `with` prevents locust from making an entry in its report
        with self.client.post('/things', catch_response=True) as post_resp:
            id = post_resp.json()['id']

            # Now poll for an ACTIVE status
            end_time = time.time() + timeout
            while time.time() < end_time:
                r = self.client.get('/things/' + id)
                if r.ok and r.json()['status'] == 'ACTIVE':
                    post_resp.success()
                    return
                elif r.ok and r.json()['status'] == 'ERROR':
                    post_resp.failure("Saw ERROR status")
                    return

                # IMPORTANT: make sure you yield to other greenlets by using
                # gevent.sleep. Otherwise, you'll block the world.
                gevent.sleep(1)
            post_resp.failure("Polling timed out")

    @task
    def do_async_thing(self):
        gevent.spawn(self._do_async_thing_handler)

The key things here are:

1. Spawning a new greenlet so your locust users don't block when running
your tasks, so your request rate is unaffected.
2. Using the `with` block to let us control the success condition
3. Using gevent.sleep in your polling loop to avoid blocking the world
4. Calling .success() or .failure() to mark the original request as a
success or failure. Locust will then compute and store the duration from the time
the request was made until the time the .success() function was called.

(There's another minor issue here... When you stop the test, any greenlets
running in the background will continue running. I tracked my greenlets in a
list and used one of locust's event hooks to kill them all when the test is stopped)




