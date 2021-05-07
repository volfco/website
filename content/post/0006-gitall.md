---
title: "Gitall"
date: 2018-09-29T11:36:33+08:00
draft: true
tags: 
  - gitall
  - project
---

== The Problem

GitHub is the go to default for a lot of us when searching for a library or piece of software. But there is more and more people switching over to GitLab- either the public one, or another public repo like GNOME's GitLab. 

This makes discovery hard. General search engines such as Bing are helpful at finding specific projects or code snippits, but it's hard to browse around topics and discover projects. So, let's make something!

== The Solution

GitAll is a project trying to make cross hub discovery easier. I'm in the process of indexing the metadata from GitHub and GitLab and putting it into a noramlized database for querying. 

The important bits are captured: name, description, topics, languages, and license. And search is run across all these attributes as well as simple discover by topic or language. 

Shout out to my friend Alex Cheng built out with the UI. Modern web development scares me.

In the current iteration, the backend is powered by a primitive Rust written server that is backed by a MySQL 8 server. Various indexers directly feed into the database; mostly written in python. Toshi (or ElasticSearch) to be added to support full text search of the repository description and possibly repository names. 

General Archatecture is DigitalOcean droplet hosting nginx for the frontend and the backend. MySQL is presently managed DigitalOcean for simplicity. 

===


== What's Next

The next major thing is to grow the database of public GitLab installations and starting to index them. If you have one or know of one, shoot me an email. I'll need an API key that gives me read access to everything you want indexed. The one condition is the repository is primarly home to Open Source software.  

Alternate platforms are also welcome. GitHub & GitLab are the two major ones I know of, but there must be others out there. sourcehut is on there as soon as it has a public directory of repos. Same condition as above applies- the focus of the platform should be open source development. 

After that, it's on to additional features. First is license discovery on GitLab; the API doesn't expost the info. Next is that I want to work on indexing READMEs for every major repository to improve search and whatever pops up. Only limitation is API rate limits. 

The lofty goal would be to deploy SourceGraph or a like product to provide code search across repositories. A large amount of disk would be needed, but I could see value and more importantly, it would be a fun challenge. 


== Will this be open sourced?

Eventually. The frontend and server are somewhat respectably built, but the indexers need 
