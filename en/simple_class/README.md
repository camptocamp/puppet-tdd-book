# Create a simple class

Now that all the [setup work is done](setup/README.md), you can start writing some code. You are going to develop this module in a behavior/test driven manner, meaning you will write tests to describe how the module should behave before writing the actual code.

We will create an openldap::client class that, when declared, will install the OpenLDAP clients tools, including the ldapsearch command, so that we can connect to an LDAP server.

## WRITE YOUR FIRST ACCEPTANCE TEST

### GETTING STARTED

Now for the first tests, we will create a test to validate the `openldap::client` class' behavior in `spec/acceptance/openldap__client_spec.rb`. By convention, you should use a double underscore to replace the double colon in the class name.

The first thing to do in any test file is require our `spec_helper_acceptance.rb` file.

```ruby
require 'spec_helper_acceptance'
 
describe 'openldap::client' do
  # tests will go here
end
```

### THE FIRST ACCEPTANCE TEST: TESTING THAT THE PUPPET CATALOG CONVERGES AT FIRST RUN

```ruby
require 'spec_helper_acceptance'
 
describe 'openldap::client' do
  describe 'running puppet code' do
    it 'should work with no errors' do
      pp = <<-EOS
        class { 'openldap::client': }
      EOS
 
      # Run it twice and test for idempotency
      apply_manifest(pp, :catch_failures => true)
      apply_manifest(pp, :catch_changes => true)
    end
  end
end
```

#### LINE 6 – 8
Store the Puppet manifest to run with `apply_manifest()` in the pp variable.

#### LINE 10
Run the manifest and catch any failure.

#### LINE 11
Run the manifest a second time and catch any change. Your manifest should converge at first run.

If you run your tests right now, you should see a nasty block of errors because the `openldap::client` class doesn’t actually exist yet. Let’s test with the centos7 nodeset:

```shell
$ BEAKER_set=centos-7 bundle exec rspec spec/acceptance/openldap__client_spec.rb
Beaker::Hypervisor, found some docker boxes to create
Provisioning docker
provisioning centos-7-x64
Using docker server at 0.0.0.0
...
openldap::client
  running puppet code
localhost $ scp /tmp/beaker20151109-12989-iond0f centos-7-x64:/tmp/apply_manifest.pp.Biy2Qi {:ignore => }
    should work with no errors (FAILED - 1)
Warning: ssh connection to centos-7-x64 has been terminated
Cleaning up docker

Failures:

  1) openldap::client running puppet code should work with no errors
     Failure/Error: apply_manifest(pp, :catch_failures => true)
     Beaker::Host::CommandFailure:
       Host 'centos-7-x64' exited with 1 running:
        puppet apply --verbose --detailed-exitcodes /tmp/apply_manifest.pp.Biy2Qi
       Last 10 lines of output were:
       	Info: Loading facts
       	Error: Evaluation Error: Error while evaluating a Resource Statement, Could not find declared class openldap::client at /tmp/apply_manifest.pp.Biy2Qi:1:9 on node centos-7-x64.wrk.cby.camptocamp.com
...
Finished in 12.43 seconds (files took 4 minutes 10.8 seconds to load)
1 example, 1 failure

Failed examples:

rspec ./spec/acceptance/openldap__client_spec.rb:5 # openldap::client running puppet code should work with no errors
```

#### LINE 1
The command to launch includes the nodeset to use in the environment variable BEAKER_set and specify the acceptance test to run.

#### LINE 6 – 7
The label of the describe blocks in the acceptance test (line 3 and 4 of `spec/acceptance/openldap__client_spec.rb`)

#### LINE 8
Scp the Puppet manifests to run to th container

#### LINE 9
Result of the test (failure)

#### LINE 14 – 23
Detail of the failure

#### LINE 25
Acceptance test run timer summary

#### LINE 25
Acceptance test result summary

#### LINE 27 – 29
Failure summary   It basically says that the Puppet catalog does not even compile because it can’t find the `openldap::client` class. So let’s write a unit test that verifies that, at least, the catalog compiles.

## WRITE YOUR FIRST UNIT TEST

 
### GETTING STARTED
We’ll now write a unit test for our `openldap::client` class that validates that the Puppet catalog compiles. Unit tests for classes lives in `spec/classes`, so let’s create the directory first.

```shell
$ mkdir spec/classes
```

Then, we’ll create a unit test file for the class openldap::client in `spec/classes/openldap__client_spec.rb`.

The first thing to do in any test file is to require our `spec_helper.rb` file.

```ruby
require 'spec_helper'
 
describe 'openldap::client' do
  # tests will go here
end
```

