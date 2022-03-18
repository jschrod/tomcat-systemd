# Tomcat-Installation With Systemd

While most Linux distributions package Tomcat, there is often the need
to deploy and run an own Apache Tomcat installation. Company policy
might prescribe certain versions, or one needs to replicate a given
server setup on a development system. This project collects a bunch of
configuration files that have been successfully used in such
circumstances.

Owing to systemd restrictions, I cannot create configuration files
that one can use as-is, without changes. Therefore users will have to
take these files and will need to adapt them to their local
installations.

These files currently target Apache Tomcat 9. They have also been used
for Tomact 8 installations, and I'm quite sure that they will be
usable with Tomcat 10 (after adaption of version numbers in file
names).



# Tomcat installation

Within these configuration files, the following Tomcat installation
structure is assumed. (All locations can be changed.)

The Tomcat installation is placed in `/opt/`.

We use two directories there, one for the Tomcat program installation
(called `CATALINA_HOME` in Tomcat documentation) and one for the
actual server instance configuration and webapps (called
`CATALINA_BASE` in Tomcat documentation). IMHO, these two installation
artifacts must always be separated: it allows minor updates of your
Tomcat program which usually require no change of your server instance
configuration. In Tomcat documentation, this is featured as a mean to
run multiple Tomcat instances on one system -- but this is also highly
valuable for any Tomcat deployment, even if there is only on instance
running on a host.


## Tomcat program installation

Tomcat program installation is simply done by unpacking the Tomcat
distribution in `/opt/`. At the time of writing this README, the
current Tomcat distribution was `apache-tomcat-9.0.58.tar.gz`, so this
results in an installation at `/opt/apache-tomcat-9.0.58/`. (Don't
forget to verify previously that your downloaded Tomcat distribution
is uncorrupted by checking XXX.)

We add a symlink `/opt/apache-tomcat9` that points to the installation
directory, here `/opt/apache-tomcat-9.0.58`. This symlink will allow
us later to update to newer Tomcat 9 releases without changing
configuration files. (It is never a good idea to use patch-level
version specifiers in generic configuration files. The danger that one
forgets to update one of these configuration lines while deploying a
new release is simply too high.)

```
puma:/opt # tar -xf /software/java/tomcat/apache-tomcat-9.0.58.tar.gz
puma:/opt # ln -s apache-tomcat-9.0.58 apache-tomcat9
```

The result looks like this:

```
puma:/opt $ ll -d *tomcat*9*
drwxrwsr-x 9 root root 4096 Mar 15 00:32 apache-tomcat-9.0.58
lrwxrwxrwx 1 root root   20 Mar 15 00:32 apache-tomcat9 -> apache-tomcat-9.0.58
```

As you see, the program installation is not owned by an account that
is connected to the Tomcat server. In fact, I would recommend that the
Linux account that the server runs under does **not** own the program
files or directories. (In fact, the output above was edited by me
manually. On our systems, I use a special account to install software
in `/opt/`, since I try to avoid working as `root` whenever possible.)

*Please note:* These commands are only for illustration. I assume that
you use a configuration management system like Ancible, Puppet, or
Chef; and that you will add a configuration to them that causes the
commands named above.


## Preparing the (1st) tomcat server instance

If you run only one Tomcat server on your host, this instance is named
`default`. (That name is the default in systemd -- it may be changed,
but you should have policy or technical reasons to do that. Document
them!)

If you want to run several Tomcat server instances on your host, choose
a sensible name for your 1st instance and use it, instead of `default`,
in the following description.

We need a Linux account that the server will run under. If you
haven't created one already, you'll need to do that now. For the sake
of this documentation, I will use the account `tomcat` that is a
member of the group `tomcat`.

We will now create the 1st server instance:

```
cd /opt
mkdir apache-tomcat9-default
rsync -a apache-tomcat9/{conf,logs,temp,work} apache-tomcat9-default
cd apache-tomcat9-default
mkdir webapps
chown -R tomcat:tomcat logs temp webapp work
```

If you want to use the Tomcat *manager* application, it's recommended
to symlink it. Then you get updates when you deploy a new Tomcat 9
program installation.

```
cd webapp
ln -s ../../apache-tomcat9/webapps/manager .
```

