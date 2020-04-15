# hume

Hume is an agnostic system administration tool/framework.  It is in fact a
command line tool to call from your shell scripts to centrally report a
certain process/script status, share a piece of data, etc. Read on!

Every sysadmin is a universe.  The same way there are awesome tools for
pentesters such as faraday, which allows each pentester to work using the
tools he likes, and still be able to structurally share and enhance the data
that comes out of them in a central location, hume provides the same for
sysadmins, devops teams, etc.

# THIS IS A WORK IN PROGRESS

Hume will include a dashboard for queries. If someone writes it. I suck at
html5/js. But the command line tool will allow you to get status on a
process, grouped by hostname. Search by tags, etc.

## What will hume be?

You have some cron scripts. They need to run correctly, and you need to know
if anything happens. You usually check logs via email or some cool
dashboard. Yeah. But it gets boring, repetitive and you end up not checking
the logs. It is a well known sysadmin syndrome.

So, write those scripts adding hume commands. Something like this:
```
#!/bin/bash
# This script updates wordpress core, plugins and themes
# on a number of different directories.
wpcli='/usr/local/bin/wp'

# Maybe hume could read this variables
HTASK="WORDPRESS_UPDATE"
HTAGS="wordpress,cron,updates"

hume -c counter-start -t "$HUME_TASK" -T wordpress,cron,updates -m "Starting wordpress update cron process"
for dir in "/var/www/site1/htdocs" "/var/www/site2/htdocs" "/var/www/site3/htdocs"
do
	hume -t "$HTASK" -T "$HTAGS" -L info -m "Updating in $dir"
        cd $dir || hume -t "$HTASK" -T "$HTAGS" -L error -m "$dir does not exist"
        $wpcli core update || hume -t "$HTASK" -T "$HTAGS" -L error -m "Core update error in $dir"
        $wpcli plugin update --all || hume -t "$HTASK" -T "$HTAGS" -L error -m "Plugins update error in $dir"
        $wpcli theme update --all || hume -t "$HTASK" -T "$HTAGS" -L error -m "Themes update error in $dir"
	hume -t "$HTASK" -T "$HTAGS" -L info -m "Update process for $dir finished"
done
hume -c counter-stop -t "$HTASK" -T "$HTAGS" -m "Finished wordpress update cron task"
```

Then you could check the status of the latest run of the task:
humequery -t WORDPRESS_UPDATE --latest --hostname="webserver"

And you would get the list of every hume event, plus a summary, including 
## Ideas for implementation

hume uses zeromq over loopback to connecto to a hume daemon on the same
server. It must reduce its footprint as much as possible, we dont want hume
to slow down other processes. I am testing zmq-ipc and zmq-tcp, as well as
PUSH/PULL and REQ/REP scenarios. A systemd unit file is included for the hume
daemon (humed). A pid file will be read and checked by the hume client to
know beforehand if the hume daemon is working. A watchdog script that
triggers a special hume mode should exist to report working status. A lack
of watchdog-pinging will help the devops team detect the situation quickly.

The hume daemon will filter/store the hume packets, and send them to the
main repository which will be a relational database coupled with redis for
current-data storage and querying.

## Weird future concepts

* Signed script verification (imagine, the script is created, signed, hume
  checks the signature and executes/notifies if signature does not verify)

* Of course, authentication and authorization will need to be implemented.

But first lets make a useful, working prototype.

## DEVELOPMENT NOTES

# Define Basic CLI usage
* register and start process execution
* add events to process --warn, --info, --error, --debug
* event flagging for process timeline (instead of starting/stopping the watch, as every event including start/stop of process will include timestamp, we use event flagging to indicate events in the process timeline)
* stop process / deregister
* Define required tables and implement sqlalchemy
using ORM https://docs.sqlalchemy.org/en/13/orm/tutorial.html
using core https://docs.sqlalchemy.org/en/13/core/tutorial.html

Alternative: Use pypika to build sql queries using high level
abstractions - https://github.com/kayak/pypika and
pydal to run the queries thru an engine agnostic layer https://github.com/web2py/pydal

Components:

hume - called from within scripts. writes to a local publish queue. maybe a fallback to sqlite that humed can read on startup?
humed - its the local publish queue. consumes messages, applies filtering/preprocessing, security and pushes to CENTRAL COMMAND
central command - probably a redis instance that receives messages from humed. workers take the different messages and do
stuff

# TODO: mention export LINENO before calling hume
