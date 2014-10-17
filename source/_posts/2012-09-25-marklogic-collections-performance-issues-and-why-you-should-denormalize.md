---
title: 'MarkLogic Collections &#8211; Performance Issues And Why You Should Denormalize!'
author: myoung
layout: post
comments: true
permalink: /post/marklogic-collections-performance-issues-and-why-you-should-denormalize/
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - MarkLogic
  - Performance
  - Programming
tags:
  - CCD
  - CRUD
  - fn:collection
  - marklogic
  - performance
  - programming
---
<!--more-->
## Background

Recently I&#8217;ve been on a new team at my work doing MarkLogic work. In a nutshell we&#8217;ve been developing a new HealthCare quality application from the ground up.  This would be our second application, but we learned enough from the first to make our priority creating a robust and scalable RESTful interface for document CRUD. With this we developed a middle tier that is almost completely configuration driven, meaning any new apps would be able to utilize this and not have to re-invent the wheel to get CRUD/Search working out-of-the-box.

## The Approach To A Problem Using Lists

The application is revolved around patient CCD&#8217;s (XML representing all of a patients history per retrieval). MarkLogic is ideal for this since it&#8217;s built on XQuery, a functional language revolved around XML. Let me paint you a picture of what led us to collection issues. Say you want as a doctor, to search &#8220;gender:M&#8221; on a database, and just keep tacking on until you get to what you want to find (end result maybe: &#8220;gender:M AND dob GT 01-10-1987 AND problem:Asthma&#8221;). Now as a doctor, I found X patients and want to save that as a NON changing list&#8230;meaning that if I search any time after this, I&#8217;ll still get those X patients, and no more that match that criteria since.

We implemented this using MarkLogic collections. Collections are handy in that you can take some documents, put them in a collection, and say:

```
for $i in fn:collection('/doctors/marc/1')
  return $i
```

That means whatever is in the collection &#8216;/doctors/marc/1&#8242; will always be there. Cool, that works great. So we implemented this &#8220;static list&#8221; as this:

1.  Take the query and execute it
2.  For each document in the result set, add them to a unique list: /lists/{unique id}
3.  change the query to list:{unique id} and add a new xml node <owner>{id of user}</owner>

The great thing about this, is you can have a collection constraint in our patients so you can just execute &#8220;list:{unique id}&#8221; and it will return all documents in that list. Awesome! It worked Great! Also note, yes we handled updating the list by re-executing it, and adding the new patients to that list. Deleting a list is also possible.

## The Problem With MarkLogic Collections

As I said, this solution worked **fantastically**&#8230;at first. It worked until we hit 1500 fairly large patient documents. The performance crawled to a stop. **The problem with collections is that the collection data is stored <span style="color: #ff0000;">internally. </span>**<span style="color: #ff0000;"><span style="color: #000000;">That means that if you decide to make a list for &#8220;gender:M&#8221;, you&#8217;re modifying and locking potentially half the documents in the database. For us, that meant a 9+ minute list creation. Not acceptable. You can try to speed this up, sure, by creating limits on how many documents you can modify, perhaps other ways involving spawning it in threads or in the background, but that still creates server overhead and gets rid of transaction safety.</span></span>

## Denormalize Your Database/The Solution

The greatest thing about a NoSQL database is that it can be denormalized. This may make some people cringe, why afterall would you want to duplicate data? If used correctly, it works great. For this specifically, it means that instead of modifying all documents in the database by adding/removing them from collections, is to use a separate file, stored as &#8216;/lists/{user}/{unique id}&#8217; that contains links (uri&#8217;s) to the documents. This is fine, especially for this application, since patient records are never deleted. I&#8217;ve shown an example further below on how to see it in action!

## The Code/TL;DR &#8211; Code or GTFO

The first thing you can do if you want to follow along is put a crap ton of delicious bacon documents into your database. You should copy/paste this into the QConsole and hit RUN! (it might take a few &#8211; it inserts 1200 documents&#8230;if it times out, insert 300 documents 4 times instead).  
<a href="https://gist.github.com/3785105" target=_blank>Click here for the code&#8230;too large to paste and the code isn&#8217;t too important</a>  
Now for the bad code. This will search for <beef>\*tips\*</beef> effectively. Any documents that match that, it will put it into a collection called &#8216;/marc/favorites&#8217;.

Then it will decide my favorites should be <pork>\*ham\*</pork>, so it will go through the collection, remove the originals, and add my new favorites. The important thing is to look at the number of documents affected and how long it takes! Ridiculous!

```
let $docs :=  
  fn:distinct-values(
    for $i in cts:search(//beef,cts:word-query("tip"))
      return fn:base-uri($i)
  )
return
  for $i in $docs return
    xdmp:document-add-collections(
      $i,
      '/marc/favorites'
    )
(: 781 documents affected - Profile 2105 Expressions PT57.822894S :)
 
let $remove-from-collections :=
  for $i in fn:collection('/marc/favorites')
    return xdmp:document-remove-collections(fn:base-uri($i),'/beef/tips')
    
let $docs := 
  fn:distinct-values(
    for $i in cts:search(//pork,cts:word-query("ham"))
      return fn:base-uri($i)
  )
 
return
  for $i in $docs return
    xdmp:document-add-collections(
      $i,
      '/marc/favorites'
    )
 
(: 1051 documents affected - Profile 4245 Expressions PT48.439176S  :)
```

So what&#8217;s the best way to do it? Create a new document of URI&#8217;s to the documents, and store that! Don&#8217;t believe me? Look at the performance without indexes!  

```
let $docs := 
  fn:distinct-values(
    for $i in cts:search(//beef,cts:word-query("tip")) return
      fn:base-uri($i)
  )
  
let $favorites :=
  <favorites>{
    for $i in $docs return
      <doc>{$i}</doc>
  }</favorites>
  
return
  xdmp:document-insert(
    '/marc/favorites',
    $favorites
  )
(: 1 document affected - Profile 2643 Expressions PT0.326612S :)
 
let $docs := 
  fn:distinct-values(
    for $i in cts:search(//pork,cts:word-query("ham")) return
      fn:base-uri($i)
  )
  
let $favorites :=
  <favorites>{
    for $i in $docs return
      <doc>{$i}</doc>
  }</favorites>
  
return xdmp:node-replace(fn:doc('/marc/favorites')/favorites,$favorites)
(: 1 document affected - Profile 2144 Expressions PT0.67532S :)
```

And to improve your search, you could use something like:

```
cts:search(/,
  cts:document-query(
    fn:doc('/marc/favorites')/favorites//doc/text()
  )
)
```
