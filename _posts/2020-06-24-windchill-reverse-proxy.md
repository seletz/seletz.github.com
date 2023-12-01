---
title: "Setting up a reverse proxy for Windchill"
date: 2020-06-24
tags: nexiles windchill security keycloak
---

## Abstract

The "Windchill Advanced Deployment Guide" describes how to set up a reverse
proxy for Windchill.  Why would that be useful?

- **Security:** Using a reverse proxy setup, clients won't directly send
  requests to the Windchill system.  This allows for tighter firewall settings.

- **Virtual Machine Setup w/o rehosting:** When using VMs to manage multiple
  Windchill environments, e.g. development, testing and production, a reverse
  proxy allows for simple cloning w/o rehost work, as the proxy may have a
  completely different FQDN as the Windchill System

![](/assets/images/windchill-reverse-proxy.png)

## Example Use Case

For my local development machine I use for the a customer Project, I wanted to try
to get OIDC authentication against Keycloak going.  Unfortunately, the dev
system runs Windows, for which the relevant Apache module is not available and
the source code seems to be UNIX only.  Additionally, PTC uses a custom built
Apache 2.2 which is a bit ancient.

## Example Setup

Task: set up a reverse proxy on my Mac which proxies requests to the Windows VM.
The proxy FQDN I'll use is `proxy.ventum-dev.nexiles.cloud` while the original FQDN
of the WT VM is `ventum-dev.nexiles.cloud`.

Steps to do:

- set two properties in Windchill to have Windchill render URLs for the proxy FQDN
- set up a Apache reverse proxy on my mac

### Windchill Configuration

I created a `nexiles-reverse-proxy.xconf` and hooked it into the `nexiles.xconf` as usual:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Configuration SYSTEM "xconf.dtd">
<Configuration xmlns:xlink="<http://www.w3.org/1999/xlink">>
  <!-- nexiles-reverse-proxy.xconf -->
  <Property
      name="wt.server.codebase"
      value="<http://proxy.ventum-dev.nexiles.cloud/Windchill">
      targetFile="codebase/wt.properties"
      overridable="true" />
      

  <Property
      name="wt.httpgw.mapCodebase"
      value="<http://ventum-dev.nexiles.cloud/Windchill">
      targetFile="codebase/wt.properties"
      overridable="true" />

</Configuration>
```

Hook it up:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Configuration SYSTEM "xconf.dtd">
<Configuration xmlns:xlink="<http://www.w3.org/1999/xlink">>
  <!-- nexiles.xconf -->

    ... many lines omitted ...

	<!-- Reverse Proxy -->
    <ConfigurationRef xlink:href="nexiles-reverse-proxy.xconf"/>

</Configuration>
```

I did the usual `xconfmanager -p` dance and restarted Windchill.

That's it for Windchill!

### Apache 2.4 Setup

I installed Apache 2.4 using homebrew:

```shell
$ brew install apache2
```

Configuration files are in `/usr/local/etc/httpd`.  I added the following changes:


```diff
--- httpd.conf.orig	2020-06-23 15:33:35.000000000 +0200
+++ httpd.conf	2020-06-23 17:38:05.000000000 +0200
@@ -49,7 +49,7 @@
 # prevent Apache from glomming onto all bound IP addresses.
 #
 #Listen 12.34.56.78:80
-Listen 8080
+Listen 127.0.0.1:80
 
 #
 # Dynamic Shared Object (DSO) Support
@@ -128,10 +128,10 @@
 LoadModule setenvif_module lib/httpd/modules/mod_setenvif.so
 LoadModule version_module lib/httpd/modules/mod_version.so
 #LoadModule remoteip_module lib/httpd/modules/mod_remoteip.so
-#LoadModule proxy_module lib/httpd/modules/mod_proxy.so
-#LoadModule proxy_connect_module lib/httpd/modules/mod_proxy_connect.so
+LoadModule proxy_module lib/httpd/modules/mod_proxy.so
+LoadModule proxy_connect_module lib/httpd/modules/mod_proxy_connect.so
 #LoadModule proxy_ftp_module lib/httpd/modules/mod_proxy_ftp.so
-#LoadModule proxy_http_module lib/httpd/modules/mod_proxy_http.so
+LoadModule proxy_http_module lib/httpd/modules/mod_proxy_http.so
 #LoadModule proxy_fcgi_module lib/httpd/modules/mod_proxy_fcgi.so
 #LoadModule proxy_scgi_module lib/httpd/modules/mod_proxy_scgi.so
 #LoadModule proxy_uwsgi_module lib/httpd/modules/mod_proxy_uwsgi.so
@@ -532,3 +532,30 @@
 SSLRandomSeed connect builtin
 </IfModule>
 
+
+# WT Proxy
+<VirtualHost *:80>
+    # ProxyPreserveHost On
+    Header set Access-Control-Allow-Origin "*"
+
+    ProxyPass         / <http://ventum-dev.nexiles.cloud/>
+    ProxyPassReverse  / <http://ventum-dev.nexiles.cloud/>
+
+    ErrorLog "/usr/local/var/log/httpd/proxy_error_log"
+    CustomLog "/usr/local/var/log/httpd/proxy_access_log" common
+
+    ServerName proxy.ventum-dev.nexiles.cloud
+</VirtualHost>
```

## Results

We have "rehosted" Windchill without actually doing so:

![](/assets/images/wt-reverse-proxy-screenshot.png)
