# streamparse

streamparse lets you run Python code against real-time streams of data. It also
integrates Python smoothly with Apache Storm.

It can be viewed as a more robust alternative to Python worker-and-queue
systems, as might be built atop frameworks like Celery and RQ. It offers a way
to do "real-time map/reduce style computation" against live streams of data. It
can also be a powerful way to scale long-running, highly parallel Python
processes in production.

## WARNING: IN DEVELOPMENT

streamparse is still in active development. It is not ready for production use.
It isn't even really ready for development use. Follow the project's progress
via our Google Group, [streamparse@googlegroups.com][google-group].

[google-group]: https://groups.google.com/forum/#!forum/streamparse

You can also reach out to [@amontalenti](https://twitter.com/amontalenti),
[@msukmanowsky](https://twitter.com/msukmanowsky) or
[@kbourgoin](https://twitter.com/kbourgoin) on Twitter.

## Demo Video

![demo video gif](https://raw.githubusercontent.com/Parsely/streamparse/master/doc/source/images/quickstart.gif)

also check out the demo of a [real-time redis wordcount topology][redis]

[redis]: https://raw.githubusercontent.com/Parsely/streamparse/master/doc/source/images/redis.gif

## Dependencies

### Java and Clojure

To run local and remote computation clusters, streamparse relies upon a JVM
technology called Apache Storm. The integration with this technology is
lightweight, and for the most part, you don't need to think about it.

However, to get the library running, you'll need (1) JDK 7+, which you can
install with apt-get, homebrew, or an installler; and (2) lein, which
you can install from the [project's page][lein-proj] or [Github][lein-github].

[lein-proj]: http://leiningen.org/
[lein-github]: https://github.com/technomancy/leiningen#leiningen

### Python dependencies

* fabric: used for remote SSH server management
* invoke: used as a local shell-based build tool

You don't actually need fabric and invoke installed separately from
streamparse; it will be installed automatically when you install the
``streamparse`` module.  However, fabric and invoke provide mechanisms for you
to extend your streamparse projects with custom build and server management
steps.

# Getting started

## Installation

Confirm that you have ``lein`` installed by running:

    lein version

You should get output similar to this:

    Leiningen 2.3.4 on Java 1.7.0_55 Java HotSpot(TM) 64-Bit Server VM

If ``lein`` isn't installed, [follow these directions][lein-install].

[lein-install]: http://leiningen.org/#install

Once that's all set, you can run:

    pip install streamparse

This will offer a command-line tool, ``sparse``. Use:

    sparse quickstart wordcount

To create a project template in the ``wordcount`` directory,
which will have this structure:

* src/
    * words.py: example Spout in Python (stream of words)
    * wordcount.py: example Bolt in Python (word count)
* topologies/
    * wordcount.clj: ``clj`` file with topology configuration (Clojure DSL)
* config.json: config file w/ Storm cluster hostnames and code locations
* project.clj: ``lein`` project file to express Storm dependencies
* fabfile.py: remote management tasks (fabric, customizable)
* tasks.py: local management tasks (invoke, customizable)

## Running locally

You can then run the local sample word count topology using:

    cd wordcount
    sparse run

This will produce a lot of output and may also download Storm dependencies upon
first run.

## Submitting

*Note*: Beta support.

Before submitting your streamparse project, you need to configure at least one
remote environment in your `config.json` file like so:

```json
{
    "library": "",
    "topology_specs": "topologies/",
    "virtualenv_specs": "virtualenvs/",
    "envs": {
        "beta": {
            "user": "user_with_ssh_access_to_all_servers",
            "nimbus": "nimbus.example.com:6627",
            "workers": [
                "storm-worker1.example.com",
                "storm-worker2.example.com",
                "storm-worker3.example.com",
                "storm-worker4.example.com"
            ],
            "log_path": "/path/to/logging",
            "virtualenv_path": "/path/to/virtualenvs"
        }
    }
}
```

A few important notes about the `user` you specify for each of the
environments in the `envs` key:

* The user must have ssh access to the servers specified in the `nimbus` and `workers` keys.
* The user must have write access to the `virtualenv_path` directory on the `workers` servers.

If you have only one topology defined in `topologies/` and one environment
defined in your `config.json`, you can submit your topology via:

    sparse submit

If you have more than one topology, you'll have to specify your topology like
so:

    sparse submit --name <topology_name>

If you have more than one environment defined in `config.json`, you'll have to
specify the environment like so:

    sparse submit --environment <environment_name>

You can use both options together to submit a specific topology to a specific
environment.

    sparse submit --environment <environment_name> --name <topology_name>

## Managing

*Note*: Beta support.

To kill a running Storm topology, use:

    sparse kill --environment=prod

## Tail log files

*Note*: Beta support.

To tail all the log files for a running topology across a production Storm
cluster, use:

    sparse tail --environment=prod

## Debugging locally

*Note*: Not yet implemented.

You can debug a local topology's Spout by running:

    sparse debug --spout=wordcount.sample_spout

This will set a breakpoint when the Spout receives its first data tuple and let you trace through it.

You can debug a local Storm topology's Bolt by running:

    sparse debug --bolt=wordcount.sample_bolt

This will set a breakpoint when the Bolt receives its first data tuple.

In both cases, debug uses ``pdb`` over a socket connection.

Topologies are automatically killed when you re-submit an existing topology to
a cluster.

# What is Storm?

Storm is a distributed real-time computation framework. Storm is sometimes
referred to as a "real-time map/reduce implementation". It allows you to define
computation as a directed acyclic graph (DAG) of processing nodes, called
Bolts. Bolts take a stream of named tuples as input. They produce one or more
streams of named tuples as output. Dependencies between Bolts are explicitly
declared. Data originates in a cluster via Spout, which simply exposes a stream
of named tuples. A Spout receives its source data from a high-performance queue
like Apache Kafka (though ZeroMQ, RabbitMQ, and other sources are also
options).

## Why should a Pythonista care about Storm?

In short, the Spout and Bolts abstraction allows you to write Python code which
transforms a live stream of data and execute it performantly across a cluster
of machines. You can parallelize each step of your computation automatically,
and you can resize your cluster dynamically to add more processing power. Storm
also provides mechanisms to guarantee fault-tolerance and at-least-once message
processing semantics. It is a strong alternative to Celery for log and stream
processing problems.

## streamparse, Clojure, and lein

streamparse allows you to write your Storm Spout and Bolt implementations in
pure Python. It also provides mechanisms for managing and debugging your
clusters.

But Storm is actually a language-independent technology. It is written in Java
and Clojure and runs on the JVM, but works with other programming languages via
its "multi-lang protocol". As a result of this flexibility, streamparse leverages
Storm's existing Clojure DSL for configuring Storm topologies.

This allows you to freely mix Python Spouts and Bolts with Java/Clojure Spouts
and Bolts (as well as Spouts and Bolts written in other languages altogether).
This is important because the community around Storm has written many re-usable
components in Java/Clojure. For example, there are several data integration
Spouts already written for tools like Kafka, RabbitMQ, ZeroMQ, MongoDB and
Cassandra. Therefore, every streamparse project is actually a mixed Python and
Clojure project.

The ``lein`` build tool is used to resolve dependencies to Storm itself,
install it locally, run Storm topologies locally, add dependencies for
community-supported JVM-based Spout implementations, offer an interactive
debugging REPL, and to package your topologies as an "uberjar" for submission
to a production Storm cluster. All of this is scripted under the hood using a
small command-line Clojure program that is included with streamparse.

You may be worried that this means to use streamparse, you need to know
Clojure.  This is not the case. The Clojure DSL is essentially just a
lightweight configuration file format that happens to be written in Clojure.

It isn't any harder than JSON since it's ultimately just configured using some
data maps.  Plus, we have plenty of examples for you to follow. And, we've
provided a simple tool for introspecting your Python Spouts and Bolts and
offering starting points for configuration.

## Roadmap

See the [Roadmap][roadmap].

[roadmap]: https://github.com/Parsely/streamparse/wiki/Roadmap
