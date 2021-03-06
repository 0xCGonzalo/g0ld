# Testing for SSL and TLS Vulnerablities

## Summary
- [Tools](#tools)
- [Protocol SSLv3](#protocol-sslv3)
- [POODLE](#poodle)
- [BEAST & CRIME](#beast-&-crime)
- [LUCKY13](#lucky13)


## Tools

You can use [testssl](https://github.com/drwetter/testssl.sh) for this purpose.

## Protocol SSLv3

This protocol is obsolete. There are public vulnerabilities that allow an attacker to decrypt the communications between the client and the server.

You can use the following command for a PoC in your report:

```
OpenSSL> s_client -ssl3 -proxy 127.0.0.1:8080 -connect <server_victim>
```

## POODLE

(CVE-2014-3566) Using SSLv3 with CBC encryption makes the server vulnerable to Man in The Middle attacks, being able to decrypt message bytes with only 256 attempts per byte (assuming the application can be forced to send the same bytes multiple times).

You can use the following command for a PoC in your report:
```
OpenSSL> s_client -ssl3 -proxy 127.0.0.1:8080 -connect <server_victim>
```

## BEAST & CRIME

(CVE-2011-3389 / CVE-2012-4929): The use of TLSv1.0 with CBC encryption allows an attacker with the ability to force traffic, decrypt communications by sending messages of variable length and observing the size differences of the encrypted message.

You can use the following command for a PoC in your report:
```
OpenSSL> s_client -tls1 -cipher AES128-SHA -proxy 127.0.0.1:8080 --connect <server_victim>
```

## LUCKY13

(CVE-2013-0169): The server supports CBC ciphers, which are vulnerable to padding oracle attacks based on the processing time of the message authentication algorithm (MAC) used.

You can use the following command for a PoC in your report:
```
OpenSSL> s_client -cipher AES128-SHA -proxy 127.0.0.1:8080 --connect <server_victim>
```


