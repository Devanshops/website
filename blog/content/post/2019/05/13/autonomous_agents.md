---
title: "Autonomous Agents"
date: 2019-05-13T13:00:00+01:00
tags: ["machines"]
draft: false
---

Today we're launching a significant new feature that allow you to create a kind of automation that run on your nodes and do not need RPC interactions to initiate actions. We call it *Choria Autonomous Agents* or *Choria Machine*, it's available as a preview feature in *Choria Server 0.11.0*

These run forever and continuously interact with your node, they keep working if the node is disconnected from the middleware and do not require a central component to function.

This release is a feature preview release, there are significant shortcomings and missing features but it's already functional. We are launching this feature very early to solicit feedback and ideas to help us prioritize future work.

## Overview

The typical orchestrations that people have done with MCollective or Choria has always taken the form of a conductor that tells the fleet what to do every step of the way.

This works fine for a lot of things especially if you use features like Sub Collectives to create isolated network-near groups where you run a daemon that orchestrates just the little cluster. It's work so well in fact that this has always just been acceptable.

Unfortunately there are number of draw backs to this:

 * It requires a lot of network traffic as one entity communicate constantly with the fleet
 * It does not scale to complex tasks
 * The orchestrator, network and brokers are all single point of failures, any failure anywhere means the managed component is not managed anymore
 * The central orchestrator can get very complex as it might need to maintain lots of state for every node

Mark Burgess has a little anecdote about this, the Mayor of a city does not constantly tell every street sweeper where and how to do their job, the sweepers are trained to do their thing on their own and so a city scales by applying this concept on every level.

For years I have tried to build some form of autonomous agent that let us describe a system being managed and it will constantly be managed. The conceptual component is a [Finite State Machine](https://en.wikipedia.org/wiki/Finite-state_machine) - nothing new about this - but I always had concerns about visibility and operability. Recent advances in tools like Prometheus but also my own work in events from the Choria daemons have made this much more viable.

I think of this a bit like a Kubernetes Operator but for anything in any environment.

<!--more-->
## Use Cases

There are many typical use cases for such an Autonomous Agent, you might already have something like this in your container scheduler or even the thermostat control of your HVAC system at home.

Any thing that has a deterministic set of states with defined transitions between those states and some way to interact with the managed systems using any language / script / tool would be enough.

We want to make it truly portable so for calling into other systems we support, today, only a exec based method that uses the typical Unix 0 or non 0 exit code as a API.

## Run Modes

These Machines run within the Choria Server continuously, when it starts it reads a configured directory and start all machines there - it will also periodically check again starting new ones, restarting changed ones or killing removed ones.

They emit a stream of events over the middleware in JSON format (with schemas) and we provide a little CLI tool to watch these events - but since they are JSON you can program something to interact with them too.

For development you can run them locally without needing middleware connections or anything, events are logged to STDOUT.

## Example

I'll put a quick example here but the [documentation has a fully worked example](https://choria.io/docs/autoagents/example/).

The example below will read *manifest.json* that might look like this:

```json
{
    "image": "registry.example.net/ops/example",
    "tag": "v1"
}
```

When this file changes or the running container does not match what is in the file it will deploy that container at that version, if ever the JSON changes or the container crashes it will be redeployed to the desired version.

Today these specifications are done in YAML but we hope to support some other language in the future - perhaps HCL or Puppet or something. A [JSON Schema](https://choria.io/schemas/choria/machine/v1/manifest.json) describes the YAML format and if you use VS Code - and probably others - you can configure your editor to associate this schema with *machine.yaml* files to get validation, code completion and context specific snippets.

You can see here integration with the system is via calling external commands - you could call Ansible, Bolt, your own scripts or anything else here as long as they have meaningful 0 or 1 exit codes.

```yaml
name: ContainerManager
version: 1.0.0
initial_state: unknown

# How the machine move between states, also declares the valid states: unknown, monitor, deploy
transitions:
 - name: absent
   from: [unknown, monitor, deploy]
   destination: deploy

 - name: deployed
   from: [deploy]
   destination: monitor

 - name: deploy_failed
   from: [deploy]
   destination: deploy

 - name: no_manifest
   from: [unknown, monitor, deploy]
   destination: unknown

# what to do when in a state and when transitioning to a state
watchers:
  - name: manifest
    type: file
    fail_transition: no_manifest
    success_transition: absent
    interval: 10s
    state_match: [unknown, deploy, monitor]
    properties:
      path: manifest.json

  - name: deploy
    type: exec
    state_match: [deploy]
    interval: 1m
    fail_transition: deploy_failed
    success_transition: deployed
    properties:
      command: ./deploy.rb

  - name: check_running
    type: exec
    state_match: [monitor]
    interval: 30s
    fail_transition: absent
    properties:
      command: ./check_running.rb
```

This will run forever and manage the container, you can observe it working on the CLI or programmatically - here we killed the running container and it was remediated:

```nohighlight
$ choria machine watch
INFO[0000] Viewing transitions on topic choria.machine.transition
INFO[0000] Viewing watcher states on topic choria.machine.watcher.*.state
example.net ContainerManager#check_running command: ./check_running.rb, previous: error ran: 0.072s
example.net ContainerManager transitioned via event absent: monitor => deploy
example.net ContainerManager#deploy command: ./deploy.rb, previous: success ran: 0.513s
example.net ContainerManager transitioned via event deployed: deploy => monitor
example.net ContainerManager#check_running command: ./check_running.rb, previous: success ran: 0.153s
```

## Roadmap

There are major obvious things left out, in no particular order:

 * Nodes should be able to download machines onto them via HTTP(S)
 * A dashboard should be made that can observe any machine - showing executions, times, transitions etc. I consider this a major current short coming.
 * More fully worked examples that you can run out of the box
 * Concurrency control so that multiple machines can share a mutex - for example 10 machines managing different packages need to coordinate around access to yum
 * Additional watchers - consul, etcd, vault etc
 * Ability to specify arbitrary environment variables to exec watchers
 * Patterns where some state creates something that takes time to become manageable, others have to somehow wait. Create a compute node and somehow have to know once Puppet has provisioned it.

## More information

This is very early days for this feature, I am keen to get feedback from the community on use cases they have, what they want to manage and more so we can iterate on this. Today we support only 2 integrations - files and exec.

I wrote [quite extensive documentation](https://choria.io/docs/autoagents/) about this but expect things to change a bit as we iteration on this and add more features.

This was built as issue [#563](https://github.com/choria-io/go-choria/issues/563) where you can see some outstanding items etc.
