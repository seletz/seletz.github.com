---
title: "Windchill Open ID Connect Login with Keycloak"
date: 2020-06-25
tags: nexiles windchill security keycloak
---

## Abstract

The goal of this article is to explain how to set Windchill up such that
Keycloak is used.  Basically we want SSO in Windchill using Keycloak.

## Prerequisites

In order to keep things short, I assume that:

- You have set up a reverse proxy in front of Windchill
- You have a running Keycloak instance
- You have set up a new "Windchill" Realm in Keycloak

## Approach

The solution has these parts:

- Set up a Keycloak Client for Windchill
- Set up OIDC authentication in the Apache proxy server
- Disabling authentication in Windchill
- Adding a servlet filter to fetch the OICD claim which contains the user name
  and return it as a remote user.
  
## Keycloak setup

### Setup a Client

In Keycloak, in your Windchill Realm, create a new client like so.  Note that
the client name – "windchill-proxy-neweden" in this example – is arbitrary.

Note also that the redirect URL must:

- Point to the FQDN of the Windchill Proxy
- Use a Path below the "secured" content, i.e. below "/Windchill"
- That path MUST NOT exist, i.e. no content may be served for that path.

![](/assets/images/wt-kc-sso/wt-sso-keycloak-kc-client-setup.png)

Copy the client secret from "credentials":

![](/assets/images/wt-kc-sso/wt-sso-keycloak-client-secret.png)

### Add *wtadmin* user

Create a administrator user fro Windchill in Keycloak.

> Use the user name for the windchill site admin chosen at Windchill installation time.
> In my case this was `wcadmin`.
{: .prompt-info }


![](/assets/images/wt-kc-sso/wt-sso-keycloak-wcadmin-user.png)

## Apache OICD Setup

For this we need `mod_auth_openidc`.  We need to compile this module from source.
The examples below work on my Mac, YMMV.  I assume that you have installed
Apache and it's build tools.  On a Mac, `brew install apache2` does the trick.

### Compile and install mod_auth_openidc

Clone and compile `cjose` package, which is a dependency for `mod_auth_openidc`:

```bash
$ git clone git@github.com:cisco/cjose.git
$ cd cjose
$ ./configure && make && make install
```

Compile and install `mod_auth_openidc`.  This will install the module
automatically to the correct location where it is expected:

```bash
$ git clone git@github.com:zmartzone/mod_auth_openidc.git
$ cd mod_auth_openidc
$ ./autogen.sh
$ ./configure && make && make install
```

### Apache Configuation

Change your Apache Proxy to match the following.  Replace the following values:

- OIDCProviderMetadataURL: Must point to the "well known" Open ID Connect
  configuration URL.  The example uses Windchill as Realm.
- OIDCClientID: The client id as defined in Keycloak
- OIDCClientSecret: The client secret from Keycloak

```text
# OIDC
LoadModule auth_openidc_module lib/httpd/modules/mod_auth_openidc.so

# WT Proxy
<VirtualHost *:80>
    # ProxyPreserveHost On
    Header set Access-Control-Allow-Origin "*"

    OIDCProviderMetadataURL <https://auth.keycloak.test/auth/realms/Windchill/.well-known/openid-configuration>
    OIDCRedirectURI <http://proxy.ventum-dev.nexiles.cloud/Windchill/oauth2callback>
    OIDCCryptoPassphrase 0123456789
    OIDCClientID <<CLIENT ID FROM KEYCLOAK HERE>>
    OIDCClientSecret <<CLIENT SECRET FROM KEYCLOAK HERE>>
    # See <https://github.com/Reposoft/openidc-keycloak-test/issues/7>
    # OIDCProviderTokenEndpointAuth client_secret_basic

    OIDCRemoteUserClaim username
    OIDCScope "openid email"

    <location /Windchill>
      AuthType openid-connect
      Require valid-user

      ProxyPass         <http://ventum-dev.nexiles.cloud/Windchill>
      ProxyPassReverse  <http://ventum-dev.nexiles.cloud/Windchill>
      SetEnv proxy-chain-auth On
    </Location>

    ErrorLog "/usr/local/var/log/httpd/proxy_error_log"
    CustomLog "/usr/local/var/log/httpd/proxy_access_log" common

    ServerName proxy.ventum-dev.nexiles.cloud
</VirtualHost>
```

Restart the server using apachectl:

```bash
$ sudo apachectl graceful
```

Now if you try to navigate to Windchill, you should be redirected to the Keycloak login page.

> If you log in using the Keycloak user you created just before, you will notice
> that login succeeds but Windchill triggers the usual
> authentication.  We'll remedy that in the next step.
{: .prompt-info }

![](/assets/images/wt-kc-sso/wt-sso-keycloak-login.png)

## Windchill Authentication Changes

As we have Keycloak for Authentication, we don't need the stock Windchill
authentication anymore.  We're going to disable it.  For this we use the PTC
supplied ant script.

> This will recreate `conf/extra/app-Windchill-Auth.conf`.  If you changed this
> configuration for some reason, be sure to make a backup and re-apply your changes
> afterwards.
{: .prompt-warning }

