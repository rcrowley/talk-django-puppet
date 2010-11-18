!SLIDE bullets

# Deploying Django with Puppet



!SLIDE bullets

# Hi, I&#8217;m Richard Crowley

* Equal opportunity technology hater.
* DevStructure&#8217;s operator and UNIX hacker.



!SLIDE bullets

# Installing Puppet

	@@@ sh
	apt-get install build-essential \
		ruby ruby-dev rubygems

	gem install puppet puppet-pip

* <https://github.com/rcrowley/puppet-pip>



!SLIDE bullets

# But first, resources

* The smallest unit of configuration.
* Have a *type*, a *name*, and *parameters*.



!SLIDE bullets small
.notes Even in Ubuntu Maverick, `pip` is out-of-date (0.7.2).

# Bootstrap a Python environment

	@@@ puppet
	stage { "pre": before => Stage["main"] }
	class python {
		package {
			"build-essential": ensure => latest;
			"python": ensure => "2.6.6-2ubuntu1";
			"python-dev": ensure => "2.6.6-2ubuntu1";
			"python-setuptools": ensure => installed;
		}
		exec { "easy_install pip":
			path => "/usr/local/bin:/usr/bin:/bin",
			refreshonly => true,
			require => Package["python-setuptools"],
			subscribe => Package["python-setuptools"],
		}
	}
	class { "python": stage => "pre" }



!SLIDE bullets

# Declare dependencies

	@@@ puppet
	package {
		"django":
			ensure => "1.2.3",
			provider => pip;
		"mysql-python":
			ensure => "1.2.3",
			provider => pip;
	}



!SLIDE bullets

# A bug!

	@@@ puppet
	package {
		"django":
			ensure => "1.2.3",
			provider => pip;
		"mysql-python":
			ensure => "1.2.3",
			provider => pip;
	}



!SLIDE bullets

# Declare *all* dependencies

	@@@ puppet
	package {
		"django":
			ensure => "1.2.3",
			provider => pip;
		"libmysqlclient-dev":
			ensure => "5.1.49-1ubuntu8.1";
		"mysql-python":
			ensure => "1.2.3",
			provider => pip;
	}



!SLIDE bullets

# Nondeterminism!

	@@@ puppet
	package {
		"django":
			ensure => "1.2.3",
			provider => pip;
		"libmysqlclient-dev":
			ensure => "5.1.49-1ubuntu8.1";
		"mysql-python":
			ensure => "1.2.3",
			provider => pip;
	}



!SLIDE bullets

# Declare interdependencies

	@@@ puppet
	package {
		"django":
			ensure => "1.2.3",
			provider => pip;
		"libmysqlclient-dev":
			ensure => "5.1.49-1ubuntu8.1";
		"mysql-python":
			ensure => "1.2.3",
			provider => pip,
			require =>
				Package["libmysqlclient-dev"];
	}



!SLIDE bullets

# Why all the fuss?

* Python packages are ignorant of<br />lower-level dependencies.
* For example, `libmysqlclient-dev`.



!SLIDE bullets

# &#8220;Deploy&#8221; to dev

	@@@ sh
	# As root!

	export GEMS="/var/lib/gems/1.8/gems"
	export RUBYLIB=$GEMS/"puppet-pip-0.0.1/lib"

	puppet apply deps.pp

* You are developing on Linux, aren&#8217;t you?



!SLIDE bullets

# Deploy!

* (For real.)



!SLIDE bullets

# First, we&#8217;ll need Fabric

	@@@ puppet
	package { "fabric":
		ensure => "0.9.3",
		provider => pip,
	}



!SLIDE bullets small

# Deploy!

	@@@ python
	from fabric.api import *

	env.hosts = ['example.com']

	def puppet():
		put('deps.pp', 'deps.pp')
	    sudo('RUBYLIB={0} puppet apply {1} deps.pp'.format(
			'/var/lib/gems/1.8/gems/puppet-pip-0.0.1/lib',
			'--templatedir=.',
		))



!SLIDE bullets

# Deploy!

* `fab puppet`



!SLIDE bullets

# If dev is like production...

* More can be automated.
* Less can vary.



!SLIDE bullets

# If you specify exact version numbers...

* You won&#8217;t suffer at the hands of someone else&#8217;s lousy QA.



!SLIDE bullets

# Iterate

	@@@ puppet
	package { "south":
		ensure => "0.7.2",
		provider => pip,
	}



!SLIDE bullets

# Iterate

* `puppet apply deps.pp`
* `fab puppet`



!SLIDE bullets

# If make changes to the environment incrementally...

* Failures will have a more obvious cause.
* Application features won't be blocked by environmental changes.



!SLIDE bullets

