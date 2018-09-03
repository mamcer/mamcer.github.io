---
layout: post
title: I cleaned up my virtual basement
subtitle: Of archived side projects
---

Something that I quickly realized and definitely love from my profession (I'm a System Engineer) is how easy and cheap is for us to create compared to other disciplines. I'm not talking about big commercial systems, if you want to build the next GitHub you won't probably go far without some level of real money investment. I'm talking about create something, just because you are learning, trying a technology or building something to address a specific problem. At the very basics you only need a computer, the tools, time and focus.

Fortunately through time it was becoming easier and cheaper the access to technology. From hardware and also software perspective. For example in the .NET world not so long ago you needed a Visual Studio license for this kind of development, limited Express editions or alternative (very good) tools like [SharpDevelop](http://www.icsharpcode.net/opensource/sd/Default.aspx). Today you are not even tied to the Windows licensed platform anymore.

In that context and since I remember I always had one or several side projects in progress. Most of them never hit 'production' but the expression "The important thing is the journey, not the destination" is true in my case.

## The Destination

POCs, MVPs, get to the RTM point. 

## The Journey

Have the opportunity to work on technologies years before I had the opportunity to work with them in a paid job (some of them I never had the opportunity). Not in particular order: TFS, Mercurial, Git, WPF, Silverlight, Windows Phone applications, WebForms, ASP NET MVC, WebAPI, Continuous Integration/Delivery/Inspection, IIS configuration, VMs with different Linux distributions, AngularJS, Docker, .NET Core, SonarQube, real time functionality with SignalR, unit testing with mstest, xunit, etc etc

I learnt a lot of .NET just by testing the latest framework features as POCs or take a new release as an excuse to upgrade a previous side project (or completely rewrite it). I remember to have TFS in a virtual machine on my 2GB of ram laptop. Installing Atlassian JIRA, Bamboo and Crucible in my media center. Implement a continuous integration process in TF Online (Now Visual Studio Team Services). Playing with Git (working with VS Express for Web) years before I have the opportunity to use it in a job environment. Deploy web applications to local VMs running different Windows Server versions. And again a long etcetera.

Also have to mention the countless of medium/large systems implementing the latest technologies designed in paper that never reached even the MVP level. 

## The Basement

Well, the experience and learning is something that becomes part of you. But at some point last year (2017) I went down to my virtual basement where all the projects were archived. I took a look and realized, first that they were many of them, they were at different level of completeness, some of them still current, a good number outdated/obsolete and all representing hours or days of work. Finally they also share something: they were all there, hidden.

## The Selection

I found zip files with code in backups disks, private repositories in Visual Studio Team Services (since 2012), Bitbucket (since 2014) and local TFS and Stash (later Bitbucket Server) instances.

From the application perspective I found C++ programs from early 2000 to 'modern' .NET web applications and rest api from 2017. 

Near the end of 2017 I finished a list of the project I wanted to publish in GitHub. Initially I expected to have all of them migrated by the end of January 2018, then March, and then.. well it took a little more time but it is done. Initially I started to review them, reorganize the code, implement some cleanups and refactors with some ReSharper help, run SonarQube analysis and fix rules, add unit tests to increase coverage, etc. Then realized it will take me almost another ten years to finish so I published them with an original untouched branch and master with some level of reorder, cleanups and refactors (no SonarQube or code coverage improvement for most of them)

## The Final Result

A curated list of twenty five projects from early 2007 to 2017 published on GitHub. 

You may think that most of them are trash (and you are most probably right!) to me it was like an enjoyable time travel task and in some way gratifyingly like when you finish something that for some reason you postponed for a long time. 

## The List

They all share the 'Legacy project' repository description in GitHub

2017

[umovies](https://github.com/mamcer/umovies)  
A collection of applications to remotely play and control media files

2016

[ateam-docs](https://github.com/mamcer/ateam-docs)  
A best practices documentation site. An example implementation of a documentation continuous delivery schema.  

[undead-music](https://github.com/mamcer/undead-music)  
An application to create playlists and control the playback of music on remote computers in real time  

[upictures-net45](https://github.com/mamcer/upictures-net45)  
A web application that scan, process and allows easily search, tag and view pictures and videos from a media collection

2015

[bam-api](https://github.com/mamcer/bam-api)  
Using Atlassian Bamboo rest api it shows several best practices metrics  

[isofinder](https://github.com/mamcer/isofinder)  
Scan and index iso files allowing search and content download through a web application

[mentoring](https://github.com/mamcer/mentoring)  
A web application that help on the implementation of a company internal employee mentoring program

[usignalr-raspberry](https://github.com/mamcer/usignalr-raspberry)  
It allows remotely play music on a Raspberry Pi

[winform-actions](https://github.com/mamcer/winform-actions)  
An application that demo different Winforms desktop application functionalities

2014

[soulstone-2](https://github.com/mamcer/soulstone-2)  
Allows to manage different hosts (which provides music), manage playlists and playback of files remotely

[wcf-rest](https://github.com/mamcer/wcf-rest)  
An example of how to implement a REST api with WCF

2013

[execution-engine](https://github.com/mamcer/execution-engine)  
An intent to create a generic task execution library (I use it in other small projects)

[file-scanner](https://github.com/mamcer/file-scanner)  
A library that provides file system scan and analysis funtionality (I use in other projects like IsoFinder)

[skype-call](https://github.com/mamcer/skype-call)  
A POC application, you click on a web application link and your home media center video call you using Skype

2012

[media-center](https://github.com/mamcer/media-center)  
Responsive web application to remotely control playback of media files in a media center

2010

[great-shot](https://github.com/mamcer/great-shot)  
An application to quickly select your best pictures

[isomount](https://github.com/mamcer/isomount)  
A wrapper of slysoft Virtual Clone Drive to quickly mount iso files

[xgamer](https://github.com/mamcer/xgamer)  
A GUI to play retro games

2009

[lucky-dev](https://github.com/mamcer/lucky-dev)  
A WPF application that connects to Microsoft TFS and help you select the best code reviewer candidate from the team according to code reviews already assigned

2008

[am](https://github.com/mamcer/am)  
An application to play mp3 files with full keyboard control. Almost no UI, intended to be a good coding sessions companion with no disruption of the current task 

[arg-holidays](https://github.com/mamcer/arg-holidays)  
It connects to a WCF service and allows you to see information of upcoming (or past) Argentinian holidays

[nacion](https://github.com/mamcer/nacion)  
A simple web application to see information about a mortgage loan, a simulator to estimate interest earned, etc. Specifically instantiated with my mortgate loan at that moment.

[soulstone](https://github.com/mamcer/soulstone)  
A Silverlight application to play music in your browser 

2007

[countdown](https://github.com/mamcer/countdown)  
A simple countdown application for my wedding

[mp3-searcher](https://github.com/mamcer/mp3-searcher)  
Scan an index Windows network shared folders looking for mp3 files. Then it allows to search and play them

