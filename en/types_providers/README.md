# Create a custom type and provider

Now that we can [install our OpenLDAP server](complex_class/README.md) and ensure that it is running, we want to be able to manage OpenLDAP databases.
For that, we will create an OpenLDAP Puppet type and a provider to manage databases using OpenLDAP’s live configuration API. We will do all this using TDD, of course!

## WRITE THE ACCEPTANCE TESTS

First, let’s create an acceptance test in `spec/acceptance/openldap_database_spec.rb` for our new type `openldap_database`

```ruby
require 'spec_helper_acceptance'
 
describe 'openldap_database' do
  context 'when running puppet code' do
    it 'should apply idempotently' do
      pp = <<-EOS
        class { 'openldap::client': } ->
        class { 'openldap::server': } ->
        openldap_database { 'dc=foo,dc=com':
          ensure => present,
        }
      EOS
 
      # Run it twice and test for idempotency
      apply_manifest(pp, :catch_failures => true)
      apply_manifest(pp, :catch_changes => true)
    end
 
    it 'can connect with ldapsearch' do
      ldapsearch('-LLL -x -b "dc=foo,dc=com"') do |r|
        expect(r.stdout).to match(/dn: dc=foo,dc=com/)
      end
    end
  end
end
```

## WRITE THE CUSTOM TYPE

Let’s write a custom Puppet type that has these properties:

* is ensurable
* has suffix as namevar
* has backend and directory properties

## WRITE THE UNIT TESTS FOR THE OPENLDAP_DATABASE TYPE

Unit tests for Puppet types live in `spec/unit/puppet/type`, so let’s create this directory first.

```shell
$ mkdir -p spec/unit/puppet/type
```

Then we can create our unit tests for our `openldap_database` type in `spec/unit/puppet/type/openldap_database_spec.rb`.

The first thing to do is to require our `spec_helper.rb` file:


```ruby
require 'spec_helper'
 
describe Puppet::Type.type(:openldap_database) do
  # Tests will go here
end
```

First, let’s validate the attributes of our custom type. We have to be sure that it accepts suffix and provider parameters and ensure, backend and directory properties:

```ruby
require 'spec_helper'
 
describe Puppet::Type.type(:openldap_database) do
  on_supported_os.each do |os, facts|
    context "on #{os}" do
      before :each do
        Facter.clear
        facts.each do |k, v|
          Facter.stubs(:fact).with(k).returns Facter.add(k) { setcode { v } }
        end
      end
 
      describe 'when validating attributes' do
        [ :suffix, :provider ].each do |param|
          it "should have a #{param} parameter" do
            expect(described_class.attrtype(param)).to eq(:param)
          end
        end
        [ :ensure, :backend, :directory ].each do |prop|
          it "should have a #{prop} property" do
            expect(described_class.attrtype(prop)).to eq(:property)
          end
        end
      end
    end
  end
end
```

#### Line 1:
Load `spec/spec_helper.rb`

#### Line 4:
Loop over all supported operating systems (in `metadata.json`)

#### Line 5:
Create a new rspec context for the operating system currently being tested

#### Line 7 – 11:
Stub all facts before each test

#### Lines 14 – 18:
Loop over each desired parameter and validate it

#### Line 19 – 23:
Loop over each desired property and validate it
Let’s also make sure that suffix is the namevar of our type:

```ruby
  describe "namevar validation" do
    it "should have :suffix as its namevar" do
      expect(described_class.key_attributes).to eq([:suffix])
    end
  end
```

### WRITE THE OPENLDAP_DATABASE TYPE

The puppet custom types live in `lib/puppet/type`, so let’s create this directory first:

We can then write our puppet type to manage OpenLDAP databases in `lib/puppet/type/openldap_database.rb`:

```ruby
Puppet::Type.newtype(:openldap_database) do
  @doc = "Manages OpenLDAP BDB and HDB databases."
 
  ensurable
 
  newparam(:suffix, :namevar => true) do
    desc "The default namevar."
  end
 
  newproperty(:backend) do
    desc "The name of the backend."
  end
 
  newproperty(:directory) do
    desc "The directory where the BDB files containing this database and associated indexes live."
  end
end
```

Let’s test:

```shell
$ bundle exec rake spec SPEC=spec/unit/puppet/type/openldap_database_spec.rb SPEC_OPTS=-fd
...
Puppet::Type::Openldap_database
  on debian-7-x86_64
    when validating attributes
      should have a suffix parameter
      should have a provider parameter
      should have a ensure property
      should have a backend property
      should have a directory property
    namevar validation
      should have :suffix as its namevar
  on redhat-7-x86_64
    when validating attributes
      should have a suffix parameter
      should have a provider parameter
      should have a ensure property
      should have a backend property
      should have a directory property
    namevar validation
      should have :suffix as its namevar
 
Finished in 0.12127 seconds (files took 0.72435 seconds to load)
12 examples, 0 failures
```