The same can be done for the *host-manager* application, or the
standard Tomcat home page (named *ROOT* in `apache-tomcat9/webapps/` --
but it's not recommended to use it on production systems...)


## Tomcat instance configuration

Per default, your Tomcat instance listens on port 8080 for HTTP
connections, has no TLS (https) defined (which is a bad thing in
itself), listens for shutdown requests on port 8005, etc. This
configuration is in `/opt/apache-tomcat9-default/conf/server.xml`.

Adapt port numbers to your liking, configure TLS, configure
clustering, etc. Tomcat documentation is your friend here,
recommendations are beyond the scope of this project.

If you symlinked Tomcat's applications *manager* or *host-manager*
above, you also have to establish users and roles in
`conf/tomcat-users.xml`. (Please note: Patches to this README are
welcome. Especially I would like to have links to some documentation
of best practice. Bonus points when they don't feature passwords
stored in clear text. I don't use this feature.)

If your application(s) use databases, connection pools must be
configured in `/opt/apache-tomcat9-default/conf/context.xml`. For now,
I leave this topic to Tomcat documentation, as well.

Logging is a bitch. IMNSHO, everyone who thinks logging is easy is a
fool with not enough experience. I don't want to dig in it; so I leave
`conf/logging.properties` as distributed by the Apache Tomcat project.
Nevertheless, one hint: It is advisable to store usage logs somewhere
else. Symlinks are your friend, e.g., replace
`/opt/apache-tomcat9-default/log` with a symlink to
`/var/log/apache-tomcat9-default/` or similar. (The latter directory
must exist and must be owned by the Tomcat run account, in our
configuration example `tomcat:tomcat`.)



# Establish Tomcat in your system

Now, you have a Tomcat installation, but your system knows nothing
about it. Actually, now is the point in time when the configuration
files of this project comes into play.


## Tomcat configuration in `/etc/`

As root, create `/etc/tomcat/`.

### Basic configuration, also usable for one default instance

Place `tomcat9.conf` there and check if you need to adapt it:
* `JAVA_HOME` is the location of your Java installation. You only need
  to set it if your Java installation is not found in the default
  system path.
* If you installed Tomcat program in some other directory, adapt
  `CATALINA_HOME`.
* `CATALINA_BASE` is the directory where the default instance is
  located. This line must only be changed if you use only one Tomcat
  instance on this host *and* have changed the instance installation
  location.
* You may want to set some Java VM options or Java system properties
  in `CATALINA_OPT`. The configuration file contains an example that
  we use for very small setups.

I didn't yet figure out a good strategy for placing Tomcat's PID file.
For now, it's in the Tomcat instance work directory. (If somebody
could provide me with a pull request to integrate this better with
systemd, I would be grateful.)

Another commonly changed Tomcat configuration is the *umask* that
Tomcat uses by default.

### Need To Run Several Tomcat Instances On This Host?

If you want to run several Tomcat instances on this host, then (and
only then!) you'll need to install `tomcat9-INSTANCE.conf` in
`/etc/tomcat/`. In the configuration filename, replace `INSTANCE` with
your chosen instance name.

Adapt the content of this configuration file. Configuration values
from this file override the entries of `/etc/tomcat/tomcat9.conf` --
if you want to use the default from there, don't redefine it with the
same value.


## Systemd service

Copy `tomcat9@.service` to `/etc/systemd/system/`.

If you changed any `/etc/` location in this configuration guide, you
have to change it in this file as well.

If your Tomcat run account is not `tomcat:tomcat`, you'll have to
change that in this file. **But note**: If you have several Tomcat
instances with different run accounts, see below.

If you use only one Tomcat instance on this host, the systemd service
is started with the command `systemctl start tomcat9@default`. Status
of this running Tomcat server is shown by `systemctl status
tomcat9@default`. (Though: The most relevant information given by this
command are the processes of the Tomcat instance. It will not show
startup error messages, these are in the Tomcat log files.)

To start this service on every boot, use `systemctl enable
tomcat9@default`.

`systemctl reload` is not supported by Tomcat.

If you use several Tomcat instances on this host, you will probably
already have guessed the systemd commands to control them: `systemctl
start tomcat9@INSTANCE` where `INSTANCE` is your Tomcat instance name
(i.e., the file name part in `/etc/tomcat/`). Similar for
enabling/disabling.

Some interesting systemd tidbit for usage of several Tomcat instances:
If you need to run every Tomcat instance under a different Linux
account, you can create
`/etc/systemd/system/tomcat9@INSTANCE.d/user.conf` with a systemd
configuration section

```
[Service]
User=tcaccount
Group=tcgroup
```

That will override the default user/group configuration in
`/etc/systemd/system/tomcat9@.service`, but will still use the other
configuration entries. (You don't need to redefine the instance
version of `EnvironmentFile`, it uses a systemd instance specifier
variable!)

Such a config file in a service instance specific systemd
configuration directory may also be used to specify dependencies to
other services. (A schoolbook example would probably be the start of a
database server before Tomcat -- but I have never seen the database
server being run on the same host as the Tomcat server on real
systems, so I consider this example as useless. Maybe the Tomcat
account is from LDAP and one wants to start after `nss-lookup.target`
or similar; that might be a better example.)


# Summary

You will install files in several locations:

```
/opt/apache-tomcat-VERSION		program installation, from Apache download
/opt/apache-tomcat9			symlink to program installation
/opt/apache-tomcat9-INSTANCE		server instance (usually/often named "default")
/etc/tomcat/tomcat9.conf		basic configuration of program installation
/etc/tomcat/tomcat9-INSTANCE.conf	instance configuration (optional)
/etc/systemd/system/tomcat9@.service	systemd service definition
```



# Notes on this setup

This setup tries to integrate common Linux setup principles:
* 3rd party software is installed in `/opt/`
* System configuration files are in `/etc/`
* Systemd service is used, with instance definitions.

That not all Tomcat configuration is done in `/etc/tomcat/` is a
deliberate choice. One could place the files from
`/opt/apache-tomcat9-INSTANCE/conf/` there as well, maybe in an
instance sub-directory. But I want an installation layout that caters
both to Linux conventions, as expected by sysadmins, and to Tomcat
conventions that are well established in many installations. To
achieve that I placed configuration that concerns server startup,
shutdown, and memory usage (controlled by environment variables) in
`/etc/tomcat/`, and Tomcat specific stuff like JDBC resource
definitions in `/opt/apache-tomcat9-INSTANCE/conf/` (controlled by
Tomcat XML configuration files). I welcome discussions about that
choice.

One important point, for me, is the **out-of-the-box support for
separation of Tomcat program and instance installations**. I have
experienced many situations where Tomcat was not updated to the latest
patch release because configuration, webapp, and work files were
intertwined with the program installation and would have to be
transferred to the new installation. In this setup, a new Tomcat
release (e.g., 9.0.54 which fixes the important DoS-Vulnerability
CVS-2021-42340) can be easily deployed by unpacking the Apache Tomcat
9 release in `/opt/`, adjusting the symlink `/opt/apache-tomcat9` and
restarting the system service. I cannot stress enough how important
such easy upgrades are. They can also be defined more easily in
Ansible, Puppet, Chef, or whatever configuration management system you
use today. In fact, that I didn't find this support in published
Tomcat systemd configurations is the primary reason why I established
this project and published it on Github.


# Legalese

All files in this project are in the *Public Domain*.

Do with them as you like, but don't accuse me if they don't work as
you expected.

The formal legal declaration is in LICENSE. applies to all
countries where the legal system has a concept of *Public Domain*.

When I created the Github project, I was presented with a choice of
licences. *Public Domain* was not among them. So I chose the most
fitting license, named *Unlicense License*. In case of legal matters
in a state that recognizes the *Public Domain* concept, that license
is binding.

Some countries have legal systems that don't have the concept of
*Public Domain*. (Actually, I live in one of them. In Germany, it's
not possible to give away a copyright.) I want to make sure my intent
in countries without a *Public Domain* copyright concept: You can use
any file here at your will, as long as you accept the *NO WARRANTY*
clause from LICENSE. If you sue me, your license to use these files is
instantly removed, and I won't reinstantiate it under any
circumstances. Use these files only if you accept that condition.

Since this question has come up in the past: These files are in the
Public Domain. Therefore you are allowed to strip my name and email
address from any file in redistribution or usage. (In fact, if you
change the config files I would recommend to strip the comments with
my name.) In redistributions, you can also delete files that you think
are irrelevant or that you don't like.


# Author

I'm Joachim Schrod, mailto:jschrod@acm.org

I'm a Unix/Linux white beard, literally. If you were active on Usenet
in the late 80s or early 90s, you probably know me.

Concerning FOSS, I'm active in the TeX developer community since 1982
and have been involved in many other free and open source projects.
