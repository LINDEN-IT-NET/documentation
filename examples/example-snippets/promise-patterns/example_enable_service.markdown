---
layout: default
title: Ensure a service is enabled and running
published: true
tags: [Examples, Policy, ntp, file editing]
reviewed: 2013-06-08
reviewed-by: atsaloli
---

Bring up time service using the CFEngine services abstraction of
processes and commands.

The `non_standard_services` bundle below is based on `standard_services`
bundle in the CFEngine Standard Library.  The Standard Library does
not include ntp today, so we have to supply our own code for it.


```cf3
body common control
{
bundlesequence => { "enable_ntp_service" };
inputs => { "libraries/cfengine_stdlib.cf" };

}

bundle agent enable_ntp_service
{

services:

  "ntp"

    service_policy => "start",
    service_method => service_ntp;
}

body service_method service_ntp
{
service_bundle => non_standard_services("$(this.promiser)","$(this.service_policy)");
}

bundle agent non_standard_services(service,state)
{
reports:

  !done::

    "Test service promise for \"$(service)\" -> $(state)";

vars:

  ubuntu::

    "startcommand[ntp]"   string => "/etc/init.d/ntp start";
    "restartcommand[ntp]" string => "/etc/init.d/ntp restart";
    "reloadcommand[ntp]"  string => "/etc/init.d/ntp reload";
    "stopcommand[ntp]"    string => "/etc/init.d/ntp stop";
    "pattern[ntp]"        string => ".*ntpd.*";

  redhat::
    "startcommand[ntp]"   string => "/etc/init.d/ntpd start";
    "restartcommand[ntp]" string => "/etc/init.d/ntpd restart";
    "reloadcommand[ntp]"  string => "/etc/init.d/ntpd reload";
    "stopcommand[ntp]"    string => "/etc/init.d/ntpd stop";
    "pattern[ntp]"        string => ".*ntpd.*";

 # METHODS that implement these ............................................

classes:

  "start" expression => strcmp("start","$(state)"),
             comment => "Check if to start a service";
  "restart" expression => strcmp("restart","$(state)"),
             comment => "Check if to restart a service";
  "reload" expression => strcmp("reload","$(state)"),
             comment => "Check if to reload a service";
  "stop"  expression => strcmp("stop","$(state)"),
             comment => "Check if to stop a service";

# Do we want to include the packages here too?

processes:

  start::

    "$(pattern[$(service)])" ->  { "@(stakeholders[$(service)])" }

             comment => "Verify that the service appears in the process table",
       restart_class => "start_$(service)";

  stop::

    "$(pattern[$(service)])" -> { "@(stakeholders[$(service)])" }

            comment => "Verify that the service does not appear in the process",
       process_stop => "$(stopcommand[$(service)])",
            signals => { "term", "kill"};

commands:

  "$(startcommand[$(service)])" -> { "@(stakeholders[$(service)])" }

            comment => "Execute command to start the $(service) service",
         ifvarclass => canonify("start_$(service)");

  restart::
    "$(restartcommand[$(service)])" -> { "@(stakeholders[$(service)])" }

            comment => "Execute command to restart the $(service) service";

  reload::
    "$(reloadcommand[$(service)])" -> { "@(stakeholders[$(service)])" }

            comment => "Execute command to reload the $(service) service";
}

```

Example run:

```
# /etc/init.d/ntp stop
 * Stopping NTP server ntpd                                                                                                                     [ OK ]
# cf-agent -f enable_service.cf -K
2013-06-08T20:11:55-0700   notice: Q: "...init.d/ntp star":  * Starting NTP server ntpd
Q: "...init.d/ntp star":    ...done.

2013-06-08T20:11:55-0700   notice: R: Test service promise for "ntp" -> start
#
```

And again, with Inform:

```
# /etc/init.d/ntp stop
 * Stopping NTP server ntpd                                                                                                                     [ OK ]
# cf-agent -KIf enable_service.cf
2013-06-08T20:11:32-0700     info: This agent is bootstrapped to '192.168.183.208'
2013-06-08T20:11:33-0700     info: Running full policy integrity checks
2013-06-08T20:11:33-0700     info: /enable_ntp_service/services/'ntp': Service 'ntp' could not be invoked successfully
2013-06-08T20:11:33-0700     info: /enable_ntp_service/services/'ntp'/non_standard_services/processes/'$(pattern[$(service)])': Making a one-time restart promise for '/usr/sbin/ntpd.*'
2013-06-08T20:11:33-0700     info: Executing 'no timeout' ... '/etc/init.d/ntp start'
2013-06-08T20:11:33-0700   notice: Q: "...init.d/ntp star":  * Starting NTP server ntpd
Q: "...init.d/ntp star":    ...done.

2013-06-08T20:11:33-0700     info: Last 2 quoted lines were generated by promiser '/etc/init.d/ntp start'
2013-06-08T20:11:33-0700     info: Completed execution of '/etc/init.d/ntp start'
2013-06-08T20:11:33-0700   notice: R: Test service promise for "ntp" -> start
```