### THE FIRST UNIT TEST : TESTING THAT THE CATALOG COMPILES

As first test, you should always be sure that your catalog compiles.

```ruby
require 'spec_helper'
 
describe 'openldap::client' do
  it { is_expected.to compile.with_all_deps }
end
```

#### LINE 4
Test that verifies that the catalog actually compiles.

If you run your test right now, you should also see a big block of errors because the class does not exist yet.

```shell
$ bundle exec rake spec SPEC_OPTS=-fd
Notice: Preparing to install into /home/puppet-tdd/puppet-openldap/spec/fixtures/modules ...
Notice: Downloading from https://forgeapi.puppetlabs.com ...
Notice: Installing -- do not interrupt ...
/home/puppet-tdd/puppet-openldap/spec/fixtures/modules
└── puppetlabs-stdlib (v4.9.0)
...
openldap::client
  should compile into a catalogue without dependency cycles (FAILED - 1)

Failures:

  1) openldap::client should compile into a catalogue without dependency cycles
     Failure/Error: it { is_expected.to compile.with_all_deps }
       error during compilation: Evaluation Error: Error while evaluating a Function Call, Could not find class ::openldap::client for foo.example.com at line 1:1 on node foo.example.com
     # ./spec/classes/openldap__client_spec.rb:4:in `block (2 levels) in <top (required)>'

Finished in 0.14854 seconds (files took 0.6331 seconds to load)
1 example, 1 failure

Failed examples:

rspec ./spec/classes/openldap__client_spec.rb:4 # openldap::client should compile into a catalogue without dependency cycles
```

#### LINE 1
The command to run to launch unit tests. It tells bundler to exec the rake task spec (defined in `puppetlabs_spec_helper`) with the environment variable `SPEC_OPTS` set to `-d` which sets the output format to documentation for a cleaner output.

#### LINE 2 – 6
Install dependencies fixtures from .fixtures.yml into spec/fixtures

#### LINE 8
The label of the describe block (line 3 of `spec/classes/openldap__client_spec.rb`)

#### LINE 9
Result of the unit test

#### LINE 11 – 18
Details of the failures

#### LINE 19
Unit test timer summary

#### LINE 20
Unit test summary

#### LINE 21 – 23
Failures summary


## WRITE THE CLASS

### GETTING STARTED

Now that everything is set up, let’s write the actual Puppet code! First, create the directory where our manifests will live.

```shell
$ mkdir manifests
```

Next, create our `openldap::client` class into manifests/client.pp

```puppet
class openldap::client {
}
```

If you save and run the unit tests again now, you’ll see that now that the class exists, our unit test passes.

```shell
$ bundle exec rake spec SPEC_OPTS=-fd
...
openldap::client
  should compile into a catalogue without dependency cycles

Finished in 0.15845 seconds (files took 0.62593 seconds to load)
1 example, 0 failures
```

and our acceptance test also passes.


```shell
$ BEAKER_set=centos-7 bundle exec rspec spec/acceptance/openldap__client_spec.rb
...
Beaker::Hypervisor, found some docker boxes to create
Provisioning docker
provisioning centos-7-x64
...
openldap::client
  running puppet code
localhost $ scp /tmp/beaker20151109-13967-1rfktlz centos-7-x64:/tmp/apply_manifest.pp.zjO9WQ {:ignore => }
localhost $ scp /tmp/beaker20151109-13967-cx62q5 centos-7-x64:/tmp/apply_manifest.pp.2P1ikI {:ignore => }
    should work with no errors
Warning: ssh connection to centos-7-x64 has been terminated
Cleaning up docker

Finished in 16.44 seconds (files took 3 minutes 15.3 seconds to load)
1 example, 0 failures
```

but it doesn’t yet do what we want… Let’s add the acceptance test that describe what we really want, i.e. be able to connect to an ldap server.

## THE NEXT CHECK: TESTING THAT WE CAN ACTUALLY CONNECT TO AN LDAP SERVER

You’ll test that you can really connect to a public ldap test server with `ldapsearch`. First, you need to create a method that will launch the `ldapsearch` command. Put it in your `spec_helper_acceptance.rb` so that you can use it in all your acceptance test files

```ruby
def ldapsearch(cmd, exit_codes = [0,1], &block)
  shell("ldapsearch #{cmd}", :acceptable_exit_codes => exit_codes, &block)
end
```

#### LINE 1
Function declaration

#### LINE 2
Use the shell beaker DSL function to actually launch command

Now you can use it in your acceptance test:

```ruby
require 'spec_helper_acceptance'
 