# We need a grown up server

	@@@ puppet
	package {
		"apache2-mpm-worker":
			ensure => "2.2.16-1ubuntu3";
		"libapache2-mod-wsgi":
			ensure => "3.2-2";
	}



!SLIDE bullets smaller

# Config files

	@@@ puppet
	file {
		"/etc/apache2/sites-available/mysite":
			content => template("mysite.erb"),
			ensure => file,
			require => Package["apache2-mpm-worker"];
		"/etc/apache2/sites-enabled/001-mysite":
			ensure => "/etc/apache2/sites-available/mysite",
			require => Package["apache2-mpm-worker"];
		"/etc/apache2/sites-enabled/000-default":
			ensure => absent,
			require => Package["apache2-mpm-worker"];
		"/usr/local/share/wsgi/mysite/mysite.wsgi":
			content => template("mysite.wsgi"),
			ensure => file;
	}



!SLIDE bullets smaller

# `mysite.erb`

	<VirtualHost *:80>
		DocumentRoot /usr/local/share/wsgi/mysite/media
		Alias /media /usr/local/share/wsgi/mysite/media
		WSGIScriptAlias / /usr/local/share/wsgi/mysite/mysite.wsgi
		WSGIDaemonProcess mysite processes=2
		WSGIProcessGroup mysite
	</VirtualHost>



!SLIDE bullets smaller

# `mysite.erb` with one process per core

	<VirtualHost *:80>
		DocumentRoot /usr/local/share/wsgi/mysite/media
		Alias /media /usr/local/share/wsgi/mysite/media
		WSGIScriptAlias / /usr/local/share/wsgi/mysite/mysite.wsgi
		WSGIDaemonProcess mysite processes=<%= processorcount %>
		WSGIProcessGroup mysite
	</VirtualHost>



!SLIDE bullets small

# `mysite.wsgi`

	@@@ python
	import os
	import sys.path

	os.environ['DJANGO_SETTINGS_MODULE'] = 'mysite.settings'
	sys.path.append('/usr/local/share/wsgi')

	import django.core.handlers.wsgi
	application = django.core.handlers.wsgi.WSGIHandler()



!SLIDE bullets

# Does `mysite.wsgi` really belong in Puppet?

* Environment versus application.
* Relatively stable versus wild and crazy.
* Push versus pull.



!SLIDE bullets

# Danger: my opinion

* Environment: pull packages, web server config, DNS, and monitoring from Puppet.
* Application: push code, including `mysite.wsgi`, with Fabric.



!SLIDE bullets small

# Automatically reload changes

	@@@ puppet
	service { "apache2":
		enable => true,
		ensure => running,
		require => Package["apache2-mpm-worker"],
		subscribe => [
			Package[
				"apache2-mpm-worker",
				"libapache2-mod-wsgi"],
			File[
				"/etc/apache2/sites-available/mysite",
				"/etc/apache2/sites-enabled/001-mysite",
				"/etc/apache2/sites-enabled/000-default",
				"/usr/local/share/wsgi/mysite/mysite.wsgi"]],
	}



!SLIDE bullets

# Apache for all

* `puppet apply --templatedir=. deps.pp`
* `fab puppet`



!SLIDE bullets

# Puppet master and agents

* Master knows best.
* Agents pull their configuration.
* Resources can be exported to other nodes.



!SLIDE bullets

# `manifests/nodes.pp`

	@@@ puppet
	node "example.com" {
		include python
		include mysite
	}



!SLIDE bullets small

# `modules/python/manifests/init.pp`

	@@@ puppet
	stage { "pre": before => Stage["main"] }
	class python {
		package {
			"build-essential": ensure => latest;
			"python": ensure => "2.6.6-2ubuntu1";
			"python-dev": ensure => "2.6.6-2ubuntu1";
			"python-setuptools": ensure => installed;
		}
		exec { "easy_install pip":
			path => "/usr/local/bin:/usr/bin:/bin",
			refreshonly => true,
			require => Package["python-setuptools"],
			subscribe => Package["python-setuptools"],
		}
	}
	class { "python": stage => "pre" }



!SLIDE bullets small

# `modules/mysite/manifests/init.pp`

	@@@ puppet
	class mysite {
		include python
		package {
			"django":
				ensure => "1.2.3",
				provider => pip;
			# Et cetera.
		}
		file {
			# Prefix template paths with the module name.
			"/etc/apache2/sites-available/mysite":
				content => template("mysite/mysite.erb"),
				ensure => file,
				require => Package["apache2-mpm-worker"];
			# Et cetera.
		}
		service {
			"apache2": # As above.
		}
		# Other resources as you like.
	}



!SLIDE bullets

# Thank you

* <https://gist.github.com/701221>

* <richard@devstructure.com> or [@rcrowley](http://twitter.com/rcrowley)
* P.S. use DevStructure.