### Disable OOTB Windchill authentication

Open a Windchill shell, change to the HTTPServer directory and do:

```bash
$ ant -DprotocolAuthOnly=true -f webAppConfig.xml regenAllWebApps
```

Restart the web server afterwards.

> This apparently just removes the `<LocationMatch />` directive which protects `/Windchill` and `/Windchill-WHC`.
{: .prompt-info }

```diff
diff --git a/ptc/Windchill_10.2/HTTPServer/conf/extra/app-Windchill-Auth.conf b/ptc/Windchill_10.2/HTTPServer/conf/extra/app-Windchill-Auth.conf
index b3beecb..375d550 100644
--- a/ptc/Windchill_10.2/HTTPServer/conf/extra/app-Windchill-Auth.conf
+++ b/ptc/Windchill_10.2/HTTPServer/conf/extra/app-Windchill-Auth.conf
@@ -18,14 +18,6 @@
 
 # Authenticated resources
 
-<LocationMatch ^/+Windchill/+(;.*)?>
-  AuthzLDAPAuthoritative off
-  AuthName "Windchill"
-  AuthType Basic
-  AuthBasicProvider Windchill-AdministrativeLdap Windchill-EnterpriseLdap 
-  require valid-user
-</LocationMatch>
-
 <LocationMatch ^/+Windchill/+infoengine/+verifyCredentials.html(;.*)?>
   AuthzLDAPAuthoritative off
   AuthName "Windchill"
@@ -139,15 +131,6 @@
   Allow from all
 </LocationMatch>
 
-<LocationMatch ^/+Windchill/+servlet/+nexiles(;.*)?>
-  AuthzLDAPAuthoritative off
-  AuthName "Windchill"
-  AuthType Basic
-  AuthBasicProvider Windchill-AdministrativeLdap Windchill-EnterpriseLdap
-  require valid-user
-</LocationMatch>
-
-
 # SSL client certificate authenticated resources
 <IfDefine SSL>
 
diff --git a/ptc/Windchill_10.2/HTTPServer/conf/extra/app-Windchill-WHC-Auth.conf b/ptc/Windchill_10.2/HTTPServer/conf/extra/app-Windchill-WHC-Auth.conf
index fe08814..4a8d7b4 100644
--- a/ptc/Windchill_10.2/HTTPServer/conf/extra/app-Windchill-WHC-Auth.conf
+++ b/ptc/Windchill_10.2/HTTPServer/conf/extra/app-Windchill-WHC-Auth.conf
@@ -15,13 +15,3 @@
   AuthLDAPBindDN "cn=Manager"
   AuthLDAPBindPassword "****************"
 </AuthnProviderAlias>
-
-# Authenticated resources
-
-<LocationMatch ^/+Windchill-WHC/+(;.*)?>
-  AuthzLDAPAuthoritative off
-  AuthName "Windchill"
-  AuthType Basic
-  AuthBasicProvider Windchill-WHC-AdministrativeLdap Windchill-WHC-EnterpriseLdap 
-  require valid-user
-</LocationMatch>
```

If you try to login again, you'll be greeted with a nice traceback.  This is
because no-one is telling Windchill the user name it should use.
We'll fix that in the next step.

### Adding a Servlet Filter for OICD Users

What we need to do is to tell Windchill the remote user it should use.  The way
this works in Windchill is pretty easy – Windchill relies on the
`HttpServletRequest.getRemoteUser()` call to return a `java.security.Principal`.
That principal object is pretty simplistic – it's just a POJO which has a
`getName().

Our user name is passed on by the proxy as HTTP parameter
`OIDC_CLAIM_preferred_user` (there are many others).  This is fine, because the
proxy server – specifically the `mod_auth_openidc module` – strips all incoming
HTTP parameters starting with `OICD_` – so spoofing users by just passing a http
parameter won't work from the client.

What we need to do is to add a servlet filter which extracts the user name,
wraps the `HttpServletRequest`, overwrites the `getUserPrincipal()` and returns a
user there which matches our user name from our OIDC claim.  Sounds scary, but
is rather simple.

```java
package com.nexiles.example.wtoidc.oidc;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;

import java.security.Principal;

@Data
@Slf4j
@AllArgsConstructor
public class OIDCUser implements Principal {
    private String name;
}
```

```java
package com.nexiles.example.wtoidc.oidc;

import lombok.extern.slf4j.Slf4j;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import javax.servlet.http.HttpSession;
import java.security.Principal;
import java.util.HashMap;

@Slf4j
public class OIDCServletWrapper extends HttpServletRequestWrapper {
    private static final String OIDC_CLAIM_USERNAME = "OIDC_CLAIM_username";

    private static HashMap<String,String> userMapping;

    static {
        userMapping = new HashMap<>();
    }

    private OIDCUser user;

    private String getOIDCClaim(HttpServletRequest request) {
        String username = request.getHeader(OIDC_CLAIM_USERNAME);
        if (username == null) {
            log.warn("No OIDC claim found in request: {}", OIDC_CLAIM_USERNAME);
            return null;
        }

        log.trace("username={}", username);

        return username;
    }