It Works!

Now let’s validate the attributes value.
We want to ensure that ensure matches present or absent and that backend matches either bdb or hdb (the 2 current supported OpenLDAP database backends) and defaults to hdb:

```ruby
  describe 'when validating attribute values' do
    describe 'ensure' do
      [ :present, :absent ].each do |value|
        it "should support #{value} as a value to ensure" do
          expect { described_class.new({
            :name   => 'dc=example,dc=com',
            :ensure => value,
          })}.to_not raise_error
        end
      end
 
      it "should not support other values" do
        expect { described_class.new({
          :name   => 'dc=example,dc=com',
          :ensure => 'foo',
        })}.to raise_error(Puppet::Error, /Invalid value/)
      end
    end
 
    describe "backend" do
      [ 'bdb', 'hdb' ].each do |backend|
        it "should support '#{backend}' as a value for backend" do
          expect { described_class.new({
            :name => 'dc=example,dc=com',
            :backend => backend,
          }) }.to_not raise_error
        end
      end
 
      it "should default to hdb" do
        expect(described_class.new({
          :name => 'dc=example,dc=com'
        })[:backend]).to eq :hdb
      end
 
      it "should not support other values" do
        expect { described_class.new({
          :name    => 'dc=example,dc=com',
          :backend => 'bar',
        } ) }.to raise_error(Puppet::Error, /Invalid value/)
      end
    end
```

And now we can adapt our custom type:

```ruby
Puppet::Type.newtype(:openldap_database) do
  @doc = "Manages OpenLDAP BDB and HDB databases."
 
  ensurable
 
  newparam(:suffix, :namevar => true) do
    desc "The default namevar."
  end
 
  newproperty(:backend) do
    desc "The name of the backend."
    newvalues('bdb', 'hdb')
    defaultto 'hdb'
  end
 
  newproperty(:directory) do
    desc "The directory where the BDB files containing this database and associated indexes live."
  end
end
```

And launch the unit tests again:

```shell
$ bundle exec rake spec SPEC=spec/unit/puppet/type/openldap_database_spec.rb SPEC_OPTS=-fd
...
Puppet::Type::Openldap_database
  on debian-7-x86_64
    when validating attributes
      should have a suffix parameter
      should have a provider parameter
      should have a ensure property
      should have a backend property
      should have a directory property
    namevar validation
      should have :suffix as its namevar
    when validating attribute values
      ensure
        should support present as a value to ensure
        should support absent as a value to ensure
        should not support other values
      backend
        should support 'bdb' as a value for backend
        should support 'hdb' as a value for backend
        should default to hdb
        should not support other values
  on redhat-7-x86_64
    when validating attributes
      should have a suffix parameter
      should have a provider parameter
      should have a ensure property
      should have a backend property
      should have a directory property
    namevar validation
      should have :suffix as its namevar
    when validating attribute values
      ensure
        should support present as a value to ensure
        should support absent as a value to ensure
        should not support other values
      backend
        should support 'bdb' as a value for backend
        should support 'hdb' as a value for backend
        should default to hdb
        should not support other values
 
Finished in 0.30884 seconds (files took 0.73265 seconds to load)
26 examples, 0 failures
```

We should ideally validate more things:
* the suffix should be `/dc=[^,]+(,dc=[^,]+)*/`
* the directory should be an absolute path
etc.

…but this will be left as an exercise for you.

## WRITE THE CUSTOM PROVIDER FOR OPENLDAP_DATABASE TYPE

Now let’s write a provider for our custom type. As said, we will use the OpenLDAP configuration API to manage the databases. So we’ll use slapcat to read the configuration and `ldapmodify` to update it. Since we’ll not have a real OpenLDAP server running on our workstation where we run the tests, we’ll have to mock the commands.

Let’s start, as usual, by writing the unit tests.

### WRITE THE UNIT TEST FOR THE OPENLDAP_DATABASE’S OLC PROVIDER

Unit tests for providers lives in `spec/unit/puppet/provider/<type>`, so let’s create this directory first:

```shell
$ mkdir -p spec/unit/puppet/provider/openldap_database
```

Next, we can create the unit tests for `openldap_database`’s `olc` provider in `spec/unit/puppet/provider/openldap_database/olc_spec.rb`:

