# Barman module for Puppet

## Description

This module manages the installation of Barman and the configuration of PostgreSQL servers to be backed up.

For further information on barman:

* [Project homepage](http://www.pgbarman.org)
* [Barman documentation](http://docs.pgbarman.org)

## Installation

The module can be installed automatically with the *puppet* command on the master, or manually by cloning the
repository in your puppet module path.

### Installing via puppet

The latest version of the module can be installed automatically by supplying the repository information to
the module installer:

    # puppet module install deviseit-barman

This will take care of the dependencies as well.

### Installing manually

If you choose to install manually, you will have to clone the repository in the module path.

## Usage

### barman

The barman class installs barman. Currently only Ubuntu and Debian are supported.

Intensive testing has only been done on Ubuntu 12.04 LTS.

In order to install Barman with the defaults option, it is sufficient to just include the
barman class:

    class { 'barman': }

The package of latest version of barman is always available in PGDG
[apt](http://apt.postgresql.org/) and
[yum](http://yum.postgresql.org/) repository.
If you want to setup it for your installation the easiest way is to
use the
[postgresql](https://github.com/puppetlabs/puppetlabs-postgresql)
module.

    class { 'postgresql::globals':
      manage_package_repo => true,
    }->
    class { 'barman': }

All the configuration options that barman accepts can be overridden from the package defaults.

Example usage:

    class { barman:
      logfile  => '/var/log/barman/something_else.log',
      compression => 'bzip2',
      pre_backup_script = '/usr/bin/touch /tmp/started',
      post_backup_script = '/usr/bin/touch /tmp/stopped',
      custom_lines = '; something'
    }

#### Parameters

* **logfile** - A different log file. The default is '/var/log/barman/barman.log'
* **compression** - Compression algorithm. Currently supports 'gzip' (default),
                    'bzip2', and 'custom'. Disabled if false.
* **pre_backup_script** - Script to launch before backups. Disabled if false
                          (default).
* **post_backup_script** - Script to launch after backups. Disabled if false
                           (default).
* **custom_lines** - Custom configuration directives (e.g. for custom
                     compression). Defaults to empty.

See the **init.pp** file for more details.

#### Facts

The module generates a fact called **barman_key** which has the content of
**/var/lib/barman/.ssh/id_rsa.pub**, in order to automatically handle the
key exchange on the postgres server via puppetdb.

If the file doesn't exist, a key will be generated.

### barman::settings

The barman::settings class set the configuration parameters to set up Barman
server. Here are included parameters specifically for Barman that are not shared
for other resources, such as PostgreSQL server (in this case the rest of the
parameters can be set as it was shown above when a barman instance is defined).

See the **settings.pp** file for more details.

#### Parameters

* **user** - The Barman user. The default value is 'barman'.
* **group** - The group of the Barman user. The default
              value is 'barman'.
* **dbuser** - The user used by Barman to connect to
               PostgreSQL database(s). It will be used to
               build the 'conninfo' Barman parameter.
               The default value is 'barman', and will be
               the same for all the PostgreSQL servers.
* **dbname** - The database where Barman can connect. It will
               be used to build the 'conninfo' Barman parameter.
               The default one is the 'postgres' database.
* **home** - The Barman user home directory. The default
             value is '/var/lib/barman', but it can be changed
             depending on the operating system.
* **autoconfigure** - This is the main parameter to enable the
                      autoconfiguration of the backup of a
                      given postgreSQL server performed by
                      Barman.

### barman::server

The barman::server class configures barman to handle backups for a PostgreSQL server.

The only required parameters are **conninfo** and **ssh_command**.

Example:

    barman::server { 'main':
      conninfo    => 'user=postgres host=server1 password=pg123',
      ssh_command => 'ssh postgres@server1',
    }

Overriding global configuration is supported for most of the parameters.

Example:

    barman::server { 'main':
      conninfo    => 'user=postgres host=server1 password=pg123',
      ssh_command => 'ssh postgres@server1',
      compression => 'bzip2',
      pre_backup_script = '/usr/bin/touch /tmp/started',
      post_backup_script = '/usr/bin/touch /tmp/stopped',
      custom_lines = '; something'
    }

#### Parameters

Many of the main configuration parameters can be passed in order to
perform overrides.

* **conninfo** - Postgres connection string. **Mandatory**.
* **ssh_command** - Command to open an ssh connection to Postgres. **Mandatory**.
* **compression** - Compression algorithm. Uses the global configuration
                   if false (default).
* **pre_backup_script** - Script to launch before backups. Uses the global
                          configuration if false (default).
* **post_backup_script** - Script to launch after backups. Uses the global
                          configuration if false (default).
* **custom_lines** - Custom configuration directives (e.g. for custom
                     compression). Defaults to empty.

## Autoconfiguration

It is possible to enable the barman Puppet module to automatically configure the
Barman server to back up a given PostgreSQL server. It is also possible the
configuration of more than one PostgreSQL server to be backed up, and moreover it
is possible to create many "host groups" when a Barman Server (in each group) can
back up one (or more) PostgreSQL server(s).

### enable autoconfigure

The parameter **barman::settings::autoconfigure** in the **barman** class enables
the inclusion of the Puppet classes involved in the autoconfiguration. The default
value is 'false'.

The parameter **barman::settings::host_group** in the **barman** class is used to
create different host groups. If the same value for this parameter is used on PostgreSQL
server(s) and on Barman server then those server are clusterised as into the same
"host group".

Those are the classes involved when autoconfiguration is enabled:

### barman::autoconfigure

This class:

* Create the .pgpass file for the 'barman' user
* Import resources exported by PostgreSQL server (crontab for the backup, PostgreSQL
superuser ssh key, .pgpass file, configuration of the single PostgreSQL server in Barman)
* Export Barman resources to the PostgreSQL server (*archive_command*, Barman user ssh key,
configurations for the *pg_hba.conf* file)

More details in the **autoconfigure.pp** file.

#### Parameters

* **host_group** - Tag the different host groups for the backup
                   (default value is set from the 'settings' class).

### barman::postgres

This class export resources to the Barman server (Barman configurations,
cron, ssh key) and import resources from it (configure *archive_mode*,
define user used by Barman to connect into PostgreSQL database(s)). It
has to be included in the PostgreSQL server.

More details in the **postgres.pp** file.

#### Parameters

* **host_group** - Tag the different host groups for the backup
                   (default value is set from the 'settings' class).
* **wal_level** - Configuration of the *wal_level* parameter in the postgresql.conf
                  file. The default value is 'archive'.
* **barman_user** - Definition of the 'barman' user used in Barman 'conninfo'. The
                    default value is set from the 'settings' class.
* **barman_dbuser** - Definition of the user used by Barman to connect to the
                      PostgreSQL database(s) in the 'conninfo'. The default value is
                      set from the 'settings' class.
* **barman_home** - Definition of the barman home directory. The default value
                    is set from the 'settings' class.
* **backup_mday** - Day of the month set in the cron for the backup schedule.
                    The default value (undef) ensure daily backups.
* **backup_wday** - Day of the week set in the cron for the backup schedule.
                    The default value (undef) ensure daily backups.
* **backup_hour** - Hour set in the cron for the backup schedule. The default
                    value is 04:XXam.
* **backup_minute** - Minute set in the cron for the backup schedule. The default
                      value is for XX:00am
* **password** - Password used by Barman to connect to PosgreSQL. The default
                 value (empty string) allows the generation of a random password.
* **server_address** - The whole fqdn of the PostgreSQL server used in Barman
                       'ssh_command' (automatically configured by Puppet).
* **postgres_server_id** - Id of the PostgreSQL server, given by its host name
                           (automatically configured by Puppet).
* **postgres_user** - The PostgreSQL user used in Barman *ssh_command*.

## License

This module is distributed under GNU GPLv3

## Author

* Giuseppe Broccolo <giuseppe.broccolo@2ndQuadrant.it>
* Giulio Calacoci <giulio.calacoci@2ndQuadrant.it>
* Francesco Canovai <francesco.canovai@2ndQuadrant.it>
* Marco Nenciarini <marco.nenciarini@2ndQuadrant.it>
* Gabriele Bartolini <gabriele.bartolini@2ndQuadrant.it>

Many thanks to Alessandro Franceschi <al@lab42.it>

### Past authors

* Alessandro Grassi <alessandro.grassi@devise.it>

### Copyright

Copyright 2012-2014 2ndQuadrant Italia (Devise.IT SRL)