describe 'openldap::client' do
  describe 'running puppet code' do
    it 'should work with no errors' do
      pp = <<-EOS
        class { 'openldap::client': }
      EOS
 
      # Run it twice and test for idempotency
      apply_manifest(pp, :catch_failures => true)
      apply_manifest(pp, :catch_changes => true)
    end
 
    it 'can connect to an ldap test server with ldapsearch' do
      ldapsearch('-LLL -h ldap.forumsys.com -D "uid=tesla,dc=example,dc=com" -b "dc=example,dc=com" -w password') do |r|
        expect(r.stdout).to match(/dn: dc=example,dc=com/)
      end
    end
  end
end
```

#### LINE 15 – 19
Declare a new test that actually runs the `ldapsearch` command. If you save and run beaker you’ll have an error because it doesn’t find the `ldapsearch` command:


```shell
BEAKER_set=centos-7 bundle exec rspec spec/acceptance/openldap__client_spec.rb
...
Beaker::Hypervisor, found some docker boxes to create
Provisioning docker
provisioning centos-7-x64
Using docker server at 0.0.0.0
...
openldap::client
  running puppet code
localhost $ scp /tmp/beaker20151109-14665-oorjrm centos-7-x64:/tmp/apply_manifest.pp.mTwar0 {:ignore => }
localhost $ scp /tmp/beaker20151109-14665-k4dxcy centos-7-x64:/tmp/apply_manifest.pp.WoIEM1 {:ignore => }
    should work with no errors
    can connect to an ldap test server with ldapsearch (FAILED - 1)
Warning: ssh connection to centos-7-x64 has been terminated
Cleaning up docker

Failures:

  1) openldap::client running puppet code can connect to an ldap test server with ldapsearch
     Failure/Error: ldapsearch('-LLL -h ldap.forumsys.com -D "uid=tesla,dc=example,dc=com" -b "dc=example,dc=com" -w password') do |r|
     Beaker::Host::CommandFailure:
       Host 'centos-7-x64' exited with 127 running:
        ldapsearch -LLL -h ldap.forumsys.com -D "uid=tesla,dc=example,dc=com" -b "dc=example,dc=com" -w password
       Last 10 lines of output were:
       	bash: ldapsearch: command not found
...
Finished in 16.24 seconds (files took 3 minutes 35.2 seconds to load)
2 examples, 1 failure

Failed examples:

rspec ./spec/acceptance/openldap__client_spec.rb:15 # openldap::client running puppet code can connect to an ldap test server with ldapsearch
```

The `ldapsearch` command is available in the `openldap-client` package on RedHat, so let’s be sure that a package resource with named `openldap-clients` exists in the catalog.

```ruby
describe ‘openldap::client’ do
  it { is_expected.to compile.with_all_deps }
  it { is_expected.to contain_package('openldap-clients').with(
    {
      :ensure => :present,
    }
  ) }
end
```

If you save and run the unit tests, you should have an error because you don’t actually have a file resource named `openldap-clients` in your catalog


```shell
$ bundle exec rake spec SPEC_OPTS=-fd
...
openldap::client
  should compile the catalogue without cycles
  should contain Package[openldap-clients] with ensure => :present (FAILED - 1)
 
