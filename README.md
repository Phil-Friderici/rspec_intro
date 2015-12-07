# Rspec testing Puppet modules

# What ?
Rspec tests describe the wanted functionality (expectations) and allows automated testing of them.
Complete tests describe all functional requirements for a module.


# Why ?
Help to find errors, Logical errors, conceptual errors and also errors caused by the used software environment.
Tests ease maintaining and implementation of new features into unknown or complex modules.
The idea of continuous integration only works with automated tests.


# Recommendations
### Right From The Start
Create the tests simultaneously while writing the code. While developing, it is best known what is really essential
and where the stumbling blocks are to be found.
In an ideal world you would define the required behaviour (tests) first before you start to implement the code of the module.


### Structure
Write the test on the motto: "from the general to the specific"

First you should test the default behaviour of the module. What does the module when called without using any optional parameter.
You could also add a test which uses most to all features at once. This depends on the case of application.

After the basic functionallity is fully tested including all details, it is sufficient to only verify individual aspects
in later tests. When testing different values for a parameter, it is fair enough to only check the affected conditions.


Example:
puppet-module-autofs (https://github.com/gusson/puppet-module-autofs)


Testing the complete resource:
```ruby
it do
  should contain_service('autofs').with({
    'ensure'    => 'running',
    'name'      => 'autofs',
    'enable'    => 'true',
    'require'   => 'Package[autofs]',
    'subscribe' => ['File[autofs_sysconfig]','File[auto.master]'],
  })
end
```

Testing an individual aspect only:
```ruby
it { should contain_service('autofs').with_ensure('stopped') }
```

### Crosschecking
Use cross checks where it makes sense. Cross checks could check that function or class calls are not included.
Resource parameters that shouldn't be set and error messages because of failures should be tested.

Testing absence:
```ruby
it { should_not contain_file('awk_symlink') }
it { should_not contain_exec('locale-gen') }
it { should have_pam__service_resource_count(0) }
it { should contain_file('/etc/logrotate.d/apache').without_owner('root') }
it { should contain_file('postfix_main.cf').without_content(/^virtual_alias_maps = hash:\/etc\/postfix\/virtual$/) }
```

Testing error messages:
```ruby
it 'should fail' do
  expect do
    should contain_class(subject)
  end.to raise_error(Puppet::Error, /Pam: vas is not supported on #{v[:osfamily]} #{v[:release]}/)
end
```

### Submodules
Each class and define should be tested isolated on it's own.
When a class (init.pp) calls another (submodule.pp) you only need to test the call and the used parameters itself.
Don't test the outcome of the other class. Confide!

Example:

[manifest/init.pp](https://github.com/ghoneycutt/puppet-module-pam/blob/8e44c135f0587e81668c2024cbb0d64a8f2c808f/manifests/init.pp#L945)
```puppet
create_resources('pam::service',$services)
```

[manifest/service.pp](https://github.com/ghoneycutt/puppet-module-pam/blob/8e44c135f0587e81668c2024cbb0d64a8f2c808f/manifests/service.pp#L35-L42)
```puppet
file { "pam.d-service-${name}":
  ensure  => $file_ensure,
  path    => "${pam_config_dir}/${name}",
  content => $my_content,
  owner   => 'root',
  group   => 'root',
  mode    => '0644',
}
```

Good testing:
spec/classes/init_spec.rb
```ruby
let (:params) { {:services => { 'testservice' => { 'content' => 'foo' } } } }

it { should have_pam__service_resource_count(1) }

it do
  should contain_pam__service('testservice').with({
    'content' => 'foo',
  })
end

```

Bad testing:
[spec/classes/init_spec.rb](https://github.com/ghoneycutt/puppet-module-pam/blob/8e44c135f0587e81668c2024cbb0d64a8f2c808f/spec/classes/init_spec.rb#L259-L271)
```ruby
let (:params) { {:services => { 'testservice' => { 'content' => 'foo' } } } }

it {
  should contain_file('pam.d-service-testservice').with({
    'ensure'  => 'file',
    'path'    => '/etc/pam.d/testservice',
    'owner'   => 'root',
    'group'   => 'root',
    'mode'    => '0644',
  })
}

it { should contain_file('pam.d-service-testservice').with_content('foo') }

```

# Boundaries
Rspec testing can't deliver the proof of being free from errors. It can only proof that the particular test cases have been met.
It can only find errors in case a test does exist.
Therefore each functional change should (mandatory) be delivered with corrosponing tests.


# Weblinks
First-hand Information:
- http://rspec-puppet.com/tutorial/
- http://rspec-puppet.com/matchers/

Rspec tipps (not Puppet related):
- http://blog.carbonfive.com/2010/10/21/rspec-best-practices/

### Verbose output while testing
~/.bashrc:
```bash
export SPEC_OPTS="--format documentation"
```