Again, the first thing to do is to require our `spec_helper.rb` file:

```ruby
require 'spec_helper'
 
describe Puppet::Type.type(:openldap_database).provider(:olc) do
  # Tests will go here
end
```

The first thing we want to do is to list the current instances of the OpenLDAP databases and use this list as a prefetch cache for the resources. So our provider must respond to the `self.instances` and `self.prefetch` methods:

```ruby
require 'spec_helper'
 
describe Puppet::Type.type(:openldap_database).provider(:olc) do
 
  on_supported_os.each do |os, facts|
    context "on #{os}" do
      before :each do
        Facter.clear
        facts.each do |k, v|
          Facter.stubs(:fact).with(k).returns Facter.add(k) { setcode { v } }
        end
      end
 
      describe 'instances' do
        it 'should have an instance method' do
          expect(described_class).to respond_to :instances
        end
      end
 
      describe 'prefetch' do
        it 'should have a prefetch method' do
          expect(described_class).to respond_to :prefetch
        end
      end
    end
  end
end
```

### WRITE THE OPENLDAP_DATABASE’S OLC PROVIDER

The puppet providers for the `openldap_database` type lives in `lib/puppet/provider/openldap_database`, so let’s create this directory first:

```shell
$ mkdir -p lib/puppet/provider/openldap_database
```

Then we can create the `olc` provider for the `openldap_database` type in `lib/puppet/provider/openldap_database/olc.rb` with a `self.instances` and a `self.prefetch` methods plus some things we’ll need:

```ruby
Puppet::Type.type(:openldap_database).provide(:olc) do
 
  commands :slapcat => 'slapcat', :ldapmodify => 'ldapmodify'
 
  mk_resource_methods
 
  def self.instances
  end
 
  def self.prefetch
  end
 
  def exists?
    @property_hash[:ensure] == :present
  end
end
```

#### Line 3:
Declares commands we will use (used by confinement) and creates helper methods dynamically for each command.

#### Line 5:
Creates accessors (getters and setters) for each property using the `@property_hash` instance variable created by the `self.prefetch` method

#### Line 13 – 15:
Defines the `exists?` method which returns a boolean based on whether the resource is prefetched.

And let’s test:

```shell
$ bundle exec rake spec SPEC=spec/unit/puppet/provider/openldap_database/olc_spec.rb SPEC_OPTS=-fd
...
Puppet::Type::Openldap_database::ProviderOlc
  on debian-7-x86_64
    instances
      should have an instance method
    prefetch
      should have a prefetch method
  on redhat-7-x86_64
    instances
      should have an instance method
    prefetch
      should have a prefetch method
 
Finished in 0.02769 seconds (files took 0.44447 seconds to load)
4 examples, 0 failures
```

### WRITE THE UNIT TESTS FOR THE SELF.INSTANCES METHOD

Now that everything is set up, let’s write the next unit test. We want to make sure that the self.instances method returns the right resources:

We’ll test with no database:

1
2
3
4
5
6
7
8
9
10
11
        context 'without databases' do
          before :each do
            described_class.expects(:slapcat).with(
              '-b', 'cn=config', '-H',
              'ldap:///???(&(objectClass=olcDatabaseConfig)(|(objectClass=olcBdbConfig)(objectClass=olcHdbConfig)))'
            ).returns ''
          end
          it 'should return no resources' do
            expect(described_class.instances.size).to eq(0)
          end
        end
view rawwithout_databases_context.rb hosted with ❤ by GitHub
Then, with one database:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
        context 'with one database' do
          before :each do
            described_class.expects(:slapcat).with(
              '-b', 'cn=config', '-H',
              'ldap:///???(&(objectClass=olcDatabaseConfig)(|(objectClass=olcBdbConfig)(objectClass=olcHdbConfig)))'
            ).returns 'dn: olcDatabase={1}hdb,cn=config
objectClass: olcHdbConfig
olcDatabase: {1}hdb
olcDbDirectory: /var/lib/ldap
olcSuffix: dc=example,dc=com

'
          end
          it 'should return one resource' do
            expect(described_class.instances.size).to eq(1)
          end
          it 'should return the resource dc=example,dc=com' do
            expect(described_class.instances[0].instance_variable_get("@property_hash")).to eq( {
              :ensure    => :present,
              :name      => 'dc=example,dc=com',
              :suffix    => 'dc=example,dc=com',
              :backend   => 'hdb',
              :directory => '/var/lib/ldap',
            } )
          end
        end
