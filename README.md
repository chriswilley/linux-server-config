# Linux Server Configuration

This README specifies how I configured my Linux VM for project #5.


## Table of contents

* [Connection info](#connection-info)
* [Software installed](#software-installed)
* [Other configuration changes](#other-configuration-changes)
* [Resources used](#resources-used)


## Connection info

Item | Value
--- | ---
IP | 54.149.57.47
SSH Port | 2200
URL | http://ec2-54-149-57-47.us-west-2.compute.amazonaws.com/


## Software installed

### ntpd

NTP server to ensure the host maintains the correct time. No changes from default config; default timezone already set to UTC.

### Nginx

Reverse proxy for apache and static file server. I chose to use Nginx because it's a good way to help secure apache from attack and it is also very efficient at serving static files. In this configuration, Nginx receives requests from port 80 and passes them to Apache on port 8080 after performing some checks to help ensure dangerous requests don't hit the web server (such as very large requests intended to bring down the server). Since port 8080 is not open on the firewall, Apache itself is not directly accessible from the Internet.

The main Nginx configuration file is located at /etc/nginx/nginx.conf. I kept the default settings in this file.

I added a config file for the catalog application, placing it in /etc/nginx/sites-available/catalog and creating a symlink in /etc/nginx/sites-enabled/. I removed the default symlink from that folder as well. I added location blocks in the file for /static (JavaScript, CSS, project images) and /media (uploaded user images) to serve files in those folders from Nginx rather than Apache to improve performance. I've placed comments in this config file for reference.

### apache/mod_wsgi

Primary web server and WSGI server for flask to run on. Modified main Listen parameter in /etc/apache2/ports.conf from:

```
Listen 80
```

to:

```
Listen 8080
```

due to the use of Nginx (see previous section).

I modified /etc/apache2/apache2.conf in two places:

```
ServerName ec2-54-149-57-47.us-west-2.compute.amazonaws.com

...

<Directory /usr/share>
        AllowOverride None
        Require all denied
</Directory>
```

The first change ensured Apache knows what it's own FQDN is. The second change provides some additional security since the web server will have no need to serve files from /usr/share.

I added a site config site in /etc/apache2/sites-available/ called catalog.conf. The settings in this file are essentially those laid out in flask's documentation on [deploying to production](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/). I created a new user under which the application would run and added that info to the config file. I also specified that the VirtualHost would respond to all requests for port 8080, and also redirected the log files to the project folder.

I added comments to the host config file for reference. I also added a symlink to the file in /etc/apache2/sites-enabled, and removed the default symlink. Finally, I created a WSGI file in /var/www/catalog to provide the mechanism for Apache to be aware of the application (which would be stored elsewhere) as well as set up the virtual runtime environment (see below).

### mod_security

Even though Nginx is protecting Apache and filtering a lot of traffic before it hits the web server, I felt it was still a good idea to install additional modules to further safeguard Apache. One standard module is [mod_security](https://www.modsecurity.org/), a free and open source web application firewall. It does a lot of the same things fail2ban does (see below), but a wider community of security experts have written rules for mod_security.

I installed the module following the docs and some resources found online. I made a small number of changes to the /etc/modsecurity/modsecurity.conf:

* I set SecResponseBodyAccess to Off on the recommendation of several online articles
* I increased the SecRequestBodyLimit to match the setting used in my application (32M); otherwise user uploaded images might get rejected
* Once I had tested browsing the application, I set the SecRuleEngine setting to On to activate blocking

I activated some of the rules in "base rules" (located in /usr/share/modsecurity-crs/base_rules/) based on what seemed to be a good baseline of coverage after reading the docs and source code. To do this, I edited the /etc/apache2/mods-enabled/modsecurity.conf and added the following lines:

```
Include /usr/share/modsecurity-crs/*.conf
Include /usr/share/modsecurity-crs/activated_rules/*.conf
```

The first line ensured that the main rules config file would be included. The second line looks for additional rules in the activated_rules folder. I then created symbolic links to the rules I wanted to activate in that folder. I then restarted Apache and tested the application.

### mod_evasive

I also installed the [mod_evasive](http://www.zdziarski.com/blog/?page_id=442) Apache module. This module helps mitigate DOS attacks on Apache servers. I followed the advice of several online articles for the config (in /etc/apache2/mods-available/evasive.conf):

```
<IfModule mod_evasive20.c>
    DOSHashTableSize    3097
    DOSPageCount        2
    DOSSiteCount        50
    DOSPageInterval     1
    DOSSiteInterval     1
    DOSBlockingPeriod   10

    DOSEmailNotify      root@localhost
</IfModule>
```

These settings (see [this site](https://www.linode.com/docs/websites/apache-tips-and-tricks/modevasive-on-apache/) for definitions of the parameters) seemed like a good baseline configuration, and in a real production scenario I would probably keep an eye on performance and tweak these if needed.

### python/pip

With the web server and reverse proxy in place, I could begin to deploy the application. For starters, I installed Python and pip. No special configuration was made to either package.

### git

I then installed git so I could clone the application to the server.

### postgresql

I then installed PostgreSQL server. The project specified that PostgreSQL should be configured not to allow external connections; as that is the default, I did not need to make any modifications. I also created a new role in the database server with the same name as the Linux user specified in the Apache settings above, and gave the user a complex password. Finally, I created the database for the application; the database role has permissions only to the application database.

### virtualenv/virtualenvwrapper

I decided to have the application run in a virtual environment. My rationale was that if I decided to deploy other flask applications to the server, I might want a different set of python modules (or different versions of the same modules) than those in the catalog app. Using virtualenv makes this possible. Virtualenvwrapper is a convenience tool to make working with virtual environments a little easier.

After installing the packages using pip, I added parameters to my user's .bashrc file:

```
export WORKON_HOME=/home/[PROJECT LINUX USER]/.virtualenvs
export PROJECT_HOME=/home/[PROJECT LINUX USER]/Projects
source /usr/local/bin/virtualenvwrapper.sh
```

I used the actual user name in place of the brackets, but for security reasons I'm not showing that username here.

These settings allowed me to create the virtual environment for the application and have the Python site-packages folder placed in the application user's home directory. I also created a folder called Project under the application user's home directory to house the application files themselves.

I then git cloned the application into the Project folder. By placing it completely outside the area Apache uses, I provided some additional security (so that, for example, a user could not browse the .git folder within the application).

The project WSGI file in /var/www/catalog sets up the runtime virtual environment and provides directory references to Python site-packages and the proper application root folder.

### project dependencies

With the project files and folders on the server, I then installed all the dependencies using:

```
pip install -r requirements.txt
```

I found that a few additional things were needed before this ran completely through, including:

* python-dev
* python-psycopg2
* libpq-dev
* libjpeg-dev

With those installed the dependencies all came through fine. I then ran setup.py in the application root folder, which establishes the database tables and some initial data.

Finally, I set up the flask instance config file with details like the PostgreSQL connection settings and API keys for Google and Facebook sign-in. I then tested the application in a browser to make sure it all worked.

### monit

I chose to install [monit](https://mmonit.com/monit/), a free Linux service monitoring and alerting tool. Although there are dozens of available options out there I have used this application before, and feel comfortable with it.

I mostly kept the default primary configuration (located in /etc/monit/monitrc), but added information for the mail server (Postfix, see below) monit would use to send alerts as well as the alert recipient email address. I also enabled the web server for monit; I did this because it's required for running command-line queries like:

```
sudo monit status
```

to show the status of all the running monit probes. Because the firewall is blocking the port monit would use, the monit web server is not accessible from the internet. Just to be safe, I added a complex password for the monit web interface user.

I uncommented the lines in the primary config file related to host status checks.

Following best practices, I copied some of the sample config files from /etc/monit/monitrc.d into /etc/monit/conf.d:

* apache2
* fail2ban
* nginx
* ntpd
* openssh-server
* postfix
* postgresql

Within each of these service config files I made changes based on my own needs. I added comments in the files for reference.

### cron-apt

[Cron-apt](http://manpages.ubuntu.com/manpages/trusty/man8/cron-apt.8.html) is a small utility for running nightly checks for updated Linux packages using apt-get. By default, the tool updates the local package repository and downloads new packages but does not install them. I kept this default behavior because over time it becomes difficult to predict how package updates will affect running applications. If you just apply all updates, applications you're hosting on the server can suddenly break after a particular package changes as the result of an update. Based on my own experience, it's better to apply updates manually. Cron-apt makes this easier by auto-downloading updates and alerting sysadmins to what has an update pending.

The utility emails the Linux admin with the result of the previous night's update check; if there are updates that look important, the admin can just run

```
sudo apt-get dist-upgrade
```

to upgrade all installed packages that have updates, or

```
sudo apt-get install --only-upgrade <packagename>
```

to update just one package. I did not make any changes to the configuration of this tool except to add

```
MAILON="always"
```

to the /etc/cron-apt/config file so that the utility would email all messages to the Linux admin rather than just errors.

### fail2ban

[Fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page) is a program written in Python that watches network traffic to the host and blocks requests from IP addresses that match user-defined filters. IP addresses that are blocked are placed in a "jail" that prevents them from making further requests for a specified period of time. Fail2ban by default uses iptables to jail IPs.

Fail2ban watches application log files for repeated requests that match the user-defined filters (written as regular expressions). Its purpose is to derail DOS attacks and other automated intrusion attempts (like brute force attempts to guess a password for SSH access).

Like monit, fail2ban works with user-defined config files (called filters). I made no changes to the program default config file located at /etc/fail2ban/fail2ban.conf. Following best pactices, I copied /etc/fail2ban/jail.conf to /etc/fail2ban/jail.local and made service changes to that file. The jail config file specifies the log files fail2ban should monitor for a given service definition as well as the port number to check. For example, the default ssh jail config is:

```
[ssh]

enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 6
```

This jail definition is enabled, checks requests to the SSH port, uses the filter named "sshd" and checks the /var/log/auth.log file. The "maxretry" parameter is how many times in a specified interval (600 seconds by default) fail2ban should allow before jailing the IP address.

The filters are located in /etc/fail2ban/filter.d/. You can add additional filter files to this folder if needed, but the ones provided are a good start for many situations. By default, every filter definition is disabled and you can enable the ones you wish fail2ban to use. I enabled the following:

* apache-common
* postfix.conf
* nginx-http-auth.conf
* sshd.conf

I also added a few of my own to further protect Nginx:

* nginx-badbots.conf
* nginx-nohome.conf
* nginx-noscript.conf

I added comments to the filter files for reference. I also changed some of the default settings in jail.local; specifically:

* I set the destemail parameter to the root alias, so that I would receive email messages from fail2ban
* I set the mta parameter to 'mail' so that fail2ban would use the mail command rather than sendmail
* I changed the action parameter to 'action_mwl' which bans an IP and emails the destemail address with whois information for the banned IP as well as a copy of the log file entry written to fail2ban.log

### postfix

Since I wanted email alerts from the various monitoring and security tools I installed, I needed an email server. I chose to install [Postfix](http://www.postfix.org/) because it was free and easier to set up than sendmail. It was so easy I only had to change one line in /etc/postfix/main.cf:

```
inet_interfaces = loopback-only
```

I changed this in order to prevent Postfix from relaying mail from an outside host. Between this and the fact that the firewall blocks the default SMTP port (25), it's highly unlikely that a spammer could use this mail server as an open relay.

I also added an alias to /etc/aliases for root and student (the sudoer account I use to login) to my own personal email address. That way mail to root@localhost or just root will be forwarded to my account.

### autolog

I installed [autolog](http://manpages.ubuntu.com/manpages/trusty/man8/autolog.8.html) to terminate idle SSH sessions rather than keeping them active indefinitely. I made minimal config changes (in /etc/autolog.conf), just adding lines for each of the SSH capable users to log them off after 15 minutes of idle time.

### AIDE

On the advice of a security scan I ran, I installed several tools including [AIDE](https://help.ubuntu.com/community/FileIntegrityAIDE) (Advanced Intrusion Detection Environment). AIDE is a file integrity monitoring tool that, as its name implies, attempts to identify and warn of intrusions into the operating system. It does this by maintaining a database of all files in the filesystem and then checking the files periodically to make sure they match the database. Discrepancies are logged and can be emailed to a sysadmin.

I basically kept the default config (in /etc/aide/aide.conf and /var/lib/aide/aide.conf) after installing the tool and setting up the database. Having this additional layer of security is comforting.

### sysstat

Another useful tool recommended by the security scan is [sysstat](http://sebastien.godard.pagesperso-orange.fr/). This gathers system performance stats and provides a number of useful reports to help sysadmins find bottlenecks and other performance problems. For example, running

```
sar -u
```

will show system load information for the current day in ten-minute intervals up to the current time. There are many other report options. The reports can be output in CSV for importing into a database.

I kept the default config for sysstat (in /etc/sysstat/sysstat). The program set up daily cron jobs to run the statistical data gathering on its own.

### auditd

The [auditd](http://manpages.ubuntu.com/manpages/trusty/man8/auditd.8.html) utility keeps a comprehensive auditing record of every action taken on a server for later forensic analysis. In addition, you can specify rules that cause auditd to watch specific files or commands and log their activity. This is a great tool for investigating an issue after the fact at a deep system level; over time as you become more familiar with a server's activity you can tweak the audit rules for your needs.

I installed this utility and set it to run as a service, keeping the default configuration (in /etc/audit/auditd.conf).

### acct

In much the same way that auditd and AIDE watch file system activity, [acct](http://manpages.ubuntu.com/manpages/trusty/en/man5/acct.5.html) keeps a comprehensive record of user activity. The acct utility maintains a database of user activity and provides command line tools to query the data. For example:

```
sudo sa
```

will display a summary of commands executed by users and shows the amount of cpu time used by each command. The utility also records commands executed by system processes running as users, so you can track down problems that might be caused by cron jobs or other automated processes.

### ClamAV

Although malware is not as common on linux systems as with Windows, it is stil recommended to use malware scanners for production systems. One such application is [ClamAV](http://www.clamav.net/). This works much like a virus scanner for a PC, checking files on access and maintaining a database of known virus patterns. I installed the daemon verion of the Clam scanner as well as an automatic updater for virus definitions. I kept the default config (found in /etc/clamav/clamd.conf).

### chkrootkit

In the same spirit as ClamAV, I installed [chkrootkit](http://www.chkrootkit.org/) in order to keep an eye on potential malware infecting the system. The utility creates a daily cron job for itself, which you must enable in /etc/chkrootkit.conf in the first line:

```
RUN_DAILY="true"
```

I also modified the daily cron job to email me with the results of the scan.


## Other configuration changes

### user accounts

I added an account with sudo access for myself to use, and one for the Udacity grader. I also added a Linux account to control the catalog application, as noted above. This account does not have sudo access.

I granted sudo access to the other two accounts by adding config files for them to /etc/sudoers.d/ and restricting permissions to 0600 on those files.

Although password authentication is not allowed in ssh, I ensured the passwords for all accounts was complex (including letters, numbers, symbols, etc).

### ssh

Following best practices, I changed the default SSH port from 22 to 2200. I did this by altering the Port line in /etc/ssh/sshd_config to:

```
Port 2200
```

I also disabled the ability for root to use ssh by adding:

```
PermitRootLogin no
```

to the same config file. All other default settings - such as disabling password-based logins - were already the way I wanted them.

### ufw

Using the "allow" command I added ufw firewall rules for SSH (port 2200), NTP and HTTP:

```
sudo ufw allow 2200/tcp
sudo ufw allow 123/tcp
sudo ufw allow http
```

As noted earlier, Nginx responds to port 80 while Apache sits behind it on port 8080 (which is not allowed through the firewall).

I then enabled the firewall using

```
sudo ufw enable
```

I also modified the application profiles in /etc/ufw/application.d/ where the ports defined were not accurate. For example, I modified the openssh-server profile as follows:

```
ports=2200/tcp  # was: ports=22/tcp
```

and the /etc/ufw/applications.d/apache2-utils.ufw.profile:

```
ports=8080/tcp  # was: ports=80/tcp
```

### backups

I thought it would be helpful to create backups of the PostgreSQL application database as well as web server logs in case of disaster. Ideally you would probably offload these to another server on a periodic basis so that you can recover if the whole server was unavailable or trashed.

I used the [PostgreSQL wiki](https://wiki.postgresql.org/wiki/Automated_Backup_on_Linux) to put together a rotating backup script for the catalog database. This script performs a pg_dump into both raw SQL and PostgreSQL's custom format for simple restore. In addition, it performs a daily and weekly backup, pruning daily backups every seven days and weekly backups every five weeks. This backup script is located in /home/[PROJECT LINUX USER]/Projects/catalog/backups/bin/.

I also created a script that gzips the logs in the project log folder and dumps them into a logbackup folder. This way the logs can be recovered if needed for later troubleshooting purposes.

### crontab

I added crontab entries for these shell scripts in /etc/crontab to make them run every morning at 1am and 1:30am respectively.

### logrotate

Because I changed the locations of the Nginx and Apache logs related to the catalog application, I modified the service config files in /etc/logrotate.d/. I added comments to those config files for reference.

### swap file

To improve performance, I added a memory swap file as one did not exist by default with the Amazon image. I gave the swap file 500MB of space to help improve system memory usage. The swap file is located at /swapfile1. I also modified /etc/fstab to ensure the swap file would be enabled on system boot.

### security audit

I decided it would be a good idea to run a third party security scanner on the server to see if there were any areas of vulnerability I should address. After some web searching I found [lynis](https://cisofy.com/lynis/), a free security auditing tool. Setup was fairly easy, and after running the audit I found several things that could be made more secure (see some of the items under Software installed).

I should note that several recommendations the scanner made I did not follow because they did not seem relevant (protection against data theft via USB thumb drives on an Amazon EC2 instance didn't seem needed; also items related to login passwords since password authentication is disabled). On the whole though, I found the exercise of running the audit and researching the tools needed to address security concerns very educational. I also improved the "hardening" score from 70 to 80 through my efforts, eliciting the high praise: "System seem to be decent hardened [sic]" from the scanner.


## Resources used

In addition to the links above and the various software docs, I used a lot of resources to help me optimize the server setup. Here are a few of the ones I used:

* [Viktor Petersson's blog](http://blog.viktorpetersson.com/post/92729917189/setting-up-monit-to-monitor-apache-and-postgresql-on-ubu): This helped me setup monit better.
* [n0where.net](https://n0where.net/send-only-postfix-server/): This blog post showed me how to set up an email server to send only and not relay messages.
* [Rob Golding's blog](http://www.robgolding.com/blog/2011/11/12/django-in-production-part-1---the-stack/): Although this post is about Django in production, the overall stack is what I was aiming for (especially the Nginx part). It's also what I've used for a couple of my own Django projects.
* [peatiscoding](http://peatiscoding.me/geek-stuff/mod_wsgi-apache-virtualenv/): This post gave me some insights above and beyond the flask docs regarding Apache and mod_wsgi setup in a virtualenv.
* [DigitalOcean Community](https://www.digitalocean.com/community/tutorials/how-to-protect-an-nginx-server-with-fail2ban-on-ubuntu-14-04): This blog post was HUGE in helping me setup fail2ban to protect Nginx. I ~~stole~~ borrowed from it liberally.
* [This DigitalOcean post](https://www.digitalocean.com/community/tutorials/how-to-set-up-mod_security-with-apache-on-debian-ubuntu) helped me get Apache mod_security running properly.
* [This site](http://www.thegeekstuff.com/2011/03/sar-examples/) was helpful in getting sysstat set up.
* [Blog Overflow](http://security.blogoverflow.com/2013/01/a-brief-introduction-to-auditd/) helped me with setting up auditd.
* [This Tecmint article](http://www.tecmint.com/how-to-monitor-user-activity-with-psacct-or-acct-tools/) had some great info on getting acct installed and working properly.
