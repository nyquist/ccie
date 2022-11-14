# Controlling CLI Access

## CLI Modes

*   User EXEC Mode

    ```
    R>
    ! To enter the Privileged Exec Mode
    R> enable
    ```
*   Privileged EXEC Mode

    ```
    R#
    ! To go back to the User Exec Mode
    R# disable
    ```
*   Configuration Mode

    ```
    ! To enter config mode
    R# config terminal
    ! To exit config mode:
    R(config)# end
    ```

To protect access to the Privileged EXEC mode, use:

```
R(config)# enable {password PASS | secret SECRET}
! secret uses a better encryption algorithm than password encryption
```

### Custom Privilege Levels

These commands are not compatible with [AAA](aaa-101.md) mode of operation.\
By default, there are only 3 privilege levels:

* Level 0 = no rights
* Level 1 = User EXEC Mode
* Level 15 = Privileged EXEC Mode

You can define new privilege levels and assign commands to them, so they can be ran by lower level users:

```
R(config)# enable level LEVEL {password PASS | secret SECRET}
! set a password for the specific level
R(config)# privilege COMMAND [all] level LEVEL STRING
! All commands starting with STRING will be allowed at the specified level
! all - all suboptions will be allowed at the specified level
! COMMAND - is the parent command of the STRING command.
!        You shoud start building a tree from exec
```

### Role Based CLI Access

This feature is similar to the Custom Privilege levels, but it requires [aaa new-model](aaa-101.md)\
With this feature you can create custome views with different access to the CLI commands.\
First you need to be in the root view in order to create other views. To move to the root view, use:

```
R# enable view
```

To access the view, you can use the enable secret or password configured on the router. Then, go into the configuration mode and create a view, and specify a secret for it:

```
R(config)# parser view VIEW-NAME
R(config-view)# secret STRING
```

Then, start adding commands to the view, using a similar approach as with Custom Privilege Levels:

```
R(config-view)# commands COMMAND {include|exclude|include-exclusive} [all] STRING
! All commands starting with STRING will be allowed at the specified level
! all - all suboptions will be allowed at the specified level
! include - adds the command to this view
! exclude - removes the command from this view
! include-exclusive - adds the command to this view and removes it from other views
! COMMAND - is the parent command of the STRING command.
!        You shoud start building a tree from exec
```

There’s also the option of configuring a super view. This view can only be a collection of other views:

```
R(config)# parser view SUPER-VIEW superview
R(config-view)# secret SECRET
R(config-view)# view VIEW-NAME
```

You can move from one view to another using the command:

```
R# enable view [VIEW-NAME]
```

an AAA attribute (cli-view-name) can be passed from an AAA server to enable users automatic access to a view.

## CLI Sessions

### Local or Remote CLI

For Local access to the device you must use the Console or AUX port. To configure Local CLI access, use:

```
! Console:
R(config)# line console 0
! AUX:
R(config)# line aux 0
```

For remote CLI acces you must use Telnet or SSH to connect to the device. To configure remote CLI sessions, configure the Virtual Terminal Interfaces using:

```
R(config)# line vty LINE-START [LINE-END]
! Usually, LINE-START=0, LINE-END=4
```

### Protecting line access

By default, the console and AUX ports allow access without asking for credentials. To enable the router to ask for credentials, use the command:

```
R(config-line)# login [local|tacacs]
! no params: use line password (default on vty)
! local: use locally defined users
! tacacs: use tacacs authentication
```

If line password is used but not defined, login will fail. To set the line password, use:

```
R(config-line)# password PASS
```

When using line password, you can also set the default privilege level of authenticated users:

```
R(config-line)# privilege level [0-15]
```

When using locally defined users, you must first set a username:

```
R(config)# username USER [privilege LEVEL] {password PASS | secret SECRET}
! default LEVEL: 1
```

The privilege level of the USER will be used when connecting on the line.

A VTY line can be used for both incoming and outgoing connections. You can define the protocols allowed on each line, using:

```
R(config-line)# transport {input|output} PROTOCOL
! PROTOCL = usually telnet or ssh
! input = incoming connections
! output = outgoing connections
```

Protocols defined with the input keyword are allowed for connections to the terminal line, while protocols defined with the output keyword are protocols that can be used to connect from that line to another host.

## Password Encryption

By default, when a configuration file is saved, passwords are saved in clear text. You can enable automatic encryption of passwords in the config files, using:

```
R(config)# service password-encryption
```

When the configuration files are saved remotely, the passwords are still sent in clear text.\
For better encryption, use secret instead of password when available

## SSH

Before using SSH, a key must be generated, but to generate a key, you need a hostname and domain name.

```
R(config)# hostname HOST
HOST(config)# ip domain-name DOMAIN
HOST(config)# crypto key generate rsa
! You will be asked for the key size.
```

### SSH Server

```
R(config)# ip ssh {timeout SEC|authentication-retries RETRIES}
R(config)# ip ssh version {1|2}
! By default both v1 and v2 users are allowed
! You can chose only one version by selecting it in this command
```

### SSH Client

```
R# ssh -l USER SERVER-IP
```

### SCP Server

[AAA authentication](aaa-101.md#authentication) must be enabled for SCP. To enable the SCP server, use:

```
R(config)# ip scp server enable
```

The users must pass an aaa login authentication and an aaa exec authorization to use scp.
