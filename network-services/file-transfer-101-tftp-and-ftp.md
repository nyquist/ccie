# File Transfer 101 – TFTP & FTP

## TFTP

### TFTP Server

You can use a Cisco Router as a TFTP Server, usually used to serve IOS images to other routers.

```
R(config)# tftp-server FILE-URL [alias ALIAS] [ACL]
! ALIAS - the server will respond to requests for the ALIAS name with the FILE-URL file
! ACL - used to limit TFTP clients
```

You can also configure the TFTP client using:

```
R(config)# ip tftp source-interface INTERFACE
```

### TFTP Client

You can use the router as a TFTP Client using a command like:

```
R# copy tftp://FILE-SRC DESTINATION-URI
R# more tftp://FILE-SRC
```

## FTP

### FTP Client

You can use the router as a FTP Client using a command like:

```
R# copy ftp://FILE-SRC DESTINATION-URI
R# more ftp://FILE-SRC
```

In contrast with TFTP, FTP offers more advanced features. This is why we can configure the FTP client even further:

```
! Specify USER and PASS:
R(config)# ip ftp username USER
R(config)# ip ftp password PASS
! Specify the source interface
R(config)# ip ftp source-interface INTERFACE
! Connect using Passive FTP
R(config)# ip ftp passive
```
