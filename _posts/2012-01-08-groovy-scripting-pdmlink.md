---
layout: post
title: "groovy: scripting PDMlink"
date: 2012-01-08 20:33
comments: true
categories: groovy java
published: true
---

In this post I'll explore how `groovy` may be used to script `Windchill PDMlink`.

<!-- more -->

Setting up the classpath
------------------------

To have acces sto the `PDMlink` libraries, I'll add a locally mounted `codebase`
directory to the `$CLASSPATH`:

```bash
    $ echo $WT_HOME
    /Volumes/settr-vm-ptc/Windchill
    $ wtclasspath
    export CLASSPATH=:/Volumes/settr-vm-ptc/Windchill/codebas...
    $ $(wtclasspath)
```

The `wtclasspath`script looks like this:

```bash
#!/bin/bash

function setup_classpath()
{
	for jar in $(find $WT_HOME/codebase/WEB-INF -name "*.jar"); do
		cp=$cp:$jar
	done
}

setup_classpath

echo "export CLASSPATH=$cp:$WT_HOME/codebase"
```


Logging In
----------

To log in, we need to fetch the `RemoteMethodServer` and set username and password:

```groovy
import wt.method.RemoteMethodServer

ms = RemoteMethodServer.getDefault()
ms.setUserName("orgadmin")
ms.setPassword("orgadmin")
```

After this, we're able to use the **windchill API**.

Fetching PDMlink Business Objects by Reference
----------------------------------------------

Fetching a business object using it's *Object Reference String* translates
also quite well to groovy:

```groovy
import wt.method.RemoteMethodServer
import com.ptc.wvs.server.util.PublishUtils


def login(args) {
    ms = RemoteMethodServer.getDefault()
    ms.setUserName(args.user)
    ms.setPassword(args.password)
    
    return ms
}

def getObjectFromOR = { oid ->
    PublishUtils.getObjectFromRef(oid)
}

def getORFromObject = { obj ->
    PublishUtils.getRefFromObject(obj)    
}

// log in
login(user: "orgadmin", password: "orgadmin")

// fetch a object by using a OR
def oid = "VR:wt.epm.EPMDocument:8483330"
def doc = getObjectFromOR(oid)

// We're able to access the properties of the EPMDocument
println "${doc.displayIdentifier} [${doc.state} ${doc.type}] ${getORFromObject(doc)}"

// for the "inspact last" feature of groovyConsole 
return doc 
```

Running this script is easy:

```bash
    $ groovy -cp $CLASSPATH wt_docinfo.groovy
    k06__core-coil_cc0000000.asm, -.17 [RELEASED CAD-Dokument] OR:wt.epm.EPMDocument:112511174
```
    
But this is all quite basic, and not using the **groovy** language features very much at all.

Fetching all iterations of an object
------------------------------------

Now we're going to use the `VersionControlHelper` service to fetch all iterations of an
object and print some information.  Please note how **groovy** makes it easy to traverse
collections using `each` and a *closure*.  Also note the `import ... as ...`:

```groovy
import wt.method.RemoteMethodServer
import com.ptc.wvs.server.util.PublishUtils
import wt.vc.VersionControlHelper as VCH

def login(args) {
    ms = RemoteMethodServer.getDefault()
    ms.setUserName(args.user)
    ms.setPassword(args.password)
    
    return ms
}

def getObjectFromOR(oid) {
    PublishUtils.getObjectFromRef(oid)
}

def getORFromObject(obj) {
    PublishUtils.getRefFromObject(obj)    
}

def info(doc) {
    "${doc.displayIdentifier} [${doc.state} ${doc.type}] ${getORFromObject(doc)}"    
}

// log in
login(user: "orgadmin", password: "orgadmin")

// fetch a object by using a OR
def oid = "VR:wt.epm.EPMDocument:8483330"
def doc = getObjectFromOR(oid)

VCH.service.allIterationsOf(doc.master).each { obj ->
    println info(obj)
}
```

The output:

```bash
    $ groovy -cp $CLASSPATH wt_iterations.groovy
    k06__core-coil_cc0000000.asm, -.17 [RELEASED CAD-Dokument] OR:wt.epm.EPMDocument:112511174
    k06__core-coil_cc0000000.asm, -.16 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:105876837
    k06__core-coil_cc0000000.asm, -.15 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:95189959
    k06__core-coil_cc0000000.asm, -.14 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:86656605
    k06__core-coil_cc0000000.asm, -.13 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:67720806
    k06__core-coil_cc0000000.asm, -.12 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:57847775
    k06__core-coil_cc0000000.asm, -.11 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:46825566
    k06__core-coil_cc0000000.asm, -.10 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:44606524
    k06__core-coil_cc0000000.asm, -.9 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:39832260
    k06__core-coil_cc0000000.asm, -.8 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:31441018
    k06__core-coil_cc0000000.asm, -.7 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:31417993
    k06__core-coil_cc0000000.asm, -.6 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:30833875
    k06__core-coil_cc0000000.asm, -.5 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:28909290
    k06__core-coil_cc0000000.asm, -.4 [INWORK CAD-Dokument] OR:wt.epm.EPMDocument:11014342
```

Cool, eh?  The code *almost* looks like **coffee script** to me!

Using the GroovyConsole and the Object Browser
----------------------------------------------

I'm no big fan of using heavy UIs when programming -- I'm a avid user of *MacVIM* after all -- but
being able to *explore* live-objects sounded at least interesting to me.

So, I fired up the `groovyConsole`:

```bash
    $ groovyConsole -cp $CLASSPATH wt_docinfo.groovy 
```

![groovy object browser](/assets/images/groovy-object-browser.jpg)

Conclusion
----------

The experiments so far have been promising.  What I want to explore further is:

- runtime performance compared to **jython**
- importing **groovy** classes in java (should be a no-brainer)
- creating a servlet using **groovy**, using `Groovy Server Pages`
- all of the above together in the **Windchill** context
- doing a side-by-side comparsion of **jython** and **groovy** code
