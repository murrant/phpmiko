PhpMiko
=======

<3 NetMiko but I'm a php developer. So I've decided to build a php alternative

Mad respect to [Kirk Byers](https://github.com/ktbyers/netmiko) for creating netmiko and being a huge inspiration to this project.

## Requires:

- Php >= 7.1

## Installation:

```bash
composer require epiecs/phpmiko
```

## Supported devices

### Implemented

- Aruba
- Cisco ios
- Cisco nx-os
- HPE Comware
- Juniper Junos

### Planned

- Avaya
- Linux
- Vyos
- Checkpoint
- ...

## Supported protocols

- ssh
- telnet

## Contents

- [PhpMiko](#phpmiko)
	- [Requires:](#requires)
	- [Installation:](#installation)
	- [Supported devices](#supported-devices)
		- [Implemented](#implemented)
		- [Planned](#planned)
	- [Supported protocols](#supported-protocols)
	- [Contents](#contents)
	- [Examples:](#examples)
		- [Connecting to a device](#connecting-to-a-device)
		- [Sending commands](#sending-commands)
			- [Sending one command as string](#sending-one-command-as-string)
			- [Sending one command as an array](#sending-one-command-as-an-array)
			- [Sending multiple commands](#sending-multiple-commands)
		- [Output](#output)
	- [Command types](#command-types)
		- [cli](#cli)
		- [operation](#operation)
		- [configure](#configure)
	- [Device types and command mapping](#device-types-and-command-mapping)
	- [Cleaning up and debugging](#cleaning-up-and-debugging)
		- [Setting raw mode](#setting-raw-mode)
		- [Closing the connection](#closing-the-connection)
	- [Contributions](#contributions)

## Examples:

### Connecting to a device

```php

require_once __DIR__ . '/vendor/autoload.php';

$device = new \Epiecs\PhpMiko\ConnectionHandler([
	'device_type' => 'junos',
	'hostname'    => '192.168.0.1',
	'username'    => 'username',
	'password'    => 'password',
	'protocol'    => 'ssh',          //default is ssh
	'port'        => 22,             //defaults to the protocol default if not set
	'secret'      => 'secret',       //default is ''
	'verbose'     => false,          //default is false
	'raw'         => false           //default is false
]);
```

When the __raw__ flag is set to true PhpMiko will not clean up output and return it as is (minus a few hidden characters so you at least get all textual output).

__secret__ is used when a runlevel requires a different password. Like for example enable mode (privileged exec) in Cisco ios. You would put the enable password in the secret field.

__verbose__ when set to true all sent and received raw packets will be ouput for debugging purposes.

```plaintext
--- output cut short for brevity, notice the arrows

-> NET_SSH2_MSG_CHANNEL_DATA (since last: 0.0013, network: 0.0002s)
00000000  00:00:00:00:00:00:00:05:64:61:74:65:0a           ........date.

<- NET_SSH2_MSG_CHANNEL_DATA (since last: 0.1556, network: 0.0011s)
00000000  00:00:00:02:00:00:00:07:64:61:74:65:0d:0d:0a     ........date...

<- NET_SSH2_MSG_CHANNEL_DATA (since last: 0.1458, network: 0.0001s)
00000000  00:00:00:02:00:00:00:1f:46:72:69:20:4a:75:6e:20  ........Fri Jun
00000010  32:31:20:31:31:3a:31:36:3a:31:32:20:43:45:53:54  21 11:16:12 CEST
00000020  20:32:30:31:39:0d:0a                              2019..
```

For a list of all __device_types__ refer to [Device types and command mapping](#device-types-and-command-mapping).

### Sending commands

When sending commands you can either provide a string or an array. Either way is fine. When providing an array the commands are run in order.

#### Sending one command as string

```php
print_r($device->operation('show interfaces ge-0/0/0'));
```

#### Sending one command as an array

```php
print_r($device->operation([
	'show interfaces ge-0/0/0',
]));
```

#### Sending multiple commands

```php
print_r($device->operation([
	'show interfaces ge-0/0/0',
	'show interfaces ge-0/0/1',
]));
```

### Output

All output will be returned as an array where the key is the command that was run

```plaintext
Array
(
    [show version] => fpc0:
--------------------------------------------------------------------------
Hostname: SW-Junos
Model: ex3300-48p
Junos: 15.1R7.9
JUNOS EX  Software Suite [15.1R7.9]
JUNOS FIPS mode utilities [15.1R7.9]
JUNOS Online Documentation [15.1R7.9]
JUNOS EX 3300 Software Suite [15.1R7.9]
JUNOS Web Management Platform Package [15.1R7.9]

)
```

## Command types

PhpMiko has 3 distinct mechanisms to send commands:

- cli
- operation
- configure

All commands are run sequentially and chained. However this is only per set of supplied commands.

After each block of commands the current mode (cli/operation/configure) of the device will be exited.

Eg. if you go into edit mode of an interface in junos or configure terminal in cisco ios and run another configure set of commands you wont start where you left of the previous time. After each run is complete there will be a clean exit.

Even though PhpMiko has 3 mechanisms not all are implemented on each device. Some devices only have 1 or 2 configuration tiers. For an overview please refer to [Device types and command mapping](#device-types-and-command-mapping).

### cli

Runs one or more cli commands on a device.
Eg. the standard linux cli in junos or user exec mode in cisco ios

```php
print_r($device->cli([
	'pwd',
	'cd /var/www ; pwd ; ls -l'
]));
```

### operation

Runs one or more operational/enable commands on a device.
Eg. cli (operational) mode in junos or privileged exec mode in cisco ios


```php
print_r($device->operation([
	'show interfaces terse',
	'show configuration interfaces ge-0/0/0'
]));
```

### configure

Sends one or more configuration commands to the device.
Eg. configuration mode in junos or global configuration (configure terminal) in cisco ios

```php
print_r($device->configure([
	'set interfaces ge-0/0/0 description "Test for documentation"',
	'edit interfaces ge-0/0/1',
	'set description "Sequential commands work"',
]));
```

## Device types and command mapping

| Vendor  	| Device      	| device_type 	| cli       	| operation        	| configure          	|
|---------	|-------------	|-------------	|-----------	|------------------	|--------------------	|
| Aruba   	| Aruba       	| aruba       	| user exec 	| privileged exec  	| configure terminal 	|
| Cisco   	| Cisco ios   	| cisco_ios   	| user exec 	| privileged exec  	| configure terminal 	|
|         	| Cisco nxos  	| cisco_nxos  	| user exec 	| user exec        	| configure terminal 	|
| Juniper 	| Junos       	| junos       	| linux cli 	| operational mode 	| configuration mode 	|
| HP      	| HPE Comware 	| comware     	| user exec 	| user exec        	| system-view        	|

## Cleaning up and debugging

### Setting raw mode

Defaults to true when calling the function.

```php
$device->raw();
$device->raw(false);
```

### Closing the connection

```php
$device->disconnect();
```

## Contributions

* HPE Comware support [murrant](https://github.com/murrant)