    private OIDCUser getUserFromRequest(HttpServletRequest request) {
        String username = getOIDCClaim(request);
        if (username == null) {
            log.trace("OIDCServletWrapper.getUserFromRequest: no user.");
            return null;
        }

        OIDCUser theUser = new OIDCUser(username);
        log.trace("OIDCServletWrapper.getUserFromRequest: user={}", theUser);
        return theUser;
    }


    /**
     * Constructs a request object wrapping the given request.
     *
     * @param request
     * @throws IllegalArgumentException if the request is null
     */
    public OIDCServletWrapper(HttpServletRequest request) {
        super(request);

        HttpSession session = request.getSession();
        log.trace("Session ID: {}", session.getId());

        user = getUserFromRequest(request);
        if (user != null) {
            String claim = (String) session.getAttribute("OICD_CLAIM");
            if (claim == null) {
                session.setAttribute("OICD_CLAIM", user.getName());
            }

            log.trace("OICD_CLAIM: {}", claim);
        }
    }

    /**
     * The default behavior of this method is to return getRemoteUser()
     * on the wrapped request object.
     */
    @Override
    public String getRemoteUser() {
        String remote_user;
        if (user != null) {
            remote_user = user.getName();
        } else {
            remote_user = super.getRemoteUser();
        }

        log.trace("OIDCServletWrapper.getRemoteUser: {}", remote_user);
        return remote_user;
    }

    /**
     * The default behavior of this method is to return getUserPrincipal()
     * on the wrapped request object.
     */
    @Override
    public Principal getUserPrincipal() {
        Principal principal;
        if (user != null) {
            principal = user;
        } else {
            principal = super.getUserPrincipal();
        }
        log.trace("OIDCServletWrapper.getUserPrincipal: {}", principal);
        return principal;
    }
}
```

Activating this is also simple:

- Place the built jar in `$WT_HOME/codebase/WEB-INF/lib`
- Enable the servlet filter by adding to `$WT_HOME/codebase/WEB_INF/web.xml`:

```diff
diff --git a/ptc/Windchill_10.2/Windchill/codebase/WEB-INF/web.xml b/ptc/Windchill_10.2/Windchill/codebase/WEB-INF/web.xml
index 1f21629..6b2f087 100644
--- a/ptc/Windchill_10.2/Windchill/codebase/WEB-INF/web.xml
+++ b/ptc/Windchill_10.2/Windchill/codebase/WEB-INF/web.xml
@@ -53,6 +53,12 @@
     <param-name>resteasy.servlet.mapping.prefix</param-name>
     <param-value>/servlet/rest</param-value>
   </context-param>
+
+  <!-- OIDC Servlet Filter -->
+  <filter>
+      <filter-name>OIDCServletFilter</filter-name>
+      <filter-class>com.nexiles.example.wtoidc.oidc.OIDCServletFilter</filter-class>
+  </filter>
   
   <filter>
     <description>Filter to monitor servlet requests as a whole; should always be first in filter list</description>
@@ -167,6 +173,11 @@
     <filter-name>WsdlServletFilter</filter-name>
     <filter-class>com.ptc.jws.servlet.filter.WsdlServletFilter</filter-class>
   </filter>
+
+  <filter-mapping>
+      <filter-name>OIDCServletFilter</filter-name>
+      <url-pattern>/*</url-pattern>
+  </filter-mapping>
   
   <filter-mapping>
     <filter-name>ServletRequestMonitor</filter-name>
@@ -946,4 +957,4 @@
     <env-entry-value>webapp:/Windchill</env-entry-value>
   </env-entry>
   
-</web-app>
\ No newline at end of file
+</web-app>
```

That's it – logging in with wcadmin should work now.

![](/assets/images/wt-kc-sso/wt-sso-keycloak-login-successful.png)

## Disclaimer, FAQ

### Disclaimer

As with all things which change the default behaviour of  Windchill, please test
the changes **thoroughly**.

The changes mentioned here are for *example purposes* only.  If you need a real working
setup, use the the things in this post as *inspiration* and do your own
research, or reach out for help.

### FAQ

Q: I added a new user in Keycloak, but get a traceback when trying to log in.  What gives?

A: You need to add the user in Windchill as well.  Use the site admin user, log
in and create users as normal.  Keycloak does authentication  – Windchill still
does it's own authorization.  For this users need to exist in Windchill.  OOTB
this means they live in the Windchill DS LDAP server.

Q:  Does this mean that I don't need LDAP anymore?

A: No.  See the answer above.  authorization is done in Windchill against
Enterprise / Administrative LDAP.

Q: I have a user in Enterprise / Administrative LDAP.  Can't log in.  WTF?

A: You need to add each and every user which needs access to Windchill to Keycloak.

Q: CANIHAZ LDAP Keycloak sync PLS?

A: RTFM Keycloak LDAP, Federation, etc and write your own tutorial. KTHANKSBYE.