Failures:
 
  1) openldap::client should contain Package[openldap-clients] with ensure => :present
     Failure/Error: it { is_expected.to contain_package('openldap-clients').with(
       expected that the catalogue would contain Package[openldap-clients]
     # ./spec/classes/openldap__client_spec.rb:5:in `block (2 levels) in <top (required)>'
...
Finished in 0.79859 seconds (files took 0.72733 seconds to load)
2 examples, 1 failure
 
Failed examples:
 
rspec ./spec/classes/openldap__client_spec.rb:5 # openldap::client should contain Package[openldap-clients] with ensure => :present
```


So, let’s fix this adding the installation of ldap client tools in `openldap::client` class:

```puppet
class openldap::client {
  package { 'openldap-clients':
    ensure => present,
  }
}
```

And launch the unit tests again:

```shell
$ bundle exec rake spec SPEC_OPTS=-fd
...
openldap::client
  should compile the catalogue without cycles
  should contain Package[openldap-clients] with ensure => :present
...
Finished in 1.24 seconds (files took 0.69623 seconds to load)
2 examples, 0 failures
```


Then, the acceptance tests:

```shell
$ BEAKER_set=centos-7 bundle exec rspec spec/acceptance/openldap__client_spec.rb 
...
Beaker::Hypervisor, found some docker boxes to create
Provisioning docker
provisioning centos-7-x64
Using docker server at 0.0.0.0
...
openldap::client
  running puppet code
localhost $ scp /tmp/beaker20151109-15756-1jslev3 centos-7-x64:/tmp/apply_manifest.pp.leqXMR {:ignore => }
localhost $ scp /tmp/beaker20151109-15756-ipwpl3 centos-7-x64:/tmp/apply_manifest.pp.Qf66ii {:ignore => }
    should work with no errors
    can connect to an ldap test server with ldapsearch
Warning: ssh connection to centos-7-x64 has been terminated
Cleaning up docker

Finished in 18.18 seconds (files took 3 minutes 7 seconds to load)
2 examples, 0 failures
```

Now we have the behavior we wanted. We can connect to an ldap server. At least on Centos/RedHat7… But what happens if you run the tests on Debian 7? Let’s see…

```shell
$ BEAKER_set=debian-7 bundle exec rspec spec/acceptance/openldap__client_spec.rb 
...
openldap::client
  running puppet code
localhost $ scp /tmp/beaker20150301-5515-13tres9 debian-7-x64:/tmp/apply_manifest.pp.XUHQAo {:ignore => }
    should work with no errors (FAILED - 1)
    can connect to an ldap test server with ldapsearch (FAILED - 2)
Destroying vagrant boxes
==> debian-7-x64: Forcing shutdown of VM...
==> debian-7-x64: Destroying VM and associated drives...
 
Failures:
 
  1) openldap::client running puppet code should work with no errors
     Failure/Error: apply_manifest(pp, :catch_failures => true)
     Beaker::Host::CommandFailure:
       Host 'debian-7-x64' exited with 4 running:
        puppet apply --verbose --detailed-exitcodes /tmp/apply_manifest.pp.XUHQAo
       Last 10 lines of output were:
       	Error: Execution of '/usr/bin/apt-get -q -y -o DPkg::Options::=--force-confold install openldap-clients' returned 100: Reading package lists...
       	Building dependency tree...
       	Reading state information...
       	E: Unable to locate package openldap-clients
       	Error: /Stage[main]/Openldap::Client/Package[openldap-clients]/ensure: change from purged to present failed: Execution of '/usr/bin/apt-get -q -y -o DPkg::Options::=--force-confold install openldap-clients' returned 100: Reading package lists...
       	Building dependency tree...
       	Reading state information...
       	E: Unable to locate package openldap-clients
       	Info: Creating state file /var/lib/puppet/state/state.yaml
       	Notice: Finished catalog run in 0.71 seconds
...
  2) openldap::client running puppet code can connect to an ldap test server with ldapsearch
     Failure/Error: ldapsearch('ldapsearch -h ldap.forumsys.com -D "uid=tesla,dc=example,dc=com" -b "dc=example,dc=com" -w password') do |r|
     Beaker::Host::CommandFailure:
       Host 'debian-7-x64' exited with 127 running:
        ldapsearch ldapsearch -h ldap.forumsys.com -D "uid=tesla,dc=example,dc=com" -b "dc=example,dc=com" -w password
       Last 10 lines of output were:
       	bash: ldapsearch: command not found
...
Finished in 12.91 seconds (files took 4 minutes 30.9 seconds to load)
2 examples, 2 failures
 
Failed examples:
 
rspec ./spec/acceptance/openldap__client_spec.rb:5 # openldap::client running puppet code should work with no errors
rspec ./spec/acceptance/openldap__client_spec.rb:15 # openldap::client running puppet code can connect to an ldap test server with ldapsearch
```

It fails because the package `openldap-clients` does not exist on Debian and thus, the `ldapsearch` command is not installed.

On Debian, the `ldapsearch` command lives in the `ldap-utils` package and not in `openldap-clients`.

We have to be sure that, on Debian family OS, Puppet installs the `ldap-utils` package and not the `openldap-clients` package. For that, we can test the content of the catalog with a unit tests.

`rspec-puppet` sends the `:facts` hash to the Puppet compiler to populate the facts or top scope variables you use in your manifests.

We have to add some logic in our unit test because our catalog will be different.

You can either populate the `:facts` hash yourself:

```ruby
require 'spec_helper'
 
describe 'openldap::client' do
  context 'on Debian7' do
    let(:facts) { { :osfamily => 'Debian' } }
    it { is_expected.to compile.with_all_deps }
    it { is_expected.to contain_package('openldap-clients').with(
      {
        :ensure => :present,
        :name   => 'ldap-utils',
      }
    ) }
  end
  context 'on RedHat' do
    let(:facts) { { :osfamily => 'RedHat' } }
    it { is_expected.to compile.with_all_deps }
    it { is_expected.to contain_package('openldap-clients').with(
      {
        :ensure => :present,
        :name   => 'openldap-clients',
      }
    ) }
  end
