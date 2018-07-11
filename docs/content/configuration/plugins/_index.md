+++
title = "Plugin Distribution"
toc = true
weight = 230
+++

MCollective have many plugins, the most common ones are _agents_.

In the past packaging was done via RPM or Deb packages, this was extremely limited requiring extra work to make configuration modules and of course only worked on those operating systems.

## Packaging

Choria includes a packager that turns a common MCollective plugin into a Puppet Module like the [Puppet Agent one](https://forge.puppet.com/choria/mcollective_agent_puppet).  It supports packaging plugins in the typical layout you might see in our official plugins, a good example is the [Puppet Agent](https://github.com/choria-plugins/puppet-agent) and others in the [Choria Plugins](https://github.com/choria-plugins/) organisation.

You can package your own modules in this manner:

```bash
$ cd youragent
$ mco plugin package --format aiomodulepackage --vendor yourco
```

This will produce a Puppet module that you can install using Choria by adding it to the list of plugins to manage:

```yaml
mcollective::plugin_classes:
  - mcollective_agent_youragent
```

## Additional Files

You can add additional files into the resulting module by creating a *puppet* directory within your repository.  These files will be copied into the final module after any templates and README files were generated.

Use this to add additional files like functions, tasks and plans into your modules.  You can also override any generated files in this manner. For example the [Puppet Agent](https://github.com/choria-plugins/puppet-agent/tree/master/puppet) has a number of Plans included.

## Plugin Configuration

Many MCollective plugins have extensive configuration, some times Server and Client side.

The _choria-mcollective_ module lets you configure any setting in any plugin via _Hiera_ data, here's an example of configuring the Puppet one:

```yaml
mcollective_agent_puppet::config:
  "splay": false
  "signal_daemon": true
```

This creates files in `/etc/puppetlabs/mcollective/plugin.d` with the per plugin settings.  This will only work on plugins distributed using the method shown above.

You can ship your own custom configuration items in the Plugin directory that gets mixed into the final default data, this can be used to add for example default Action Policies that allow read-only aciton.

Placing this in the `.plugin.yaml` of the top directory of the Puppet Agent causes a default ACL where anyone can run the *last_run_summary* and *status* actions:

```yaml
mcollective_agent_puppet::policies:
  - action: "allow"
    callers: "*"
    actions: "last_run_summary status"
    facts: "*"
    classes: "*"
```

This can be used for any setting including config keys.
