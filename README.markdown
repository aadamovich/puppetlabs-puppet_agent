# puppet_agent

[![Build Status](https://travis-ci.org/puppetlabs/puppetlabs-puppet_agent.svg?branch=master)](https://travis-ci.org/puppetlabs/puppetlabs-puppet_agent)

#### Table of Contents

1. [Overview](#overview)
2. [Module Description - What the module does and why it is useful](#module-description)
3. [Setup - The basics of getting started with puppet_agent](#setup)
    * [What puppet_agent affects](#what-puppet_agent-affects)
    * [Setup requirements](#setup-requirements)
    * [Beginning with puppet_agent](#beginning-with-puppet_agent)
4. [Usage - Configuration options and additional functionality](#usage)
    * [Puppet 3 Upgrades](#puppet-3-upgrades)
    * [Puppet 4 Upgrades](#puppet-4-upgrades)
5. [Reference](#reference)
    * [Public classes](#public-classes)
    * [Private classes](#private-classes)
    * [Parameters](#parameters)
    * [Tasks](#tasks)
6. [Limitations - OS compatibility, etc.](#limitations)
    * [Known issues](#known-issues)
7. [Development - Guide for contributing to the module](#development)

## Overview

A module for upgrading Puppet agents. Supports upgrading from Puppet 4 puppet-agent packages to later versions.

## Module Description

The puppet_agent module installs the Puppet Collection 1 repo (as a default, and on systems that support repositories); migrates configuration required by Puppet to new locations used by puppet-agent; and installs the puppet-agent package, removing the previous Puppet installation. When starting from Puppet 3, it will upgrade to the latest Puppet open-source release, or to the latest puppet-agent package supported by your PE installation.

If a package_version parameter is provided, it will ensure that puppet-agent version is installed. The package_version parameter is required to perform upgrades starting from a puppet-agent (Puppet 4) package.

This module expects Puppet to be installed from packages.

## Setup

### What puppet_agent affects

* Puppet, Facter, Hiera, and MCollective.
* Puppet's SSL directory and puppet.conf.
* MCollective's server.cfg and client.cfg.
* Removes deprecated settings from puppet.conf.
* Updates puppet.conf and server.cfg for behavioral changes in puppet-agent (future parser is now the default, and MCollective has a new varlog location).

### Setup requirements

Your agents must be running Puppet 3 with `stringify_facts` set to 'false', or Puppet 4+. Agents should already be pointed at a master running Puppet Server 2.1 or greater, and thus successfully applying catalogs compiled with the Puppet 4 language.

To compile this module, you must use Puppet 3.7 with future parser or newer, meaning it can be applied to masterless Puppet 3.7 or newer, or earlier Puppet 3 agents connecting to a Puppet 3.7 or newer master.

#### Configuring `stringify_facts`

On systems running Puppet 3.x, you can configure the `stringify_facts` settings with either a dedicated Puppet class or the [`puppet_conf`](https://forge.puppet.com/puppetlabs/puppet_conf) task module.

For example, this class can configure the settings:

```puppet
include ::puppet_agent::prepare::stringify_facts
```

This [Puppet Enterprise task](https://puppet.com/docs/pe/2017.3/orchestrator/puppet_tasks_overview.html) can also configure the settings:

```bash
puppet task run puppet_conf action=set section=main setting=stringify_facts value=false --nodes example-38-box.vm
```

This [bolt task](https://puppet.com/docs/bolt/0.x/bolt.html) can also configure the settings over SSH/WinRM:

```bash
bolt task run puppet_conf action=set section=main setting=stringify_facts value=false --nodes example-38-box.vm
```

> *Warning:* `stringify_facts` was [deprecated in Puppet 3.8](https://docs.puppet.com/puppet/3.8/deprecated_settings.html#stringifyfacts--true) and [removed in Puppet 5](https://puppet.com/docs/puppet/5.3/upgrade_major_pre.html#stop-stringifying-facts-and-check-for-breakage).

### Beginning with puppet_agent

Install the puppet_agent module with `puppet module install puppetlabs-puppet_agent`.

## Usage

### Puppet 3 Upgrades

Add the class to agents you want to upgrade:

~~~puppet
include ::puppet_agent
~~~

This installs the latest released version of Puppet from Puppet Collection 1.

To upgrade with this module, first stand up a Puppet Server 2.1 master---which supports backward compatibility with Puppet 3 agents---and point the agent you want to upgrade at that master. Once you've confirmed the agent runs successfully against the new master, and thus the Puppet 4 language, apply the class to the agent and confirm that it checks back in after a successful upgrade. Further details on upgrading are available [here](http://docs.puppetlabs.com/puppet/4.2/reference/upgrade_major_pre.html).

As part of preparing the agent for Puppet 4, the module performs several significant steps:
* Copies SSL files (based on their location settings: ssldir, certdir, privatedir, privatekeydir, publickeydir, requestdir) to new Puppet 4 defaults, and restore those settings to default in puppet.conf.
* Resets non-deprecated settings to defaults: disable_warnings, vardir, rundir, libdir, confdir, ssldir, and classfile.
* Resets logfile in MCollective's server.cfg and client.cfg.
* Adds new libdir and plugin.yaml locations to MCollective's server.cfg and client.cfg.

> **Note:** The upgrade does not change several config options. Anything else that's been explicitly configured will not be changed to reflect new default locations in Puppet 4. Some of these options are:
* Puppet's logdir
* MCollective's logfile

### Puppet 4 Upgrades

Add the class to agents you want to upgrade, specifying the desired puppet-agent version:

~~~puppet
class {'::puppet_agent':
  package_version => '1.4.0',
}
~~~

This will ensure the version `1.4.0` of the puppet-agent package is installed. For version `1.4.0` and later, it will also remove the deprecated `pluginsync` setting from `puppet.conf`, unless explicitly managed elsewhere.

## Reference

### Public classes
* [`puppet_agent`](#class-puppetagent)

### Private classes
* `puppet_agent::install`: Installs packages.
* `puppet_agent::install::remove_packages`: For platforms that can't perform in-place upgrades, removes the old packages.
* `puppet_agent::install::remove_packages_osx`: Removes the old packages on Mac OS X.
* `puppet_agent::osfamily::*`: Platform-specific preparation performed before upgrades.
* `puppet_agent::prepare`: Prepares the agent for upgrade.
* `puppet_agent::prepare::package`: Stages packages locally for install, on platforms that can't install from remote packages.
* `puppet_agent::prepare::*`: Prepare various file and ssl configuration.
* `puppet_agent::service`: Ensures the services are running.
* `puppet_agent::windows::install`: Handles Windows package installation.

### Parameters

#### Class: puppet_agent

##### `arch`

The architecture version you wish to install. Defaults to `$::architecture`. This parameter is [ignored](#known-issues) in Windows Server 2003.

##### `collection`

The Puppet Collection to track. Defaults to `PC1`.

##### `is_pe`

Install from Puppet Enterprise repos. Enabled if communicating with a PE master.

##### `manage_repo`

Boolean to determine whether to configure zypper/yum/apt/solaris repositories. Defaults to `true`.
If set to false, it is assumed an internally hosted repository will be used for the installation,
and the native package providers will be used to query pre-configured repos on the host being upgraded.

##### `package_name`

The package to upgrade to, i.e., `puppet-agent`. Currently, the default and only accepted value is `puppet-agent`.

##### `package_version`

The package version to upgrade to. When upgrading from Puppet < 4.0, defaults to the puppet master's latest supported version
if compiled with A PE master or undef otherwise (meaning get the latest Open Source release). Explicitly specify a version to
upgrade from puppet-agent packages (implying Puppet >= 4.0).

##### `service_names`

An array of services to start, normally `puppet` and `mcollective`. If the array is empty, no services are started.

##### `source`

Alternate source from which you wish to download the latest version of Puppet. On the Windows operating system this is the absolute path to the MSI file to install, for example:
``` puppet
  source => 'C:/packages/puppet-agent-1.7.0-x64.msi'
```

##### `install_dir`

The directory the puppet agent should be installed to. This is only applicable for Windows operating systems and when upgrading the agent to a new version; it will not cause re-installation of the same version to a new location. This must use backslashes for the path separator, and be an absolute path, for example:
``` puppet
  install_dir => 'D:\Program Files\Puppet Labs'
```

##### `install_options`

An array of additional options to pass when installing puppet-agent. Each option in the array can be either a string or a hash. Each option is automatically quoted when passed to the install command.

With Windows packages, note that file paths in `install_options` must use backslashes. (Since install options are passed directly to the installation command, forward slashes aren't automatically converted like they are in `file` resources.) Backslashes in double-quoted strings _must_ be escaped, while backslashes in single-quoted strings _can_ be escaped. The default value for Windows packages is `REINSTALLMODE="maus"`.

``` puppet
  install_options => ['PUPPET_AGENT_ACCOUNT_DOMAIN=ExampleCorp','PUPPET_AGENT_ACCOUNT_USER=bob','PUPPET_AGENT_ACCOUNT_PASSWORD=password']
```

##### `msi_move_locked_files`

This is only applicable for Windows operating systems. There may be instances where file locks cause unncessary service restarts.  By setting to true, the module will move files prior to installation that are known to cause file locks. By default this is set to false.

``` puppet
  msi_move_locked_files => true
```

### Tasks

#### `puppet_agent::version`

Checks for the version of puppet-agent package installed. Returns results as `{"version": "<ver>", "source": "<how version was
detected>"}`. If a version cannot be found, returns `{"version": null}`.

#### `puppet_agent::install`

Installs the puppet-agent package. Currently only supports Linux variants: Debian, Ubuntu, SLES, RHEL/CentOS/Fedora. A specific
package `version` can be specified; if not, will install or upgrade to the latest Puppet 5 version available.

**Note**: The `puppet_agent::install_shell` task requires the `facts::bash` implementation from the [facts](https://forge.puppet.com/puppetlabs/facts) module. Both the `puppet_agent` and `facts` modules are packaged with Bolt. For use outside of Bolt make sure the `facts` module is installed to the same `modules` directory as `puppet_agent`.

## Limitations

Mac OS X Open Source packages are currently not supported.

### Known issues

* In masterless environments, modules installed manually on individual agents cannot be found after upgrading to Puppet 4.x. You should reinstall these modules on the agents with `puppet module install`.

In addition, there are several known issues with Windows:

* To upgrade the agent by executing `puppet agent -t` interactively in a console, you must close the console and wait for the upgrade to finish before attempting to use the `puppet` command again.
* MSI installation failures do not produce any error. If the install fails, puppet_agent continues to be applied to the agent. If this happens, you'll need to examine the MSI log file to determine the failure's cause. You can find the location of the log file in the debug output from either a puppet apply or an agent run; the log file name follows the pattern `puppet-<timestamp>-installer.log`.
* On Windows Server 2003, only x86 is supported, and the `arch` parameter is ignored. If you try to force an upgrade to x64, Puppet installs the x86 version with no error message.
* On Windows Server 2003 with Puppet Enterprise, the default download location is unreachable. You can work around this issue by specifying an alternate download URL in the `source` parameter.

Specifically in the 1.2.0 Release:
* For Windows, you must trigger an agent run after upgrading so that Puppet can create the necessary directory structures.

## Development

Puppet, Inc. modules on the Puppet Forge are open projects, and community contributions are essential for keeping them great. We can’t access the huge number of platforms and myriad hardware, software, and deployment configurations that Puppet is intended to serve. We want to keep it as easy as possible to contribute changes so that our modules work in your environment. There are a few guidelines that we need contributors to follow so that we can have a chance of keeping on top of things. For more information, see our [module contribution guide.](https://docs.puppet.com/forge/contributing.html)

## Maintenance

See [MAINTAINERS](MAINTAINERS)

Tickets: https://tickets.puppetlabs.com/browse/MODULES
