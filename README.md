# Service Registry #

This is a simple service registry that can be used in a distributed
service oriented architecture.

It is designed to be simple, scalable and fault tolerant.

What it is:

* simple
* eventually consistent
* scalable

What it is not:

* a coordination service


## The API

Through this section we refer to the service registering its instance
as "the client".


### Registering

Each service instance that wants to make its presence known should
update its instance entry every 30th second.  This is done via PUT
request to a path that follows this format: `/<service>/<id>` with
content like this:

    {
      "site": "eu-1b",
      "host": "ec2-NN-NN-NN-NN.compute-1.amazonaws.com",
      "port": 1232,
      "updated_at": "<iso 8601 format>",
      "started_at": "<iso 8601 format>",
      "annotations": {
        "version": "1.2",
      }
    }

The `updated_at` field should contain a timestamp of when the last PUT
request was performed.  The client is responsible for setting this
field.  Users of the registry can use this feel to detect instances
that are not behaving correctly.  `started_at` should be an estimate
of when the instance was started.

The update request that the service instance do every 30th second also
acts as a heartbeat signal to the registry.  If it has not received a
request for T seconds it will assume the instance is dead and will
remove it from the registry.


### Query Service Registry

The most common query is look up all instances of a specific service.
This is done by issuing a GET to `/<service>`.  The result looks
something like this:

    {
      "<instance>": {
        "site": "eu-1b",
        "host": "ec2-NN-NN-NN-NN.compute-1.amazonaws.com",
        "port": 1232,
        "updated_at": "<iso 8601 format>",
        "started_at": "<iso 8601 format>",
        "annotations": {
          "version": "1.2",
        },
      },
      ...
    }

The result contains all registered instances for the service.  

## Internals

The instances of the service registry runs a gossip-like protocol to
exchange data.  All state is replicated to all peers through this
protocol.

If we look at the configuration:

    name: sr-sto-1
    cluster:
      sr-sto-1:
        host: sr-sto-1.sto.example.com
        port: 3222
      sr-ash-1:
        host: sr-ash-1.ash.example.com
        port: 3222
      sr-lon-1:
        host: sr-lon-1.lon.example.com
        port: 3222

The cluster is made out of three servers, `sr-sto-1`, `sr-ash-1` and
`sr-lon-1`.  The `name` field sets the local name and this should be
different for each instance of the service registry.  The `sr-sto-1`
instance will talk to `sr-ash-1` and `sr-lon-1` at regular intervals.
When `sr-sto-1` talks to `sr-lon-1` it will also receive state updates
for `sr-ash-1`.  This means that if there are network errors that
cause a partial network failure where two nodes cannot talk to
eachother they can still receive updates through the third peer.

Note that there's no requirement for having an odd number of instances.
