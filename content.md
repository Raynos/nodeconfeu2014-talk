## Slide 1

Good morning guys, now I know this is the "frontend" track,
  but I'm afraid I won't be talking about any frontend.

Today we'll talk about how to deploy services into production
  and how to keep them running like a well oiled machine.

A bit of background on me, I'm Raynos on the internets. I work
  at Uber on the developer productivity team. It's our job to 
  make it easier for other developers to get shit done.

Today I'll talk about the tools and processes we use at Uber to
  make everyone's live easier. And because I'm all about small
  modules and open source, everything we will talk about today
  can be used by anybody.

## Slide 2

A bit of history, I joined uber about 10 months ago. On my first
  day I was given a "little" project to get my feet wet.

It was a simple statsd proxy that did some normalization,
  something you can write in a day or two.

That was pretty cool, up until the point my coworker Jeremy
  "ironfist" suurkivi asked me to deploy it to 50 machines and
  make sure it never goes down. He said just "productionize" it.

The trick is that ironfist thinks this is trivial, the code he
  writes just never goes down. He once wrote 500 lines of C and
  that process just never went down. Another time he hacked a 
  100 line coffeescript program together and ran it from his
  screen in production for 3 weeks, never went down.

Just "productionize" it.

Now for us mortals we have to do some work to keep our services
  up.

## Slide 3

How do you just "productionize" something.

Well first you need to add logging. How does logging help?

It gives you insight into what your program is doing. You will
  want to log warnings about weird things happening in your
  process so that if something goes wrong you can come back 
  to your application and see what happens.

We log to disk so you can identify what's happening on a
  particular machine and we log to kafka. Kafka is a shared
  queue that every machine can write to. This means you can 
  read from kafka once you apply your app to 10 servers.

Grepping kafka for errors just after a deploy is a really
  useful feature.

The logger also send errors to Sentry which we use for error
  reporting. Everything that reaches sentry is a bug and everyone
  on the team gets an email about it.

If you see new sentry errors after a deploy you know you've done
  an oops.

## Slide 4

Cool. the logger tells you when errors happen, and if you wanted
  to you can shift through hundreds of kilobytes of warnings.

But when you want an oversight of what your processes are doing
  you really want a dashboard and a monitoring system.

In comes statsd, it allows you to count and time things.

We use statsd to monitor everything that happens.

Generally we monitor occurances of warnings, so that we can
  be notified of a spike in warnings.

We monitor all traffic, including 400s and 500s by default for
  every service.

We also monitor a lot of custom information for our business objects
  and events so that we can be notified of any anamolies.

## Slide 5

We don't want logging and monitoring to be enabled heavily  
  when we are working locally.

We need a way to be able to configure how our applications work
  locally and in various production environments.

This is best done with a set of JSON files. We allow JSON files
  for local environment, production environment and for the
  various data centers we have. 

We then overwrite configuration using a CLI so that the
  infrastructure can manage where we log to or what port
  we start on.

We also use remote overwrites heavily, we have a service in
  place to be able to update configuration dynamically without
  restarting our applications, this allows us to do many things
  including debugging, A/B testing and toggling features.

## Slide 6

Next, let's talk about uncaught exceptions. Shit is going to go
  bad, you will deploy code with bugs.

And you want to know about them.

The three steps to dealing with uncaught exceptions are

Get them out of the process. The uncaught-exception module
  we used has two techniques, it will use the logger to write
  the error to all backends, we support asynchronous logging
  so we will know when the error is flushed.

However we also write the error to the backup file descriptor,
  we've had too many production exceptions getting swallowed
  because there we a bug in the logger.

Next we do a graceful shutdown, a node.js process has to go
  down on all unhandled thrown exceptions, no excuses, no
  special snowflakes, no "but I know better". However we do
  want to exit cleanly.

And finally we abort the process and we dump core. The core dump
  can then be used for later analysis or deleted.

## Slide 7

Every service you put into production, from day one, will need
  a control port.

A control port is something internal listening on localhost that
  you can talk to by sshing into the production machine and
  telling it to do things.

Currently the control port we open sourced has two features,
  the ability to turn CPU profiling on at run time and the
  ability to turn heapdumps on at runtime.

This is a really nice thing to just permanently have, because
  you when you want to debug a bug in production you are going
  to wish you had setup the CPU and heap instrumentation. If
  you try and add that instrumentation by redeploying you
  probably will not be able to reproduce the bug.

## Slide 8

The final part to productionizing a service is to take ownership
  and responsibility.

If you put something in production and it goes down, you wake
  up at 3am and resolve it within 10 minutes. If it's still
  unresolved you escalate and get help from managers and work on it
  until it's resolved.

We need tooling in place to ensure that if a failure happens
  that an engineer deals with it. We use nagios for this.

We've configured nagios to let us know if a health check fails.
  Note a health check does two things, it lets us know the
  http server is up and that important dependencies of the
  service are up.

We've configured nagios to monitor our graphite dashboards
  for any spikes worth telling us about.

We've configured nagios to tell pager duty to wake engineers up
  and if no engineers wake up then we wake up ALL the managers
  and directors.

## Slide 9

Making it easy to productionize an application is definetly one
  step towards building high quality services.

The other step is getting started in the first place. Really we
  want to build a new service and get it deployed in one day.

There is actually a lot involved, you have to scaffold out
  some default files, get a git project setup, get a wiki setup.

You have to setup CI, no-one likes configuring jenkins, you 
  have to make sure your tests work, that your code coverage works.

You have to make your application deployable. The way we deploy
  is by checking information about your service into puppet so
  that we can use puppet to deploy all services to a single EC2
  box.

Puppet is an arcane art, we really want to shield most developers
  from this complexity and have it just work for them.

## Slide 10

Potter is a modular tool that comes with almost no features
  out of the box.

It has two features, a modular help page you can plug into and
  a way to install plugins.

Internally we all install the potter base plugin. Which ships
  with the ability to create internal libraries and internal
  services.

What does it mean to create a new thing, potter is a staged
  workflow tool. The first stage is the local workflow.

Here we check for any naming conflicts, create all the files
  you will need for your project, create a local gitolite
  repository and ensure your tests pass locally.

This means run npm install and make sure your scaffold has 100%
  code coverage (we do code coverage at uber.)

This is all you need to work locally, your comfortably setup.

Most of the time you will go onto the local workflow step
  immediately but sometimes you just want to experiment with
  an idea locally in "secret".

## Slide 11

The next workflow is the shared workflow. Here we make interacting
  with our infrastructure at uber easier but it can interact
  with anything you want.

We push your code upto a git server whether thats an internal
  gitolite server or github.

We setup the documentation and wiki websites for you, whether
  that's a github README and a github wiki or an internal wiki
  website and an internal documentation website.

We setup CI for you, whether that's jenkins or travis. We ensure
  that we integrate the CI with your local workflow, if you
  make a pull request on github we start a build, if you make
  an arcanist differential internally we trigger a build.

Trust me, you only want one person to configure jenkins once and
  then never have to deal with that again forever.

## Slide 12

EC2 blabla

## Slide 13

production blabla

## Slide 14

write your own blabla

## Slide 15

thanks
