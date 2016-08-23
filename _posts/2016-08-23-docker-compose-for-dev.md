---
layout: post
title: Using docker-compose for webapp development
excerpt: At dubizzle we lean towards a service-oriented architecture whenever it makes sense, this means that when you're developing, you may need to run several server stacks. Some of us use docker-compose, a super-convenient way of coordinating containers and easily spinning up and taking down services on your development machine (and elsewhere). This guide will walk you through some of the thicker parts of setting up such an environment. In this tutorial we assume some basic familiarity with docker.
---
### Note: The original version of this blog post appeared on my former employer's [technical blog](http://blog.dubizzle.com/boilerroom/2016/08/18/setting-development-environment-docker-compose/)

## Intro

At dubizzle we lean towards a service-oriented architecture whenever it makes sense, this means that when you're developing, you may need to run several server stacks. Some of us use docker-compose, a super-convenient way of coordinating containers and easily spinning up and taking down services on your development machine (and elsewhere). This guide will walk you through some of the thicker parts of setting up such an environment. In this tutorial we assume some basic familiarity with docker.

## Before we begin
We assume you're running docker natively on linux for the most part (hereby referred to as 'your_docker_host'), however, docker-compose works just as well on OSX with docker-machine, including phenomenally smooth support for mounting volumes.

## My First docker-compose Stack

As a primer, we will first spin up a group of containers with dependencies between them. Something like this should do nicely:

```
version: '2'
services:
  webapp:
    image: <your_web_app_here>:latest
    ports:
      - "80:80"
    depends_on:
      - postgres
      - memcached

  memcached:
    image: memcached:latest

  postgres:
    image: postgres:9.3
```

A word of warning before we continue, the above configuration above would have no long-term persistency, I won't get in to persistency for the sake of brevity, that's a blog post on it's own, but two ways to do it are: data-only containers and named volumes (docker version > 1.9.0), though I haven't tried the latter myself yet.

