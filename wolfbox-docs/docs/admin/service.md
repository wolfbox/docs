Service Management
==================

Introduction
------------

The Wolf approach to managing services, as with other design decisions, is simple,
robust, and flexible. While straightforward Unix-style shell scripts drive the
boot process, Wolf incorporates modern ideas to streamline the system.

In particular, Wolf uses an [Upstart](http://upstart.ubuntu.com/)-inspired
event-triggering system where services start upon system events, or other services
starting.

For instance, if a service requires network access, it makes sense to start it
when the boot scripts signal the `networking` event.

Service Reference
-----------------

### Service File

Service files live in `/etc/rc.d`, and are executable scripts taking one command
argument, typically accepting at least `start`, `stop`, `status`, and `help`.
Each possible command calls a function in the service file.

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

### Events

The Wolf init system provides a simple means of dispatching events to automatically
start services and run tasks. This works by placing executable scripts in
`/etc/rc.d/events/` of the following form:

    [name].[event_[target]]

Common events are:

* `filesystem`: The root filesystem is available.
* `networking`: The network is ready.
* `system_ready`: All system initialization is complete.
* `start_[service]`: `service` has started.
* `stop_[service]`: `service` has stopped.
* `ctrl-alt-del`: The user has pressed the Ctrl-Alt-Del key sequence, and the system
  will be rebooting.

### Disabling Services and Event Handlers

To prevent events from triggering a handler, create a file named like the event
handler, but with the suffix `.disable`.

For example, to prevent the system message bus from starting automatically,
you can run the following command:

    $ sudo touch /etc/rc.d/events/dbus.filesystem.disable

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
