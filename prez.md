class: center, middle

# Taskqueues go wild

![](http://www.davisbase.com/wp-content/uploads/2014/04/developer-crazy-shutterstock_10777099-300x200.jpg)

### (in web applications context)

12/10/2015 - Adrien Di Pasquale

.logo[
  ![](https://upload.wikimedia.org/wikipedia/commons/thumb/0/08/Drivy_logo.png/320px-Drivy_logo.png)
]


---
class: center, middle

# Intro

---
class: middle

.full[
  ![](https://drive.google.com/uc?id=0B0Rb_B4jRdUZRnd2Mzc5WHRWZjg)]

???

- asynchronous tasks / jobs 
- I/O latency outside 
- Broker = messaging queue = redis / 0MQ / RabbitMQ 
- CRON = Clock 
- Workers often execute the same code as the App.
- usecases examples : mails / big task scale workers


---

## My taskqueue is better than yours

- Celery / Sidekiq / Resque / MRQ / RQ / (Beanstalkd) ...


- Efficiency 

- Reliability

- Visibility

???

- efficiency = Fast with cheap workers
- reliability = no jobs lost / retries / auto-reboots ...
- visibility = live monitoring / debug tools / (tracebacks)

---

## Why so many problems ?

- single-threaded languages

- jobs concurrency

- third party dependencies

???

- same code in very different context web != workers
- concurrency is much higher that regular web dynos

---
class: middle, center

# Check your code 

![](https://drive.google.com/uc?id=0B0Rb_B4jRdUZUWdnZWJUc29SakE)

---

## Good Job :thumbsup:

- idempotent
- (re-entrant)
- "stateless"
- least args possible

???

- idempotent = can be safely run several times : *add example*
- re-entrant = can stop in the middle and be ran again in another process : *add example*
- stateless = you cannot expect your models to be in the same state as they were when you enqueued. you should check for it or pass the state in the args.

---

## Thread-safe jobs

class methods / variables (e.g. Resque Job)

```rb
class SomeJob
  def self.perform(some_param)
    SomeClass.some_var = some_par
    SomeClass.do_something
  end
end
```

- when ran in parralel, `some_var` will be the same process-wise

- better :

```rb
class SomeJob
  def self.perform(some_param)
    some_class = SomeClass.new(leak_name)
    some_class.do_something
  end
end
```

???

*diagram of parallel run ?*

---

## Thread-safe jobs

mutable instance variables (e.g. MRQ Job)

```py
class SomeJob(Job):
  some_mutable = []
  def run(self, params):
    some_mutable.push(params["some_param"])
```

- when ran in parallel, `some_mutable` will be the same process-wise

- better :

```py
class SomeJob(Job):
  def run(self, params):
    some_mutable = []
    some_mutable.push(params["some_param"])
```

???

*diagram of parallel run ?*

---
class: center, middle

# My broker is a liar

![](https://drive.google.com/uc?id=0B0Rb_B4jRdUZSGI0SVJqS0l3SjQ)

---

## Contract breaches

- single job dequeued by 2 workers `BLPOP`
- ...

???

- read your broker specs / issues. or use redis.
- don't go too exoctic wild there are LOADS of very good messaging brokers out there. 

---

## Jobs loss

- RAM-based broker process crashed

=> set up periodic logging to disk

???

- RAM-based dbs (redis) mean you have to backup regularly if you don’t want to loose jobs.
- do not use RAM-based broker if you can't accept losing a few jobs once in a while

---


## Jobs discarding

- no more storage space

  - tracebacks pollution

  - too many arguments


- connection problems

=> monitor your broker machine / set up alerts on your SAAS provider 

???

- there are good SAAS redis providers out there.

---
class: center, middle

# Runtime Problems
*it works on my machine*

---

## Queue like a boss

.full[
  ![workers different configs](https://drive.google.com/uc?id=0B0Rb_B4jRdUZX2tCYkd5NFlDWEE)
]

???
  
  - worker with different configs  : 
  - 2gb ram + 4 threads 
  - 256mb ram + 100 threads

---

## Queue like a boss

.full[
  ![workers + queues](https://drive.google.com/uc?id=0B0Rb_B4jRdUZSVd6MHZ1YXA5aFk)
]

???

  - optimal config = least workers to dequeue everything in time
  
  - There is no one-size-fits-all solution for optimizing this. You'll have to iterate and find out what works best for your jobs with your specific workers. And you'll have to change it over time as you update jobs and their mean runtime evolves independently. Monitoring is king.

  - Queues named after the bit of logic they handle are *generally* a bad idea though. 
  - It's more scalable to group jobs in queues by their properties : mean runtime, acceptable dequeuing delay.

---

## <strike>Avoid</strike> Anticipate congestions

Bad day scenario :
- Job A becomes 10x slower than before 
- Job B keeps failing and delay retries in cascade
- Worker can't handle everything anymore


- Jobs slowdowns will happen. Be prepared
- monitor


???

  A resource (DB, 3rd party) may be slow / unavailable. or a code update side effect.
  You need to be able to scale lots of worker quickly. and know your limits so you don't overload a resource / consume all the network. 
  monitor so you notice it before the broker implodes / the end users get angry.

---

## Workers crash

- Memory exceeded

  - (Heroku R14s are NOT monitored by error trackers)


- auto restart is crucial

???

  

---

## Clock (CRON) crash

- Undetected syntax errors

- Runtime error

=> monitor it’s status (ping it) and effect (heartbeat)

???

  Syntax errors can go undetected : clock is rarely ran in development and even less covered by tests.
  Runtime error can easily go undetected as we often run the clock as a background process of a proper worker.

---

## Exceptions & retries

- Jobs **will** fail

- => Retry strategy
- => Track exceptions with external service. 

???

  Different jobs may have different retry strategies. 
  All I/O calls (especially HTTP) should be expected to fail as a regular behaviour. 
  You should retry these jobs several times before considering them as properly failed. 
  You can implement increasing retries delays to handle temporarily unavailabale resources

  external service not very clear

---

## Connections problems

![]()

- DB overload
- DB max connection outreached
  - (provider limit || low ulimit)
- slave dbs

- => Use connection pools 
- => Know your limits

???

  A worker typically hits the DB way harder than a regular web process. Try and estimate how hard before scaling your workers.
  Make sure your workers use connection pools, and are 'good citizens', releasing the unused ones, reconnecting on deconnections ... Dimension your pools according to your DB limits. 

---

# Conclusion

Sorry for using the word "monitor" <s>534</s> 535 times. 

Workers are cheap, don't over optimize unless you need to.

---

# resources

http://www.slideshare.net/bryanhelmig/task-queues-comorichweb-12962619

http://www.rockstarprogrammer.org/post/2008/oct/04/what-matters-asynchronous-job-queue/
