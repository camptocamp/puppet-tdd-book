# Set up your environment

In this series of blog posts, we will write a Puppet module that manages OpenLDAP client and OpenLDAP server. We will make this module compatible with both Debian 7 and RedHat 7. We will explain all the aspects of Puppet Agile development through the detailed examples of class, define, type, provider, function, and fact.

*Disclaimer: This blog posts are widely inspired by the wonderful [tutorial to write puppet-lint plugins](http://puppet-lint.com/developer/tutorial/), so you may notice some similarities in structures or sentences.*

## PREREQUISITES

* [Ruby](https://www.ruby-lang.org/) 1.9.3 (or above) and knowledge of the language
* [Bundler](http://bundler.io/)
* [Virtualbox](https://www.virtualbox.org/)
* [Vagrant](https://www.vagrantup.com/)
* Knowledge of the [rspec testing framework](http://rspec.info/)
* Knowledge of the [Beaker acceptance framework](http://github.com/puppetlabs/beaker)
* Knowledge of [serverspec](http://serverspec.org/)

## SETUP

First, get a skeleton project set up. The first thing you need to do is create a folder for your project. For convention’s sake, you should use `puppet-<something>`.

```shell
$ mkdir puppet-openldap
$ cd puppet-openldap
```

## README.MD
Every project needs a README file. It is common to use markdown to style its content.

## LICENSE
If you are not familiar with the various licenses commonly used in Open Source projects, visit « Choose A License » to have a look at some options. When you find one you are happy with, drop it in a file called LICENSE in the root of your project.

## metadata.json
You should add a file named metadata.json in the root of your project in order to store some metadata needed to publish your module on the [Puppet Forge](https://forge.puppetlabs.com/).

```json
{
  "name": "camptocamp-openldap",
  "version": "0.0.1",
  "author": "Camptocamp",
  "license": "Apache-2.0",
  "summary": "Puppet OpenLDAP module",
  "source": "https://github.com/camptocamp/puppet-openldap",
  "dependencies": [
    {
      "name": "puppetlabs/stdlib",
      "version_requirement": ">=3.2.0 <5.0.0"
    }
  ],
  "operatingsystem_support": [
    {
      "operatingsystem": "Debian",
      "operatingsystemrelease": [
        "7"
      ]
    },
    {
      "operatingsystem": "RedHat",
      "operatingsystemrelease": [
        "7"
      ]
    }
  ]
}
```

A few interesting lines:

### LINE 8 – 13
We will probably depend on `puppetlabs-stdlib` at some point in our project.

### LINE 14 – 27
Declare the operating system supported by your module; this information will be automatically used by `rspec-puppet-facts` to loop over any supported operating system for tests.

## CONFIGURE ACCEPTANCE TESTS
We will use Beaker to launch acceptance tests. [Beaker](https://github.com/puppetlabs/Beaker) is the Puppetlabs’s acceptance testing tool used to test the behavior of a Puppet module on different operating systems. It can run tests on various hypervisors, including virutalbox (through Vagrant), Docker or OpenStack. You can use Beaker to test [serverspec resources](http://serverspec.org/)).

## CONFIGURE Gemfile
Bundler is a dependency manager tool for Ruby projects. It reads its configuration from the project’s `Gemfile`.

```ruby
source 'https://rubygems.org'
 
gem 'puppet',       :require => false
gem 'beaker-rspec', :require => false
```

### LINE 1
Tells Bundler to fetch dependencies from RubyGems over HTTPS.

### LINE 3 – 4
Our project depends on puppet and beaker-rspec. Now that our Gemfile is in place, let’s tell bundler to install everything needed to write our module.

```shell
$ bundle install --path vendor/bundle
Fetching gem metadata from https://rubygems.org/..........
Resolving dependencies...
Installing CFPropertyList 2.3.0
Installing i18n 0.7.0
Installing json 1.8.2
…
Installing specinfra 2.16.0
Installing serverspec 2.9.1
Installing beaker-rspec 5.0.1
Using bundler 1.7.4
Your bundle is complete!
It was installed into ./vendor/bundle
```

## ADD NODE FILES
You need to add a node file for Beaker. Node files indicate the nodes that the tests will be run on. Create the folder where the nodesets will be stored:

```shell
$ mkdir -p spec/acceptance/nodesets
```

Now, we’ll create two nodesets to run acceptance tests on Docker.

One for Centos 7 64bits in `spec/acceptance/nodesets/centos-7.yml`:

```yaml
HOSTS:
  centos-7-x64:
    platform: el-7-x86_64
    hypervisor: docker
    image: centos:7
    docker_preserve_image: true
    docker_cmd: '["/usr/sbin/init"]'
CONFIG:
  type: aio
```
  
### LINE 3
Platform to use to setup Beaker

### LINE 4
Hypervisor to use ; here we’ll use Docker because we'll then be able to run acceptance tests on Travis CI.

### LINE 5
Docker image from [https://hub.docker.com](Docker Hub) box to use.

### LINE 6
Tell beaker to keep the Docker image for multiple spec runs. 

### LINE 7
Launch init process instead of sshd.

### LINE 9
Tells Beaker that we are using Puppet All-In-One installation (installation path differs from Puppet Enterprise)

And one for Debian 7 64bits in `spec/acceptance/nodesets/debian-7.yml`:

```yaml
HOSTS:
  debian-7-x64:
    platform: debian-7-amd64
    hypervisor: docker
    image: debian:7
    docker_preserve_image: true
    docker_cmd: '["/sbin/init"]'
CONFIG:
  type: aio
```

## CREATE SPEC_HELPER_ACCEPTANCE.RB

Next, you need to create a `spec/spec_helper_acceptance.rb` file to configure Beaker:

```ruby
require 'beaker-rspec'

install_puppet_agent_on hosts, {}

RSpec.configure do |c|
  module_root = File.expand_path(File.join(File.dirname(__FILE__), '..'))
  module_name = module_root.split('-').last

  # Readable test descriptions
  c.formatter = :documentation

  # Configure all nodes in nodeset
  c.before :suite do
    # Install module
    puppet_module_install(:source => module_root, :module_name => module_name)
    hosts.each do |host|
      on host, puppet('module','install','puppetlabs-stdlib'), { :acceptable_exit_codes => [0,1] }
    end
  end
end
```

### LINE 3
Install Puppet Agent (a.k.a. puppet AIO) on every host in our nodeset. Beaker supports multiple hosts in a nodeset. This functionality will not be explained here.

### LINE 15
Tells Beaker to scp our sourcedir into container's modulepath.

### LINE 16-18
Tells Beaker to install dependencies into container’s modulepath.

## CONFIGURE UNIT TESTS
We will use [rspec-puppet](https://github.com/rodjek/rspec-puppet) to run unit tests.

### CONFIGURE GEMFILE
The `puppetlabs_spec_helper` gem provides some helpers to ease rspec-puppet configuration. Let’s add it to our `Gemfile`:


```ruby
source 'https://rubygems.org'
 
gem 'puppet',                 :require => false
gem 'beaker-rspec',           :require => false
gem 'puppetlabs_spec_helper', :require => false
```

### CONFIGURE .fixtures.yml
The `.fixtures.yml` file is a YAML file used by `puppetlabs_spec_helper` to setup an environment for your modules with all its dependencies in the `spec/fixtures directory`. You have to, at least, add an entry that adds a symlink to your code base:

```yaml
fixtures:
  forge_modules:
    stdlib: 'puppetlabs-stdlib'
  symlinks:
    openldap: "#{source_dir}"
```

### CONFIGURE Rakefile

You need to require `puppetlabs_spec_helper/rake_tasks` in your `Rakefile` so that you can call the spec rake task that will run the `spec_prep` task for preparing your test environment using the `.fixtures.yml` file, then run the `spec_standalone task` that actually launches the unit tests, and finally run the `spec_clean` task to clean up your testing environment by deleting all the stuff in the `spec/fixtures` directory.

```ruby
require 'puppetlabs_spec_helper/rake_tasks'
```

### CONFIGURE spec/spec_helper.rb
Simply use require `puppetlabs_spec_helper/module_spec_helper` for now

```ruby
require 'puppetlabs_spec_helper/module_spec_helper'
```

Now that you have completed the setup of your environment, you are ready to start writing your module with TDD, as we will see next, in Chapter 3!