end
```

Or, probably better, let [`rspec-puppet-facts`](https://github.com/mcanevet/rspec-puppet-facts) populate the `:facts` hash for you and avoid duplicate code:

```ruby
require 'spec_helper'
 
describe 'openldap::client' do
  on_supported_os.each do |os, facts|
    context "on #{os}" do
      let(:facts) do
        facts
      end
 
      it { is_expected.to compile.with_all_deps }
 
      case facts[:osfamily]
      when 'Debian'
        it { is_expected.to contain_package('openldap-clients').with(
          {
            :ensure => :present,
            :name   => 'ldap-utils',
          }
        ) }
      else
        it { is_expected.to contain_package('openldap-clients').with(
          {
            :ensure => :present,
            :name   => 'openldap-clients',
          }
        ) }
      end
    end
  end
end
```

#### LINE 4
Loop over every supported operating system declared in your `metadata.json`.

Add the `rspec-puppet-facts` gem to your `Gemfile`:

```ruby
source 'https://rubygems.org'
 
gem 'puppet',                 :require => false
gem 'beaker-rspec',           :require => false
gem 'puppetlabs_spec_helper', :require => false
gem 'rspec-puppet-facts',     :require => false
view rawGemfile.rb hosted with ❤ by GitHub
And update your gemset:
1
$ bundle update
```

Load `rspec-puppet-facts` in your `spec/spec_helper.rb`

```ruby
require 'puppetlabs_spec_helper/module_spec_helper'
require 'rspec-puppet-facts'
include RspecPuppetFacts
```

Then, run the unit tests again

```ruby
$ bundle exec rake spec SPEC_OPTS=-fd
...
openldap::client
  on debian-7-x86_64
    should compile the catalogue without cycles
    should contain Package[openldap-clients] with ensure => :present and name => "ldap-utils" (FAILED - 1)
  on redhat-7-x86_64
    should compile the catalogue without cycles
    should contain Package[openldap-clients] with ensure => :present and name => "openldap-clients"
 
Failures:
 
  1) openldap::client on debian-7-x86_64 should contain Package[openldap-clients] with ensure => :present and name => "ldap-utils"
     Failure/Error: it { is_expected.to contain_package('openldap-clients').with(
       expected that the catalogue would contain Package[openldap-clients] with name set to "ldap-utils" but it is set to "openldap-clients"
     # ./spec/classes/openldap__client_spec.rb:15:in `block (4 levels) in <top (required)>'
...
Finished in 1.01 seconds (files took 0.69753 seconds to load)
4 examples, 1 failure
 
Failed examples:
 
rspec ./spec/classes/openldap__client_spec.rb:15 # openldap::client on debian-7-x86_64 should contain Package[openldap-clients] with ensure => :present and name => "ldap-utils"
```

Now we can fix the `openldap::client` class:

```puppet
class openldap::client {
  $package_name = $::osfamily ? {
    'Debian' => 'ldap-utils',
    'RedHat' => 'openldap-clients',
  }
  package { 'openldap-clients':
    ensure => present,
    name   => $package_name,
  }
}
```

And run the tests again

```shell
$ bundle exec rake spec SPEC_OPTS=-fd
...
openldap::client
  on debian-7-x86_64
    should compile the catalogue without cycles
    should contain Package[openldap-clients] with ensure => :present and name => "ldap-utils"
  on redhat-7-x86_64
    should compile the catalogue without cycles
    should contain Package[openldap-clients] with ensure => :present and name => "openldap-clients"
...
Finished in 1.1 seconds (files took 0.70695 seconds to load)
4 examples, 0 failures
```

It works!

Let’s try the acceptance test with the Debian7 nodeset to test the actual behavior:

```shell
$ BEAKER_set=debian-7 bundle exec rspec spec/acceptance/openldap__client_spec.rb 
...
openldap::client
  running puppet code
localhost $ scp /tmp/beaker20150301-11248-qtq1sw debian-7-x64:/tmp/apply_manifest.pp.zR7atA {:ignore => }
localhost $ scp /tmp/beaker20150301-11248-1uv28s2 debian-7-x64:/tmp/apply_manifest.pp.e6JdFV {:ignore => }
    should work with no errors
    can connect to an ldap test server with ldapsearch
Destroying vagrant boxes
==> debian-7-x64: Forcing shutdown of VM...
==> debian-7-x64: Destroying VM and associated drives...
 
Finished in 34.33 seconds (files took 4 minutes 8.8 seconds to load)
2 examples, 0 failures
```

It works too!

Now that you know how to develop a simple Puppet class in a behavior/test driver manner, we will see in Chapter 3 how to code a more complex class.