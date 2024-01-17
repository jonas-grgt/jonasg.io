---
title: "My Open Code Overture" 
description: "As a fervent supporter of open source, I believe in not just writing code but opening it up for shared learning and collaboration"
draft: false
slug: "My Open Code Overture"
tags: ["projects", "java", "opensource"]
showDate: false
showAuthor: false
showReadingTime: false
showEdit: false
showComments: false
sharingLinks: false
---

{{< lead >}}
:books: Some noteworthy projects I've created along the way.
{{< /lead >}}

## Xjx

Java based XML serializing and deserializing (serdes) library.

🏡 homepage: https://github.com/jonasgeiregat/xjx

### Example deserialization usage:

```java
class Gpx {
    @Tag(path = "/gpx", items = "wpt")
    List<Wpt> wpts;
}

class Wpt {
    @Tag(path = "/gpx/wpt/name")
    String name;
}
```
```java
var gpx = new XjxSerdes().read("""
<gpx version="1.0"
     creator="ExpertGPS 1.1 - https://www.topografix.com"
     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xmlns="http://www.topografix.com/GPX/1/0"
     xsi:schemaLocation="http://www.topografix.com/GPX/1/0 http://www.topografix.com/GPX/1/0/gpx.xsd">
     
    <time>2002-02-27T17:18:33Z</time>
    <bounds minlat="42.401051" minlon="-71.126602" maxlat="42.468655" maxlon="-71.102973"/>
    <wpt lat="42.438878" lon="-71.119277">
        <ele>44.586548</ele>
        <time>2001-11-28T21:05:28Z</time>
        <name>5066</name>
        <desc><![CDATA[5066]]></desc>
        <sym>Crossing</sym>
        <type><![CDATA[Crossing]]></type>
    </wpt>
</gpx>
""", Gpx.class);
```
