# Kamailio v5.3 - Install Guide

*Guide to install Kamailio SIP Server v5.3 (stable) from Git repository.*
For more about Kamailio Project visit: [kamailio.org](https://www.kamailio.org).

    Main author:
    Daniel-Constantin Mierla

    Support: <sr-users@lists.kamailio.org>

## 1. Overview

This is a step by step tutorial about how to install and maintain Kamailio SIP
server version 5.3.x using the sources downloaded from GIT repository -
the choice for those willing to write code for Kamailio or to try the new
features to be released in the future with the next major stable version.
This document focuses on Kamailio v5.3.x with MySQL support, using a Debian stable system.

## 2. Prerequisites

To be able to follow the guidelines from this document you need `root` access.
The following packages are required before proceeding to the next steps.

- *git* client: `apt-get install git-core` - it is recommended to have a recent
  version, if your Linux distro has an old version, you can download newer one
  from [git-scm.com](http://git-scm.com)
- *gcc* and *g++* compilers: `apt-get install gcc g++`
- *flex* - `apt-get install flex`
- *bison* - `apt-get install bison`
- *libmysqlclient-dev* - `apt-get install libmysqlclient-dev` (or: `apt install default-libmysqlclient-dev`)
- *make* and *autoconf* - `apt-get install make autoconf`
- if you want to enable more modules, some of them require extra libraries:
  - *libssl* - `apt-get install libssl-dev`
  - *libcurl* - `apt-get install libcurl4-openssl-dev`
  - *libxml2* - `apt-get install libxml2-dev`
  - *libpcre3* - `apt-get install libpcre3-dev`

*Important Note*: starting with version `4.3.0`, Kamailio uses the directory
*/var/run/kamailio/* for creating FIFO and UnixSocket control files. You may have
to complete the section related to installation of `init.d` script for creating
`/var/run/kamailio` even if you plan to start Kamailio manually from command line.
The alternative is to set different paths via parameters of *jsonrpcs*
and *ctl* modules.
*Note*: *g++* compiler is needed for couple of modules that link to C++ libraries,
such as app_sqlang, phonenum or ndb_cassandra.

### 2.1 MySQL Or MariaDB Server

To complete all the steps in this tutorial, it is required to have a *MySQL* or *MariaDB* server installed.
Consult the documentation of *MySQL* or *MariaDB* server for Debian for a proper installation.
For testing purposes, it can just be done with `apt-get install mysql-server` or `apt-get install default-mysql-server`.
During or after installation you may have to complete some configuration steps, such as setting the password for mysql root user or initialize the database system.

## 3. Getting Sources From GIT

First of all, you have to create a directory on the file system where the sources
will be stored.

    mkdir -p /usr/local/src/kamailio-5.3
    cd /usr/local/src/kamailio-5.3

Download the sources from GIT using the following commands.

    git clone --depth 1 --no-single-branch https://github.com/kamailio/kamailio kamailio
    cd kamailio
    git checkout -b 5.3 origin/5.3

*Note: if your git client version does not support --no-single-branch command line parameter, then just remove it*

## 4. Tuning Makefiles

The first step is to generate build config files.

    make cfg

Next step is to enable the MySQL module. Edit *modules.lst* file:

    nano -w src/modules.lst
    # or
    vim src/modules.lst

Add *db_mysql* to the variable *include_modules*.

    include_modules= db_mysql

Save the *modules.lst* and exit.
*NOTE*: this is one mechanism to enable modules which are not compiled by
default, such as lcr, dialplan, presence -- add the modules to
*include_modules* variable inside the *modules.lst* file, like:

    include_modules= db_mysql dialplan

Alternative is to set `include_modules` variable with the list of extra modules
to be included for compilation when building `Makefile` cfg:

    make include_modules="db_mysql dialplan" cfg

*NOTE*: If you want to install everything in one directory (so you can delete
all installed files at once), say `/usr/local/kamailio-5.3`, then set `PREFIX`
variable to the install path in `make cfg ...` command:

    make PREFIX="/usr/local/kamailio-5.3" include_modules="db_mysql dialplan" cfg

More hints about `Makefile` system at:

- [kamailio.org/wiki/devel/makefile-system](https://www.kamailio.org/wiki/devel/makefile-system)

## 5. Compile Kamailio

Once you added the mysql module to the list of enabled modules, you can compile Kamailio:

    make all

You can get full compile flags output using:

    make Q=0 all

## 6. Install Kamailio

When the compilation is ready, install Kamailio with the following command:

    make install

### 6.1 What And Where Was Installed

The binaries and executable scripts were installed in: `/usr/local/sbin`

These are:

- *kamailio* - Kamailio SIP server
- *kamdbctl* - script to create and manage the Databases
- *kamctl* - script to manage and control Kamailio SIP server
- *kamcmd* - CLI - command line tool to interface with Kamailio SIP server

To be able to use the binaries from command line, make sure that
`/usr/local/sbin` is set in `PATH` environment variable. You can check that with
`echo $PATH`. If not and you are using `bash`, open `/root/.bash_profile` and
at the end add:

    PATH=$PATH:/usr/local/sbin
    export PATH

Kamailio modules are installed in:

    /usr/local/lib/kamailio/modules/

Note: On 64 bit systems, `/usr/local/lib64` may be used.
The documentation and readme files are installed in:

    /usr/local/share/doc/kamailio/

The man pages are installed in:

    /usr/local/share/man/man5/
    /usr/local/share/man/man8/

The configuration file was installed in:

    /usr/local/etc/kamailio/kamailio.cfg

*NOTE:*: In case you set the PREFIX variable in `make cfg ...` command, then
replace */usr/local* in all paths above with the value of `PREFIX` in order to
locate the files installed.

## 7. Create MySQL Database

To create the `MySQL` database, you have to use the database setup script.
First edit *kamctlrc* file to set the database server type:

    nano -w /usr/local/etc/kamailio/kamctlrc

Locate `DBENGINE` variable and set it to `MYSQL`:

    DBENGINE=MYSQL

You can change other values in *kamctlrc* file, at least it is recommended to
change the default passwords for the users to be created to connect to database.
Note that the existing line with `DBENGINE` or other attributes may be commented,
uncomment by removing the `#` character at the beginning of the line.
Once you are done updating *kamctlrc* file, run the script to create the
database used by Kamailio:

    /usr/local/sbin/kamdbctl create

You can call this script without any parameter to get some help for the usage.
You will be asked for the domain name Kamailio is going to serve (e.g.,
`mysipserver.com`) and the password of the `root` MySQL user. The script will
create a database named `kamailio` containing the tables required by Kamailio.
You can change the default settings in the kamctlrc file mentioned above.
The script will add two users in `MySQL`:

- **kamailio** - (with default password `kamailiorw`) - user which has full
access rights to `kamailio` database

- **kamailioro** - (with default password `kamailioro`) - user which has
read-only access rights to `kamailio` database

*IMPORTANT: do change the passwords for these two users to something different
that the default values that come with sources.*

## 8. Edit Configuration File

To fit your requirements for the VoIP platform, you have to edit the
configuration file.

    /usr/local/etc/kamailio/kamailio.cfg

Follow the instruction in the comments to enable usage of MySQL. Basically you
have to add several lines at the top of config file, like:

    #!define WITH_MYSQL
    #!define WITH_AUTH
    #!define WITH_USRLOCDB

If you changed the password for the `kamailio` user of MySQL, you have to update
the value for `db_url` parameters.
You can browse [kamailio.cfg](https://github.com/kamailio/kamailio/blob/master/etc/kamailio.cfg)
online on GIT repository.

## 9. Running Kamailio

There are couple of variants for starting/stopping/restarting Kamailio,
the recommended ones being via `init.d` script or `systemd` unit, a matter of
what the Debian OS is configured to use.

### 9.1 Init.d Script

To install the `init.d` script, run in Kamailio source code directory:

    make install-initd-debian

Follow any instructions that may be printed by the above commad.
Then you can start/stop Kamailio using the following commands:

    /etc/init.d/kamailio start
    /etc/init.d/kamailio stop

### 9.2 Systemd Unit

To install the `systemd` unit, run in Kamailio source code directory:

    make install-systemd-debian

Follow any instructions that may be printed by the above commad.
Then you can start/stop Kamailio using the following commands:

    systemctl start kamailio
    systemctl stop kamailio

### 9.3 Kamctl

You may need to edit edit `/usr/local/etc/kamailio/kamctlrc` and set the
`PID_FILE` and `STARTOPTIONS` attributes.
The you can use:

    kamctl start
    kamctl stop

### 9.4 Command Line

Kamailio can be started from command line by executing the binary with specific
parameters. For example:

- start Kamailio

      /usr/local/sbin/kamailio -P /var/run/kamailio/kamailio.pid -m 128 -M 12

- stop Kamailio

      killall kamailio
      #or
      kill -TERM $(cat /var/run/kamailio/kamailio.pid)

## 10. Ready To Rock

Now everything is in place. You can start the VoIP service, creating new
accounts and setting the phones.
A new account can be added using `kamctl` tool via:

    kamctl add username password

If `SIP_DOMAIN` was not set in `kamctlrc` file do one of the following
option.

- run in terminal:

      export SIP_DOMAIN=mysipserver.com
      kamctl add username password

- or edit `/usr/local/etc/kamailio/kamctlrc` and add:

      SIP_DOMAIN=mysipserver.com

and then run again `kamctl add ...` as above.

- or give the username with domain in `kamctl add ...` parameter:

      kamctl add username@mysipserver.com password

Instead of `mysipserver.com` it has to be given the real domain for the SIP service
or the IP address of Kamailio.

## 11. Maintenance

The maintenance process is very simple right now. You have to be user `root` and
execute following commands:

    cd /usr/local/src/kamailio-5.3/kamailio
    git pull origin
    make all
    make install
    /etc/init.d/kamailio restart

Now you have the latest Kamailio v5.3.x running on your system.

### 11.1 When To Update

Notification about GIT commits are sent to the mailing list:
*sr-dev@lists.kamailio.org*. Each commit notification contains the reference
to the branch where the commit has been done. If the commit message contains
the lines:

    Module: kamailio
    Branch: 5.3

then an update has been made to Kamailio 5.3 version and it will be available
to the public GIT in no time.

## 12. Support

Questions about how to use Kamailio and the content of kamailio.cfg can be
addressed via email to:

- [sr-users@lists.kamailio.org](http://lists.kamailio.org/cgi-bin/mailman/listinfo/sr-users)

More documentation resources can be found at:

- [www.kamailio.org/w/documentation](https://www.kamailio.org/w/documentation/)
- [github.com/kamailio/kamailio-wiki](https://github.com/kamailio/kamailio-wiki)

## 13. Contributions

Anyone is welcome to contribute to this document. It is recommended to make a
pull request via:

- [github.com/kamailio/kamailio-wiki/pulls](https://github.com/kamailio/kamailio-wiki/pulls)

This version of the document is in GIT branch `master`.
Errors and other issues can be reported via the tracker at:

- [github.com/kamailio/kamailio-wiki/issues](https://github.com/kamailio/kamailio-wiki/issues)
