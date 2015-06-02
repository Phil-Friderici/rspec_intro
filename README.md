# Rspec testing für Puppetmodule

# Was ?
Rspec Tests beschreiben die gewünschten Funktionalitäten (Erwartungshaltungen) und ermöglichen die automatisierte Überprüfung davon.
Vollständige Tests beschreiben alle funktionalen Anforderungen die an ein Modul gestellt werden.


# Warum ?
Rspec Tests helfen Fehler zu finden. Logische Fehler, konzeptionelle Fehler und auch Software Fehler.
Rspec Tests erleichtern es in komplexen Modulen Änderungen ohne unerwünschte Nebenwirkungen vorzunehmen. (zB: fremde Module)
Fehlende automatische Tests füren die Idee der kontinuierlichen Integration (CI) ad absurdum.


# Empfehlungen
### Von Anfang an
Entwickle die Tests von Anfang an parallel zur Funktionalität. Während der Entwicklung weiss man am besten worauf es ankommt und wo
eventuelle Stolpersteine liegen.
Im Idealfall wird erst mit den Tests das geforderte Verhalten definiert und anschließend im Modul implementiert.

### Aufbau
Schreibe die Test nach dem Motto: "Vom Allgemeinen zum Speziellen"

Als erstes sollte die Grundfunktionalität des Modul getestet werden. Bewährt hat sich je ein Test mit minimaler (nur Pflichtparameter)
und maximaler (inklusive optionaler Parameter) Funktionalität. Dies ist vom Einzelfall abhängig.

Nachdem die Grundfunktionalität vollständig überprüft wurde, reicht es in den folgenden Test nur noch die Einzelaspekte zu
überprüfen. Wenn unterschiedlichen Werte für einen Parameter getestet werden, reicht es jetzt nur die dadurch beeinflussten
Zustände zu überprüfen.

Beispiel:  
ActiveDirectory Modul

Vollständig:
```ruby
it {
  should contain_file('afs_init_script').only_with({
    'ensure'  => 'file',
    'path'    => '/etc/init.d/openafs-client',
    'owner'   => 'root',
    'group'   => 'root',
    'mode'    => '0755',
    'source'  => 'puppet:///modules/afs/openafs-client-RedHat',
  })
}
```

Einzelaspekt:
```ruby
it {
  should contain_file('afs_init_script').with({
    'ensure'  => 'absent',
  })
}
```

### Gegenproben
Verwende Gegenproben wo es Sinn ergibt. Gegenproben können das Nichtvorhandensein von Funktions- oder Klassenaufrufen sein.
Parameter die nicht gesetzt sein dürfen und auch der Abbruch im Fehlerfalle sollen getestet werden.

Nichtvorhandensein testen:
```ruby
it { should_not contain_package('pkg') }
it { should_not contain_file('awk_symlink') }
it { should_not contain_exec('locale-gen') }
it { should have_pam__service_resource_count(0) }
```

Abbruch testen:
```ruby
it 'should fail' do
  expect {
    should contain_class('pam')
  }.to raise_error(Puppet::Error,/Pam: vas is not supported on #{v[:osfamily]} #{v[:release]}/)
end
```

### Submodule
Jede Klasse (oder Define) soll für sich isoliert getestet werden.
Wenn eine Klasse (init.pp) eine andere aufruft (submodule.pp), braucht in der aufrufenden Klasse nur der Aufruf der anderen Klasse
und der verwendeten Parameter überprüft werden. Teste nicht das Ergebnis der zweiten Klasse. Vertraue!

Beispiel:
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

Guter Test:  
spec/classes/init_spec.rb
```ruby
it { should have_pam__service_resource_count(1) }
```

Schlechter Test:  
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


# Grenzen
Spec Tests können keinen Nachweis der Fehlerfreiheit erbringen. Es kann lediglich nachweisen das bestimmte Testfälle erfolgreich sind.
Es können nur Fehler gefunden werden für die ein Test existiert.
Jede Funktionalitätsänderung sollte dementsprechend (zwingend) mit entsprechenden Tests ausgeliefert werden.


# Weblinks
Informationen aus erster Hand:
- http://rspec-puppet.com/tutorial/
- http://rspec-puppet.com/matchers/

Rspec Erfahrungsbericht (nicht Puppet spezifisch):
- https://blog.softwareinmotion.de/tag/rspec/

### Ausführliches Ausgabeformat
~/.bashrc:
```bash
export SPEC_OPTS="--format documentation"
```
