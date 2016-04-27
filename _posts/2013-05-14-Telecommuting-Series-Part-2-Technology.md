---
layout: post
author: Kevin Baughman
author-image: kevin_baughman.jpg
title: Telecommuting Series, Part 2 - Technology
summary: As a developer we are expected to use or at least be aware of a number of different technologies.   Having a home office requires knowing more about these technologies as there may not be someone you can easily contact to help you when you run into any problems.

---

As a developer we are expected to use or at least be aware of a number of different technologies.   Having a home office requires knowing more about these technologies as there may not be someone you can easily contact to help you when you run into any problems.
 
Below are some of technologies a developer would to be aware of at Switchfly, either directly or indirectly:
 
## Client/Server:

* Couchbase Server – shared data
* Git/Gitlab – source control
* Postgres/Hibernate/Ehcache - database
* Apache/Tomcat – GUI
* Coldfusion/Railo – legacy architecture
 
## Other:

* Intellij IDEA – Java IDE
* Selenium – system/GUI testing
* Freemarker - GUI
* Spring/Guice
* Confluence - documentation
* Jenkins – continuous build
* Maven – build control
* Skype – voice
* Readytalk/Screenhero – screen sharing
* VPN/RDP
* Windows/Mac/Linux
 

For the last few years I’ve been using the corporate VPN and RDP from a personal Windows PC to access my corporate PC located remotely.  

### Advantages

* Most of the above technologies are located either on my remote PC or corporate servers (e.g., postgres server, couchbase server)
* IT can maintain/backup the PC
* I can access the corporate PC from any location in the world from just about any platform (e.g. an iPad or my Windows laptop)
 
### Disadvantages

* When my internet is down I can’t do any work
* When the VPN is down I can’t do any work
* When my corporate PC crashes I have to get someone at the corporate office reboot it
 
However, as my corporate PC was getting a bit old, I was given the option to switch to a Macbook Pro laptop.   I decided to go that route which has the following:

### Advantages

* I get a much faster machine
* Because it is much faster, I can locally run Postgres, Couchbase, and other servers
* I’m not dependent on the internet or the VPN (except on demand for GIT source code access)
* The Macbook Pro can replace my Window’s laptop (which I was taking with me whenever I traveled)
 
### Disadvantages

* I must maintain the servers
* I must perform regular backups
 
The decision on using a VPN to access your remote development machine or running everything locally doesn't really affect the next important part of working remotely… communicating with your co-workers.  We use tools like skype, readytalk, and Screenhero.  Some people, both remote and local, have started using Google Docs to collaborate on documents.