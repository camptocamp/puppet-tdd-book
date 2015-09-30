# Create a custom function

Now that we have a custom type and provider to manage OpenLDAP databases, we will add two properties to manage <code>rootdn</code> and <code>rootpw</code> to declare the admin user of the database. Normally, you should use the <code>slappasswd</code> tool to hash a password (by default, it used the SSHA algorithm, but we don't want to depend on this tool installed on the Puppet master, so we'll hash the clear password using a pure Ruby approach).

## Write the acceptance test

First, we'll write the acceptance test. We want to be able to connect to the database using admin credentials. Let's add a new context in <code>spec/acceptance/openldap_database_spec.rb</code>:

```ruby
  context 'when setting rootdn and rootpw' do
    it 'should work with no errors' do
      pp = <<-EOS
        class { 'openldap::client': } ->
        class { 'openldap::server': } ->
        openldap_database { 'dc=bar,dc=com':
          ensure    => present,
          directory => '/tmp',
          rootdn    => 'cn=admin,dc=bar,dc=com',
          rootpw    => openldap_password('secret'),
        }
      EOS

      # Run it twice and test for idempotency
      apply_manifest(pp, :catch_failures => true)
      apply_manifest(pp, :catch_changes => true)
    end

    it 'can connect with ldapsearch' do
      ldapsearch('-LLL -x -b "dc=foo,dc=com" -D "cn=admin,dc=bar,dc=com" -w secret') do |r|
        expect(r.stdout).to match(/dn: dc=foo,dc=com/)
      end
    end
  end
```

## Write the unit test

Now let's write the unit test. Unit tests for custom functions live in <code>spec/unit/puppet/parser/functions</code>, so let's create this directory first:

```shell
$ mkdir -p spec/unit/puppet/parser/functions
```

The first thing we'll do, as usual, is require our spec_helper:

```ruby
require 'spec_helper'

describe Puppet::Parser::Functions.function(:openldap_password) do
  # Unit tests go here
end
```

We'll then need to initialize the <code>scope</code> variable needed to call function on it.

```ruby
require 'spec_helper'

describe Puppet::Parser::Functions.function(:openldap_password) do
  let(:scope) { PuppetlabsSpec::PuppetInternals.scope }
end
```

Now that we have a scope, we can loop over all supported operating systems using <code>rspec-puppet-facts</code> and stub every fact in the top scope:

```ruby
require 'spec_helper'

describe Puppet::Parser::Functions.function(:openldap_password) do
  let(:scope) { PuppetlabsSpec::PuppetInternals.scope }

  on_supported_os.each do |os, facts|
    context "on #{os}" do
      before :each do
        facts.each do |k, v|
          scope.stubs(:lookupvar).with("::#{k}").returns(v)
          scope.stubs(:lookupvar).with(k).returns(v)
        end
      end
      # Unit tests go here
    end
  end
end
```

Everything is now set up, so let's start writing some tests.

### Check that the function exists

Let's first validate that the function is available:

```ruby
      it 'should exist' do
        expect(
          Puppet::Parser::Functions.function('openldap_password')
        ).to eq('function_openldap_password')
      end
```

Let's try the unit tests:

