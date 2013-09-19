# Puppet Environments

In this proof-of-concept, we describe how dynamic Puppet environments can be levereged to achieve a combination of current community best practices for team workflows and rapid turnaround for single-user development and testing.
We will differentiate between *branch environments* that map to branches in the git repository, and static environments, such as *aliases* and *user environments*.  The term "static," is used loosely here, in the sense that these environments are configured manually, whereas branch environments are created and deleted automatically by a script.

Jump to:
- [Branch environments](#branch-environments)
- [Static environments](#static-environments)
  - [Aliases](#aliases)
  - [User environments](#user-environments)
- [Configuration](#configuration)

### <a id="branch-environments">Branch environments</a>

Branch environments are maintained by [r10k](https://github.com/adrienthebo/r10k) and correspond directly to git branches.  To use these, first push your commits to the central repository and specify `--environment=BRANCH` when running the agent. It may take a couple of seconds until **r10k** has created the environment. The update process is currently asynchronous because it relies on GitHub service hooks, but that could be avoided by installing a local [gitolite](http://gitolite.com/gitolite/) instance, which could then update Puppet masters synchronously.

Benefits:
- Branch environments lend themselves well to a work-flow based on "feature branches" that can be reviewed and merged independently.
- Branch environments can represent ongoing developments and can easily be worked on cooperatively.
- Long-lived branches could represent staging environments, or entirely different sites.

Restrictions:
- Branch names are restricted to alphanumeric characters and underscores due to [environment name restrictions](http://docs.puppetlabs.com/guides/environment.html).
- Static environments take precedence over branch environments in our setup.

### <a id="static-environments">Static environments</a>

Static environments, in contrast to the branch environments, are created by the system administrator and correspond very much to the static configuration variant in `puppet.conf` (defining each environment in its own section).  We can further distinguish at least two subtypes of static environments: aliases and user environments.

#### <a id="aliases">Aliases</a>

There is at least one environment alias called `production`, pointing to the `master` branch environment.  This environment has to exist because we don't want to name our default branch `production`, but that is the default environment name for both the Puppet master as well as the Puppet agent.

#### <a id="user-environments">User environments</a>

User environments must be maintained by the users themselves in `~/puppet` on any Puppet master.  Home directories are mounted via **glusterfs** and are thus shared between all masters.  To use a user environment, specify `--environment=USERNAME` when running the agent.

Our setup supports "sparse" user environments.  User environments don't need to have the full set of modules installed, or any modules at all.  If a module is missing in a user environment, it will be loaded instead from the `production` environment.

The advantage over branch environments is that changes can be tested "on the fly", without the need to commit and push them first.  Another plus is that even users without push access, but with an account on the Puppet master, can test their changes against nodes under their control and then submit a pull request.  The downside is that all work must happen in a directory that is accessible to all Puppet masters.

### <a id="configuration">Configuration</a>

Puppet [environments](http://docs.puppetlabs.com/guides/environment.html#what-an-environment-is) can be defined either statically, with a configuration section named after the environment (e.g., `[production]`), or dynamically in the `[master]` section, by setting at least `modulepath` and `manifestdir` to a path that interpolates the `$environment` setting.

We only use the dynamic environment definition, combined with a script that populates the base directory for dynamic environments with symlinks to actual environment locations.

Excerpt from `puppet.conf` that enables dynamic environments:
```text
[master]
manifestdir=$confdir/environments/dynamic/$environment/manifests
modulepath=$confdir/environments/dynamic/$environment/modules:$confdir/environments/dynamic/$environment:$confdir/environments/dynamic/production/modules
```
Note the final entry in `modulepath`, which will cause modules to be loaded from the `production` environment as a fallback in case the actual dynamic environment doesn't have all the required modules.  This is to support "sparse" user environments as explained below under *Static environments*.

Here's what the directory tree in `$confdir/environments` might look like:
```text
environments
├── static
│   └── production -> ../branches/master
│   └── ustuehler -> /home/ustuehler/puppet
├── branches
│   └── master
└── dynamic
    ├── master -> ../branches/master
    ├── production -> ../static/production
    └── ustuehler -> ../static/ustuehler
```
The `r10k.yaml` configuration file:
```text
# The location to use for storing cached Git repos
:cachedir: '/etc/puppet/r10k/cache'

# A list of git repositories to create
:sources:
  # This will clone the git repository and instantiate an environment per
  # branch in /etc/puppet/environments/branches
  :infrastructure:
    remote: 'git@github.com:my-org/infrastructure'
    basedir: '/etc/puppet/environments/branches'

# This directory will be purged of any directory that doesn't map to a
# git branch
:purgedirs:
  - '/etc/puppet/environments/branches'
```