So the above configuration, when you run `docker-compose up`, will be nice enough to make sure that both memcached and postgres are up and running before bringing up your webapp, older versions of docker-compose required that you explicitly declare every network dependency as well. More recent versions of docker-compose use an internal DNS mechanism that allows every container to talk to every other container, unless explicitly configured otherwise (see [docker networking documentation](https://docs.docker.com/compose/networking/)). Personally I have found that if you declare your really hard dependencies in `depends_on` (say, mainly backing stores), you'll generally be okay.

If I were to run something like `ping postgres` from within the `webapp` container, I'd expect a response at this point. The postgres and memcached containers are, by default configured to only expose their relevant ports within the docker-compose network. We have also explictly declared that the host's port 80 should be mapped to port 80 of our webapp. If i were to `curl your_docker_host:80`, I'd expect the webapp container to respond.

Make sure you understand what's happening above, as we'll be slowly evolving the above config throughout this tutorial.

## My Second, more useful, docker-compose Stack

I'll now throw in a few useful docker-compose tricks, while still fairly basic, are leaps ahead of the first stack.

```
version: '2'
services:
  webapp:
    image: <your_web_app_here>:latest
    ports:
      - "80:80"
    depends_on:
      - postgres
      - memcached
    volumes:
     - /path/to/my/code/on/my/dev/machine:/path/to/the/code/in/the/container
    env_file: env_vars/webapp_env_vars.env

  memcached:
    image: memcached:latest

  postgres:
    image: postgres:9.3
```

env_vars/webapp_env_vars.env:

```
DB_HOST=postgres
DB_PORT=5432
DB_USER=some_cool_dev
DB_PASSWORD=OMG_PLS_K33P_4_S3CRET
MEMCACHED_HOST=memcached
MEMCACHED_PORT=11211
SOME_VARIABLE_MY_WEBAPP_READS=a_value_i_use_to_configure_it

```

You should notice two new things here:
1) `webapp` declares an env_file from which to pick up env vars. This is important because we configure a lot of our applications with environment variables, so that the same docker image can run in different environments, taking inspiration from [12-factor apps](http://12factor.net/). The variables in this env file are loaded up into the container when it runs. It also means, in line with best practices, we can avoid checking in secrets and passwords in to our code and source control.

2) `webapp` also declares a volume. In this case we want to get some development done, so we mount our source code in the place the webapp container expects to serve it, this will vary depending on your setup so I only provide nonsense paths. But what it means is that making a change on your own machine will reflect inside the container, how cool is that!? It's super cool. Because we don't want to build a new image every time we want to test a code change.

## DOUBLE TIME!

You guessed it, it's time to fire up two, independent but dependent stacks. Let's say microservice `webappone` and microservice `webapptwo`. They will talk to each other, the outside web will also want to talk to them. You can already guess we have an issue here. Port 80. Port 80 is an issue... Solution? Toss in a Load Balancer/reverse proxy to handle port 80 and route requests based on hostnames.

```
version: '2'
services:
  haproxy:
    image: haproxy:1.6.4
    volumes:
      - ./conf/haproxy:/usr/local/etc/haproxy # assuming you have that folder structure on your own machine
    ports:
      - "80:80"
    depends_on:
      - webappone
      - webapptwo

  webappone:
    image: <your_web_app_one_here>:latest
    depends_on:
      - postgresone
      - memcachedone
    volumes:
      - /path/to/my/code/on/my/dev/machine/code_for_one:/path/to/the/code/in/the/container
    env_file: env_vars/webapp_one_env_vars.env

  memcachedone:
    image: memcached:latest

  postgresone:
    image: postgres:9.3

  webapptwo:
    image: <your_web_app_two_here>:latest
    depends_on:
      - postgrestwo
      - memcachedtwo
    volumes:
      - /path/to/my/code/on/my/dev/machine/code_for_two:/path/to/the/code/in/the/container
    env_file: env_vars/webapp_two_env_vars.env

  memcachedtwo:
    image: memcached:latest

  postgrestwo:
    image: postgres:9.3

```

env_vars/webapp_one_env_vars.env:

```
DB_HOST=postgresone
DB_PORT=5432
DB_USER=THE_BEST_ALPHA
DB_PASSWORD=OMG_PLS_K33P_4_S3CRET
MEMCACHED_HOST=memcachedone
MEMCACHED_PORT=11211
SOME_VARIABLE_MY_WEBAPP_READS=a_value_i_use_to_configure_it
```

env_vars/webapp_two_env_vars.env:

```
DB_HOST=postgrestwo
DB_PORT=5432
DB_USER=THE_BEST_BETA
DB_PASSWORD=OMG_PLS_K33P_4N0TH3R_S3CR3T
MEMCACHED_HOST=memcachedtwo
MEMCACHED_PORT=11211
SOME_VARIABLE_MY_WEBAPP_READS=a_value_i_use_to_configure_it
```

conf/haproxy/haproxy.cfg:

```
# This is a DEVELOPMENT haproxy configuration to be run inside a docker
# container, please refrain from thinking about using it in production.

global
        user haproxy
        group haproxy

defaults
        log     global
        mode    http
        option  httplog
        balance roundrobin
        contimeout 5000
        clitimeout 50000
        srvtimeout 50000

frontend http
        bind 0.0.0.0:80
        default_backend host_one # we feel like putting priority on webappone
        acl host_one hdr(host) -i one.something.local
        acl host_two hdr(host) -i two.something.local
        use_backend one if host_one
        use_backend two if host_two

# Reminder, docker-compose provides the hostnames via internal DNS, we'll explicitly
# specify the ports though because we feel like it. Look at how beautiful the
# uniformity is though. Every webapp exposes port 80. #NO #CONFLICT #NO #WAR
backend one
        server webapp_one webappone:80

backend two
        server webapp_two webapptwo:80
```

To explain: we have two webapps, each listening (in it's own container) on port 80, we now have the ENTIRE STACK being represented by HAProxy listening on port 80 and routing requests based on headers. Why HAProxy? because. you can toss in whatever reverse proxy you like, nginx or otherwise.

So a request lifecycle looks roughly like:

Browser hits up your_docker_host (because you put one.something.local and two.something.local in your hosts file pointing there (if dev)):
>hey, I'd like two.something.local, bro, please serve.

haproxy:
>oh wow, this guy wants two.something.local, I should totally hit up webapptwo:80 on my internal network to see what he/she says.

webapptwo:
>request has come in with host two.something.local, That's actually me! I should totes mcgotes serve that, what path did they want again, OH! Here it is!, RETURNIFY

Traverse in reverse and you have a response to the browser. Done.


## Bonus section: local, self-signed SSL.

So your website is mainly in SSL and you've ALWAYS had difficulty duplicating it locally. Fear no more. You can now serve your stack locally, with SSL to boot.

HAProxy supports SSL termination since 1.5.X, [see here](https://www.digitalocean.com/community/tutorials/how-to-implement-ssl-termination-with-haproxy-on-ubuntu-14-04). You'll notice in the previous config we mounted an haproxy config. We'll reuse this mount (not the cleanest config, but we're just demonstrating here) in order to also serve HTTPS traffic. We need to do 2 things:

1. start the haproxy container listening on port 443 (if you've gotten this far, you should know how to allow haproxy to listen on port 443)
2. push our self-signed certificate in to our haproxy container (as already stated, we've already mounted a volume, so we only need to add our .crt file in to that folder, next to the haproxy config)
3. tell our haproxy.cfg to start using this self signed certificate for termination, add: `bind 0.0.0.0:443 ssl crt /usr/local/etc/haproxy/self.pem`
to your `frontend http` block.

and there you have it. two webapps served via https with unified ports under a docker stack. Note that your webapps themselves will still receive regular HTTP traffic. Hence why it's called SSL termination


## Lessons learned the hard way
### docker-compose is quirky:

you may need to take things in to your own hands and completely `rm` containers if:

1. you change env vars
2. you change ports
3. you change just about anything else.

this has gotten better with more recent versions of docker-compose but sometimes just does not work right.

### A recommended alias
To avoid insanity, alias `docker-compose` to `dc`. seriously, save the keystrokes.

### Sometimes volumes linger
Not sure if this is still an issue, but you may find (especially with short-lived containers that have mounted volumes) that you quickly run out of disk space. learn how to clean up after docker. this is less of an issue recently but still something to keep in mind.

### Mount your code, but don't forget to restart your services in between changes
even if you mount your code and modify it, it does not mean the service has been restarted. restart it manually if you have to. Or, stop it and run a development server just like you'd do locally, advantages:

1. if you mess up anything in the container, recovery is a simple docker rm away.
2. you will no longer need to install all sorts of weird dependencies (that may or may not conflict between services) on your one precious development machine. you can keep up to date as requirements change without the hassle.[1]

### Go all in, or don't go in at all
You must go in all the way or not at all, do not attempt to half-way in to docker-compose for development. If a requirements file has changed causing you to need an image update, update the image in your config file, do not linger, do not constantly reinstall requirements manually. you will fall behind, you will fail.

Keep docker and docker-compose itself up-to-date, docker has developed insanely fast over the past few years, docker-compose even faster. They constantly introduce stuff that solve problems, unfortunately that also sometimes means retiring old methodologies and rewriting some configurations. Don't wait until it's totally broken at the worst possible time. My configurations have only gotten cleaner and more maintainable by doing this.

Do not give up quickly. docker-compose is an investment that will pay you back when you least expect it, if you've done it right, it will force you to think of production more often than you think. While mere mortals forget that there's an nginx config that behaves SLIGHTLY differently here or there, you will have faced and dealt with that strange nginx config in your day-to-day development activities. Your stack is pretty damn close to prod. hopefully. You can test and develop things that others may only be able to properly test on production or staging environments. Bask in the glory. Rub it in their faces.

### Build it for yourself, maintain it for imaginary others.
Even if it's just you working with docker-compose, build the configuration and check in your changes like it's an actual project. This means that others can pick up and improve on it if it piques their interest. "Oh hey, I heard you've got a magic system that bring up an entire development environment?" - yeah, go check the repo.

### Autoclustering caveats
Some software (such as ElasticSearch) have the ability to auto-cluster by default, if you have two services which both happen to have a backing service that does this, you'll want to prevent it from happening automatically, there are a few ways to prevent this:

1. Isolate networks
2. disable clustering in your config somehow, depends on the software
3. specify explicit cluster names (in the case of ElasticSearch) that will stop containers that should be independent from tossing data at each other.

### Consider templating docker-compose.yml itself
I personally have a template from which I generate docker-compose.yml files. This makes it easy to switch unnecessary services on and off, combine that with an haproxy config template and you've struck gold. A development machine that can run parts of the stack you're currently working on and turn off the rest. Consider a templating language like jinja2.


## Footnotes
[1] - Sometimes a webapp requires very specific version of libraries that if you want to install on a mac vs on linux is different. The single biggest advantage I've seen to using docker for development is not dealing with this. I just pull the container and go.
