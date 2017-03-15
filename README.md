# Nagios NRPE and Plugins

This guide is to show to compile Nagios NRPE agent and it's plugin on the server that does not have any Internet connectivity.

## Prerequisites
1. Download these files from the Internet and upload to `/tmp`
```
https://nagios-plugins.org/download/nagios-plugins-2.1.4.tar.gz
https://github.com/NagiosEnterprises/nrpe/archive/3.0.1.tar.gz
```

2. Make sure following packages are installed.

	-     NRPE `gcc glibc glibc-common openssl-devel perl wget`
	-     Nagios Plugin `gcc glibc glibc-common gd gd-devel make net-snmp m4 gettext automake autoconf wget gettext`

## Nagios NRPE installation
```
cd /tmp
tar -zxvf nrpe-3.0.1.tar.gz
cd nrpe-3.0.1
./configure --enable-command-args
make all
make install-groups-users
make install
make install-config
```

### Update Service File
The /etc/services file is used by applications to translate human readable service names into port numbers when connecting to a machine across a network.
```
echo >> /etc/services
echo '# Nagios services' >> /etc/services
echo 'nrpe    5666/tcp' >> /etc/services
```

### Install Service / Daemon
This installs the service or daemon files.
```
make install-init
chkconfig nrpe on
```

### Configure Firewall
Port 5666 is used by NRPE and needs to be opened on the local firewall.
```
iptables -I INPUT -p tcp --destination-port 5666 -j ACCEPT
service iptables save
```

### Update Configuration File
The file nrpe.cfg is where the following settings will be defined. It is located at `/usr/local/nagios/etc/nrpe.cfg`
```
allowed_hosts=127.0.0.1,[Nagios Server IP Address]
dont_blame_nrpe=1
```

### Start Service / Daemon
```service nrpe start```

### Verifying NRPE service listening
```
/usr/local/nagios/libexec/check_nrpe -H 127.0.0.1
NRPE v3.0
```

## Nagios Plugin installation
NRPE needs plugins to operate properly. The following steps will walk you through installing Nagios Plugins.

```
cd /tmp
tar xzvf nagios-plugins-2.1.4.tar.gz
cd nagios-plugins-2.1.4
./configure
make
make install
```

### Nagios Plugin Verification
```
/usr/local/nagios/libexec/check_nrpe -H 127.0.0.1 -c check_load
OK - load average: 0.48, 0.20, 0.10|load1=0.480;15.000;30.000;0; load5=0.200;10.000;25.000;0; load15=0.100;5.000;20.000;0;
```

### Troubleshooting

#### nrpe: unrecognized service

1. Create a init script that contain below codes.

```
#!/bin/bash
#
# chkconfig: 2345 20 80
# nrpe daemon 
# description: The NRPE daemon communicates with the nagios daemon \
#              transmitting vital system & hardware information about \
#              different services.
#
# Author: Nick Winn
# 
# $Revision: 1.2 $

# Source function library.
. /etc/init.d/functions

# Get config.
. /etc/sysconfig/network

# Check that networking is up.
[ "${NETWORKING}" = "no" ] && exit 0
[ -f /etc/nrpe.cfg ] || exit 0

NRPE="/usr/sbin/nrpe"
PIDFILE="/var/run/nrpe.pid"
CFG="/etc/nrpe.cfg"

RETVAL=0

start() {
    echo -n $"Starting NRPE: "
    "${NRPE}" -c $CFG -d > /dev/null
    RETVAL=$?
    echo "OK"
}

stop() {
    if [ -f "${PIDFILE}" ]; then
	echo -n $"Stopping NRPE: "
	killall ${NRPE} > /dev/null
	echo "OK"
    fi
}

restart() {
    stop
    start
}

# See how we were called.
case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  status)
        status NRPE
        RETVAL=$?
        ;;
  reload)
        restart
        ;;
  restart)
        restart
        ;;
  condrestart)
        if [ -f "${PIDFILE}" ]; then
            restart
        fi
        ;;
  *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart}"
        exit 1
        ;;
esac

exit $RETVAL

```

*Source https://support.nagios.com/kb/article.php?id=515*