view rawwith_one_database_context.rb hosted with ❤ by GitHub
And finally with two databases:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
        context 'with two databases' do
          before :each do
            described_class.expects(:slapcat).with(
              '-b', 'cn=config', '-H',
              'ldap:///???(&(objectClass=olcDatabaseConfig)(|(objectClass=olcBdbConfig)(objectClass=olcHdbConfig)))'
            ).returns 'dn: olcDatabase={1}hdb,cn=config
objectClass: olcHdbConfig
olcDatabase: {1}hdb
olcDbDirectory: /var/lib/ldap1
olcSuffix: dc=foo,dc=com

dn: olcDatabase={1}bdb,cn=config
objectClass: olcBdbConfig
olcDatabase: {1}bdb
olcDbDirectory: /var/lib/ldap2
olcSuffix: dc=bar,dc=com

'
          end
          it 'should return two resource' do
            expect(described_class.instances.size).to eq(2)
          end
          it 'should return the resource dc=foo,dc=com' do
            expect(described_class.instances[0].instance_variable_get("@property_hash")).to eq( {
              :ensure    => :present,
              :name      => 'dc=foo,dc=com',
              :suffix    => 'dc=foo,dc=com',
              :backend   => 'hdb',
              :directory => '/var/lib/ldap1',
            } )
          end
          it 'should return the resource dc=bar,dc=com' do
            expect(described_class.instances[1].instance_variable_get("@property_hash")).to eq( {
              :ensure    => :present,
              :name      => 'dc=bar,dc=com',
              :suffix    => 'dc=bar,dc=com',
              :backend   => 'bdb',
              :directory => '/var/lib/ldap2',
            } )
          end
        end
view rawwith_two_databases_context.rb hosted with ❤ by GitHub
If you launch the unit tests now, it will obviously fails because self.instances returns nil and we try to call the size method on it.

WRITE THE PROVIDER’S SELF.INSTANCES METHOD
Let’s write the provider code that populates the instances Array:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
Puppet::Type.type(:openldap_database).provide(:olc) do
 
  commands :slapcat => 'slapcat', :ldapmodify => 'ldapmodify'
 
  mk_resource_methods
 
  def self.instances
    databases = slapcat(
      '-b', 'cn=config', '-H',
      'ldap:///???(&(objectClass=olcDatabaseConfig)(|(objectClass=olcBdbConfig)(objectClass=olcHdbConfig)))'
    )
    databases.split("\n\n").collect do |paragraph|
      suffix = backend = directory = nil
      paragraph.split("\n").collect do |line|
        case line
        when /^olcDatabase: /
          backend = line.match(/^olcDatabase: \{\d+\}(bdb|hdb)$/).captures[0]
        when /^olcDbDirectory: /
          directory = line.split(' ')[1]
        when /^olcSuffix: /
          suffix = line.split(' ')[1]
        end
      end
      new({
        :ensure    => :present,
        :name      => suffix,
        :suffix    => suffix,
        :backend   => backend,
        :directory => directory,
      })
    end
  end
 
  def self.prefetch
  end
 
  def exists?
    @property_hash[:ensure] == :present
  end
end
view rawolc2.rb hosted with ❤ by GitHub
And if we test now:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
$ bundle exec rake spec SPEC=spec/unit/puppet/provider/openldap_database/olc_spec.rb SPEC_OPTS=-fd
...
Puppet::Type::Openldap_database::ProviderOlc
  on debian-7-x86_64
    instances
      should have an instance method
      without databases
        should return no resources
      with one database
        should return one resource
        should return the resource dc=example,dc=com
      with two databases
        should return two resource
        should return the resource dc=foo,dc=com
        should return the resource dc=bar,dc=com
    prefetch
      should have a prefetch method
  on redhat-7-x86_64
    instances
      should have an instance method
      without databases
        should return no resources
      with one database
        should return one resource
        should return the resource dc=example,dc=com
      with two databases
        should return two resource
        should return the resource dc=foo,dc=com
        should return the resource dc=bar,dc=com
    prefetch
      should have a prefetch method
 
Finished in 0.11436 seconds (files took 0.45002 seconds to load)
16 examples, 0 failures
view rawrun_unit_test4.sh hosted with ❤ by GitHub
Now that we have the self.instances method, we want to be able to create a database. We will now code the self.prefetch method, to pass the discovered instances to the catalog resources.

THE PREFETCH METHOD
The prefetch method is used to build a cache of the resources which can be used to easily assert the existence and synchronisation state of the resource, using the @property_hash instance variable.

