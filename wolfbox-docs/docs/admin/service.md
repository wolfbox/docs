Service Management
==================

Introduction
------------

The Wolf approach to managing services, as with other design decisions, is simple,
robust, and flexible. While straightforward Unix-style shell scripts drive the
boot process, Wolf incorporates modern ideas to streamline the system.

In particular, Wolf uses an [Upstart](http://upstart.ubuntu.com)-inspired
event-triggering system where services start upon system events and other services
starting.

For instance, if a service requires network access, it makes sense to start it
when the boot scripts signal the `networking` event; or if your web application
requires [MongoDB](http://mongodb.org), then you might start it on the
`start_mongodb` event.

initstatus
----------

`initstatus` is a helper script for both showing the state of services, and
enabling/disabling service event handlers.

### Listing Services

    $ sudo initstatus

The output of `initstatus` is only sure to be correct when run as root. For
security reasons, non-administrators are not permitted to view the processes of
other users. This will prohibit some service files from knowing their state unless
run as root.

### Enabling a Service

    $ sudo initstatus --enable <service>[.<event>]

If you fail to specify an event, then all events for the given service will
be enabled.

### Disabling a Service

    $ sudo initstatus --disable <service>[.<event>]

If you fail to specify an event, then all events for the given service will
be disabled.

Service Reference
-----------------

### Service File

Service files live in `/etc/rc.d`, and are executable scripts taking one command
argument, typically accepting at least `start`, `stop`, `status`, and `help`.
Each possible command calls a function in the service file.

Service names must contain only letters, numbers, and dashes.

The following skeleton is common:

    #!/usr/bin/sh
    . /etc/rc.d/rc.functions

    service_start() {
        # ...
    }

    service_stop() {
        # ...
    }

    service_status() {
        # Must return 0 if the service is running, 2 if it is not, and 1 on error
        # ...
    }

    service_help() {
        # ...
    }

    command_dispatch "${1}"

All service files will generally source the helper library `/etc/rc.d/rc.functions`.
It provides the following functions:

* `msg [text]`: Display some informational text.
* `msg_error [text]`: Display an error message.
* `emit_event [name]`: Emit an event.
* `command_dispatch [command]`: Map the given command name to a function, and
  emits a signal of the form `command_name` where `name` is the filename of the
  service file. For example, starting the service `udev` will emit `start_udev`.
* `kill_pidfile [pidfile] [signal=TERM]`: Send a signal to the given `pidfile`.
  `signal` defaults to `TERM` to ask that the service shut down, but can be any
  signal name.
* `status_pidfile [pidfile]`: Display whether or not the service is running, based
  on the existence and contents of `pidfile`.

### daemon

The `daemon` command is a helper to run daemons and manage PID-files, and that can
respawn a crashed service.

### PID File Helper

Because most services share many common features, a higher-level helper is provided:

    #!/usr/bin/sh
    . /etc/rc.d/rc.pidfile

    run <command>

    command_dispatch "${1}"

This will automatically s

The features may be used, but only `run` is required:

* `run <command>`
* `respawn`
* `user <username>`
* `pre_start()`
* `post_start()`
* `pre_stop()`
* `post_stop()`

### Events

The Wolf init system provides a simple means of dispatching events to automatically
start services and run tasks. This works by placing executable scripts in
`/etc/rc.d/events/` of the following form:

    [name].[event_[target]]

All events are added to the system log for inspection.

Common events are:

* `filesystem`: All filesystems listed in `/etc/fstab` are available.
* `networking`: The network is ready.
* `system_ready`: All system initialization is complete.
* `start_[service]`: `service` has started.
* `stop_[service]`: `service` has stopped.

### Disabling Services and Event Handlers

To prevent events from triggering a handler, create a file named like the event
handler, but with the suffix `.disable`.

For example, to prevent the system message bus from starting automatically,
you can run the following command:

    $ sudo touch /etc/rc.d/events/dbus.filesystem.disable

This is equivalent to:

    $ sudo initstatus --disable dbus.filesystem

### Local Initialization Script

The `system_ready` event executes the file `/etc/rc.d/rc.local` once all other
services are ready.  Miscellaneous tasks may happen in this script, and you may
manually start services here.

Example
-------

Suppose you have a daemon called `myserviced`. Create the following file in
`/etc/rc.d/myserviced`:

    #!/usr/bin/sh
    . /etc/rc.d/rc.functions

    service_start() {
        # Create a PID file, and restart myserviced if it crashes.
        daemon -p /var/run/myserviced.pid \
               -r \
               /usr/bin/myserviced
    }

    service_stop() {
        kill_pidfile /var/run/myserviced.pid
    }

    service_status() {
        status_pidfile /var/run/myserviced.pid
    }

    service_help() {
        echo "myserviced [start | stop | status]"
    }

    command_dispatch "${1}"

If `myserviced` does not require any particular services, then you can start it
when Wolf has initialized itself by creating the following file in
`/etc/rc.d/events/myserviced.system_ready`:

    #!/usr/bin/sh
    /etc/rc.d/myserviced start

Ensure that both files are executable by running the following command:

    $ sudo chmod 755 /etc/rc.d/myserviced /etc/rc.d/myserviced.system_ready