```shell
$ bundle exec rake spec SPEC=spec/unit/puppet/parser/functions/openldap_password_spec.rb SPEC_OPTS=-fd
...
false
  on debian-7-x86_64
    should exist (FAILED - 1)
  on redhat-7-x86_64
    should exist (FAILED - 2)

Failures:

  1) false on debian-7-x86_64 should exist
     Failure/Error: expect(
       
       expected: "function_openldap_password"
            got: false
       
       (compared using ==)
     # ./spec/unit/puppet/parser/functions/openldap_password_spec.rb:16:in `block (4 levels) in <top (required)>'

  2) false on redhat-7-x86_64 should exist
     Failure/Error: expect(
       
       expected: "function_openldap_password"
            got: false
       
       (compared using ==)
     # ./spec/unit/puppet/parser/functions/openldap_password_spec.rb:16:in `block (4 levels) in <top (required)>'

Finished in 0.06486 seconds (files took 0.72167 seconds to load)
2 examples, 2 failures

Failed examples:

rspec ./spec/unit/puppet/parser/functions/openldap_password_spec.rb:15 # false on debian-7-x86_64 should exist
rspec ./spec/unit/puppet/parser/functions/openldap_password_spec.rb:15 # false on redhat-7-x86_64 should exist
```

It obviously fails because we don't have the <code>openldap_password</code> function code yet. Let's start writing it.

## Write the function

Now that we have a skeleton for our unit tests, let's start writing the function. Puppet custom functions live in <code>lib/puppet/parser/functions</code>, so let's create this directory:

```shell
$ mkdir -p lib/puppet/parser/functions
```
Now let's write the function:

```ruby
module Puppet::Parser::Functions
  newfunction(:openldap_password, :type => :rvalue, :doc => <<-EOS
    Returns the openldap password hash from the clear text password.
  EOS
  ) do |args|
    # Function code goes here
  end
end
```

If we launch the unit tests again, it should work because the function now does exist:

```shell
$ bundle exec rake spec SPEC=spec/unit/puppet/parser/functions/openldap_password_spec.rb SPEC_OPTS=-fd
...
function_openldap_password
  on debian-7-x86_64
    should exist
  on redhat-7-x86_64
    should exist

Finished in 0.06929 seconds (files took 0.77882 seconds to load)
2 examples, 0 failures
```

## Write more tests

We want to make sure that our function takes one and only one argument (the plain text password). So let's check that it fails if we pass zero or two arguments:

```ruby
      context 'when passing no arguments' do
        it 'should fail' do
          expect {
            scope.function_openldap_password([])
          }.to raise_error Puppet::ParseError, /Wrong number of arguments given/
        end
      end

      context 'when passing two arguments' do
        it 'should fail' do
          expect {
            scope.function_openldap_password(['foo', 'bar'])
          }.to raise_error Puppet::ParseError, /Wrong number of arguments given/
        end
      end
```

And the actual code in the function:

```ruby
module Puppet::Parser::Functions
  newfunction(:openldap_password, :type => :rvalue, :doc => <<-EOS
    Returns the openldap password hash from the clear text password.
  EOS
  ) do |args|

    raise(Puppet::ParseError, "openldap_password(): Wrong number of arguments given") if args.size < 1 or args.size > 1
  end
end
```

Let's test:

```shell
$ bundle exec rake spec SPEC=spec/unit/puppet/parser/functions/openldap_password_spec.rb SPEC_OPTS=-fd
...
function_openldap_password
  on debian-7-x86_64
    should exist
    when giving no arguments
      should fail
    when giving two arguments
      should fail
  on redhat-7-x86_64
    should exist
    when giving no arguments
      should fail
    when giving two arguments
      should fail

Finished in 0.14474 seconds (files took 0.707 seconds to load)
6 examples, 0 failures
```

It works!

## Write the unit test that verifies the hash

Ultimately, we want our function to return the SSHA of the password with the 4 first characters of the SHA1 of the fqdn as salt.

Let's calculate the expected result in <code>irb</code> (that's what slappasswd does), the fqdn <code>foo.example.com</code> comes from rspec-puppet-facts:

```shell
$ irb
irb(main):001:0> require 'base64'
=> true
irb(main):002:0> require 'digest'
=> true
irb(main):003:0> password='secret'
=> "secret"
irb(main):004:0> salt=Digest::SHA1.digest('foo.example.com')[0..4]
=> "\xC4\x1D\xBC,\xE6"
irb(main):005:0> "{SSHA}" + Base64.encode64("#{Digest::SHA1.digest("#{password}#{salt}")}#{salt}").chomp
=> "{SSHA}jZdUkbyDYvmpSKg0x/k879g+RY7EHbws5g=="
```

 So we code the unit test this way:

```ruby
      context 'when giving a secret' do
        it 'should return the SSHA of the password with the first 4 characters of sha1("foo.example.com") as salt' do
          expect(
            scope.function_openldap_password(['secret'])
          ).to eq('{SSHA}jZdUkbyDYvmpSKg0x/k879g+RY7EHbws5g==')
        end
      end
```

## Write the function code

```ruby
module Puppet::Parser::Functions
  newfunction(:openldap_password, :type => :rvalue, :doc => <<-EOS
    Returns the openldap password hash from the clear text password.
  EOS                                                  
  ) do |args|
                                     
    raise(Puppet::ParseError, "openldap_password(): Wrong number of arguments given") if args.size < 1 or args.size > 1
 
    password = args[0]      
    salt = Digest::SHA1.digest(lookupvar('::fqdn'))[0..4]  
    
    "{SSHA}" + Base64.encode64("#{Digest::SHA1.digest("#{password}#{salt}")}#{salt}").chomp
  end
end
```

This is not very secure because every hash on one node is generated using the same salt, but it is just an example.

## Launch the unit test

```shell
$ bundle exec rake spec SPEC=spec/unit/puppet/parser/functions/openldap_password_spec.rb SPEC_OPTS=-fd
...
function_openldap_password
  on debian-7-x86_64
    should exist
    when giving no arguments
      should fail
    when giving two arguments
      should fail
    when giving a secret
      should return the SSHA of the password with the first 4 characters of sha1("foo.example.com") as salt
  on redhat-7-x86_64
    should exist
    when giving no arguments
      should fail
    when giving two arguments
      should fail
    when giving a secret
      should return the SSHA of the password with the first 4 characters of sha1("foo.example.com") as salt

Finished in 0.12028 seconds (files took 0.45853 seconds to load)
8 examples, 0 failures
```

It works!

## Launch the acceptance test

### On RedHat

```shell
$ BEAKER_set=centos-7-x86_64-vagrant bundle exec rspec spec/acceptance/openldap_database_spec.rb 
...
openldap_database
  when not setting rootdn and rootpw
localhost $ scp /tmp/beaker20150407-4273-1a19l9o centos-7-x64:/tmp/apply_manifest.pp.F6GY0s {:ignore => }
localhost $ scp /tmp/beaker20150407-4273-brpequ centos-7-x64:/tmp/apply_manifest.pp.nEHJi7 {:ignore => }
    should work with no errors
    can connect with ldapsearch
  when setting rootdn and rootpw
localhost $ scp /tmp/beaker20150407-4273-cw80mz centos-7-x64:/tmp/apply_manifest.pp.5xYinE {:ignore => }
localhost $ scp /tmp/beaker20150407-4273-rlhd7a centos-7-x64:/tmp/apply_manifest.pp.YKZNQ2 {:ignore => }
    should work with no errors
    can connect with ldapsearch
Destroying vagrant boxes
==> centos-7-x64: Forcing shutdown of VM...
==> centos-7-x64: Destroying VM and associated drives...

Finished in 33.09 seconds (files took 1 minute 53.74 seconds to load)
4 examples, 0 failures
```

It works! We can create a database with a rootdn and a rootpw and connect to it using the credentials.

### On Debian

Now let's test on Debian:

```shell
$ BEAKER_set=debian-7-x86_64-vagrant bundle exec rspec spec/acceptance/openldap_database_spec.rb 
...
openldap_database
  when not setting rootdn and rootpw
localhost $ scp /tmp/beaker20150407-6104-1xk90v8 debian-7-x64:/tmp/apply_manifest.pp.SRNaJW {:ignore => }
localhost $ scp /tmp/beaker20150407-6104-9ni8y1 debian-7-x64:/tmp/apply_manifest.pp.FwTqA4 {:ignore => }
    should work with no errors
    can connect with ldapsearch
  when setting rootdn and rootpw
localhost $ scp /tmp/beaker20150407-6104-7tk1rl debian-7-x64:/tmp/apply_manifest.pp.azrQpS {:ignore => }
localhost $ scp /tmp/beaker20150407-6104-k6hhs0 debian-7-x64:/tmp/apply_manifest.pp.F9u9f8 {:ignore => }
    should work with no errors
    can connect with ldapsearch
Destroying vagrant boxes
==> debian-7-x64: Forcing shutdown of VM...
==> debian-7-x64: Destroying VM and associated drives...

Finished in 36.59 seconds (files took 1 minute 31.02 seconds to load)
4 examples, 0 failures
```

Everything is OK.