1
2
3
4
5
6
7
8
  def self.prefetch(resources)
    databases = instances
    resources.keys.each do |name|
      if provider = databases.find{ |database| database.name == name }
        resources[name].provider = provider
      end
    end
  end
view rawprefetch.rb hosted with ❤ by GitHub
Line 1:
The method takes a list of catalog resources as argument, and is expected to associate RAL resources to each of them, if they already exist.

Line 2:
Retrieve all the discovered resources from the self.instances method. This is the most common way to prefetch resources when resources can all be automatically discovered by self.instances.

Line 4:
For each catalog resource passed to the method, look into the discovered instances to find a RAL resource with a matching name.

Line 5:
If a matching resource is found, associate it to the catalog resource. This will set the values of the @property_hash instance variable for this resource based on the values set in the self.instances method for this resource.

THE UNIT TEST FOR THE CREATE METHOD
Now let’s check that, when we want to create a resource, it generates a valid ldif and that the resource exists. In spec/unit/puppet/provider/openldap_database/olc_spec.rb:

THE CREATE METHOD
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
  def create
    ldif = %Q{dn: olcDatabase=#{resource[:backend]},cn=config
changetype: add
objectClass: olc#{resource[:backend].to_s.capitalize}Config
olcDatabase: #{resource[:backend]}
olcDbDirectory: #{resource[:directory]}
olcSuffix: #{resource[:suffix]}
olcAccess: to *
  by dn.exact=gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth manage
  by * break
olcAccess: to *
  by * read
}
    begin
      execute(%Q{echo -n "#{ldif}" | ldapmodify -Y EXTERNAL -H ldapi:///})
    rescue Exception => e
      raise Puppet::Error, "LDIF content:\n#{ldif}\nError message: #{e.message}"
    end
    ldif = %Q{dn: #{resource[:suffix]}
changetype: add
objectClass: dcObject
objectClass: organization
dc: #{resource[:suffix].split(/,?dc=/).delete_if { |c| c.empty? }[0]}
o: #{resource[:suffix].split(/,?dc=/).delete_if { |c| c.empty? }.join('.')}
}
    begin
      execute(%Q{echo -n "#{ldif}" | ldapmodify -Y EXTERNAL -H ldapi:///})
    rescue Exception => e
      raise Puppet::Error, "LDIF content:\n#{ldif}\nError message: #{e.message}"
    end
    @property_hash[:ensure] = :present
  end
view rawcreate.rb hosted with ❤ by GitHub
Finally, let’s run the acceptance test to see if it really works.

ON REDHAT 7
1
2
3
4
5
6
7
8
9
10
11
12
13
14
$ BEAKER_set=centos-7-x86_64-vagrant bundle exec rspec spec/acceptance/openldap_database_spec.rb
...
openldap_database
  running puppet code
localhost $ scp /tmp/beaker20150325-32609-1y6r3y0 centos-7-x64:/tmp/apply_manifest.pp.VtyjcH {:ignore => }
localhost $ scp /tmp/beaker20150325-32609-1p5vwqa centos-7-x64:/tmp/apply_manifest.pp.Dr6fc7 {:ignore => }
    should work with no errors
    can connect with ldapsearch
Destroying vagrant boxes
==> centos-7-x64: Forcing shutdown of VM...
==> centos-7-x64: Destroying VM and associated drives...
 
Finished in 28.65 seconds (files took 2 minutes 0.8 seconds to load)
2 examples, 0 failures
view rawrun_acceptance_redhat.sh hosted with ❤ by GitHub
Yes, it does!

ON DEBIAN 7
1
2
3
4
5
6
7
8
9
10
11
12
13
14
BEAKER_set=debian-7-x86_64-vagrant bundle exec rspec spec/acceptance/openldap_database_spec.rb
...
openldap_database
  running puppet code
localhost $ scp /tmp/beaker20150325-3533-1uv6518 debian-7-x64:/tmp/apply_manifest.pp.wOicrY {:ignore => }
localhost $ scp /tmp/beaker20150325-3533-dby97z debian-7-x64:/tmp/apply_manifest.pp.UwTEok {:ignore => }
    should work with no errors
    can connect with ldapsearch
Destroying vagrant boxes
==> debian-7-x64: Forcing shutdown of VM...
==> debian-7-x64: Destroying VM and associated drives...
 
Finished in 36.46 seconds (files took 1 minute 48.05 seconds to load)
2 examples, 0 failures
view rawrun_acceptance_debian.sh hosted with ❤ by GitHub
Now that we have a functional type and provider to manage an OpenLDAP database, we will create a function to manage the database’s Root Password in the coming part IV.