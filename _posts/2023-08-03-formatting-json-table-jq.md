---
title: "Formatting JSON into a Table using JQ"
date: 2023-08-08
tags: tools json windchill nexiles
---

I recently had the need to get a nicely formatted table of build numbers and
plugin descriptions of the some customer Windchill Test System.  Because of
reasons, I only had access to the gateway API which provides this data in JSON
form:

```json
{
  "count": 11, 
  "items": [
    {
      "blueprint": "appservice", 
      "build": "949b5b8", 
      "date": "2023-04-20", 
      "description": "nexiles|gateway app service", 
      "doc_url": "https://windchill.example.test/Windchill/servlet/nexiles/tools/services/apps/static/docs/index.html", 
      "module": null, 
      "name": "nexiles.gateway.appservice", 
      "version": "1.9.9"
    }, 
    ...
     
    {
      "blueprint": "drawinglistexport", 
      "build": "c33ffa8", 
      "date": "2023-08-01", 
      "description": "Fancy Plugin Name", 
      "doc_url": "https://windchill.example.test/Windchill/servlet/nexiles/tools/services/drawinglistexport/static/docs/index.html", 
      "module": null, 
      "name": "another plugin", 
      "version": "3.2.1.0.dev0"
    }
  ]
}
```

Because of Yak Shaving, I wanted to use [jq](https://manpages.org/jq) to get the description and build
properties and format it into a table.

Turns out that [JQ is a beast](https://jqlang.github.io/jq/manual/#basic-filters) and Turing-complete (I guess). Also, [stack-overflow](https://stackoverflow.com/questions/32960857/how-to-convert-arbitrary-simple-json-to-csv-using-jq#32965227).

FWIW, this is what I ended up not using at all:

```bash
$ tabs 1,50,80
$ curl -s https://windchill.example.test/nexiles/servlet/nexiles/tools/plugins/index.json -u user:pass | jq -r '.items | ["description", "build"], map( [.[ "description", "build" ]])[] | @tsv'
description                                      build
nexiles|gateway app service                      949b5b8
nexiles|gateway attribute service                949b5b8
nexiles|gateway file service                     949b5b8
nexiles|gateway number service                   949b5b8
nexiles|gateway principal service                949b5b8
nexiles|gateway query service.                   949b5b8
nexiles|gateway zip service                      949b5b8
Fancy Plugin Name                                c33ffa8
```

A nice little find is the [tabs](https://manpages.org/tabs) command – this sets
tab stops in your terminal.  So effectively the JQ command does:

- build a header array
- build a array of columns
- use the JQ internal @tsv filter to output tab-separated-values
- The terminal then interprets the tabs and uses the tab stops at position 1, 50 and 80 to display the result.

BTW, the intermediate format used for the @tsv filter looks like this:

```json
[
  "description",
  "build"
]
[
  "nexiles|gateway app service",
  "949b5b8"
]
[
  "nexiles|gateway attribute service",
  "949b5b8"
]
[
  "nexiles|gateway file service",
  "949b5b8"
]
[
  "nexiles|gateway number service",
  "949b5b8"
]
[
  "nexiles|gateway principal service",
  "949b5b8"
]
[
  "nexiles|gateway query service.",
  "949b5b8"
]
[
  "nexiles|gateway zip service",
  "949b5b8"
]
[
  "Fancy Plugin Name",
  "c33ffa8"
]
```
