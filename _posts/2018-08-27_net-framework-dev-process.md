---
layout: post
title: .NET Framework sample solution
subtitle: With tools
---

In this post I will describe different parts of a .NET full framework solution. This is the result of years reviewing and incorporating ideas from other people and adding my own experience and criteria. 

The solution was part of an exercise proposed in a technical interview. This is not intended to be an example from the code perspective itself. The solution proposed at that time constraint exercise was very basic (and is almost untouched). It just intend to be an example of a potential structure, process and use of tools.

You can find the source code in [GitHub](https://github.com/mamcer/atm-fun).

If you take a look at the code you may not agree with certain code conventions, for example: The tests projects should be suffixed Tests instead of Test, they will all be in a test/ directory at root level, private member should not include underscores, etc etc.

I remember to have disussions about where the open curly brace should be positioned. Also remember to review different code standards and write (in MS Word) the company .NET code guidelines to be followed by every project. 

The truth is that in many contexts there is no standard to rule them all. There are bad practices, good practices and not a single standard to follow. In that context I like to think that a good standard is one you feel confortable with and most importantly all the team agreed to follow, which make it the team standard.

This solution structure was evolving through years and there is no reason to stop that. If you point an enhacement in a nice way I will definitely take a look.

> About the opening curly brace, we all know it should be in it is own line. This is not Java.. joke

## Solution Structure

This is something I changed several times through the years. Also directly related with the source control system in use: trunk, branches and tags with SVN, Main, Dev and Release with TFS, etc. But that was many years ago. With Git (or any other descentralized SCM) you can have a more cleaner root folder. The proposed configuration is standard (I should add `test`) plus a `tool` directory, details below.


        doc/
        src/
        tool/
        build.cmd
        LICENSE
        open-cover.proj
        Settings.stylecop
        README.md
        vstest-console.proj

I really like the console/terminal. That is why I usually include several .cmd files for different actions like build, run tests, measure coverage, trigger a sonar-qube analysis, etc. They don't need to be in the source code repository. You can for example include them in a Continuous Integration server, copy the *.proj files and execute all the commands included in the .cmd files as tasks defined in your build plan. But if you want to quickly run all tests or review the code coverage increase of the last test in your local environment, this is an option.

![cmd files](https://raw.githubusercontent.com/mamcer/atm-fun/master/doc/cmd-files.png)

## Code

You can try to implement all the DDD layers in every project, I did. But for small/medium size projects I like the following standard structure (names can vary)

**Application**: Implement services which have direct access to the data layer. This layer can be directly consumed by a Web application or REST api for example, to expose those services.  
**Core**: Contains main application entities.  
**Data**: The implementation of the repositories and all the code that directly interacts with our data store.  
**Database**: A built in way to have the database included in the source repository. Some people like to have all the schema and scripts in the repository (and write diff scripts by hand). In several cases a combination of repository tagging between deployments and Entity Framework diff scripts does the trick, plus some level of sql scripts for non-schema changes.  
**Crosscutting**: Caching, dependency injection container, logging, etc. Divided in Core and MainModule to separate the definition with the specific implementation.  
**Common**: Here I usually include the Global Assembly Info configuration, the web deploy parameter files, Static code analysis exceptions file, etc.  
**Web, Rest API, Desktop App, Windows Service**: Any type of application that consumes the services defined in the Application layer.  

## Tools & Libraries

My set of tools have changed through time. Also my development environment: from Windows 98 (C++) to Windows 10 (with some Windows server versions in between). Since a couple of years ago my personal computer is based on Linux + .NET Core + [VSCode](https://code.visualstudio.com/) + Git + Docker + [Jetbrains Rider](https://www.jetbrains.com/rider/) (very good in most cases, but I like the minimalism of VSCode)

Here is a list of tools I used in the proposed solution. Not in particular order.

### StyleCop

I was a big fan of [Stylecop](https://github.com/StyleCop) (back in 2009). Writing a code standard guideline Word document and then customizing a StyleCop ruleset to enforce it. It definitely adds value but in my case the build penalty and check this kind of issues in code reviews makes me don't use it anymore.

### VS Code Analysis

I used to know it as [FXCop](https://en.wikipedia.org/wiki/FxCop).. Like with Stylecop I remember to spend several hours customizing a ruleset and apply it by default in all my projects. The build penalty is something to take into account, specially with big solutions.

### Unit Testing

I worked a lot with MSTest (by years the default Visual Studio option) try a couple of projects with [NUnit](https://nunit.org/) and lately moved to [xUnit](https://xunit.github.io/). The proposed example is based on MSTest + [Moq](https://github.com/moq/moq4).

I like to have a `vstest-console.cmd` file wich in combination with a `vstest-console.proj` look for all the test assemblies in the solution, run your tests and give you feedback in a matter of seconds:

![vstest console](https://raw.githubusercontent.com/mamcer/atm-fun/master/doc/vstest-console.png)

### Code Coverage

Some time ago you were tied to a higher than professional Visual Studio license in order have the code coverage functionality enabled. That was usually my case at work. But working on side projects, sometimes with [Visual Studio Express](https://en.wikipedia.org/wiki/Microsoft_Visual_Studio_Express), you need to implement something else. That was when I found [OpenCover](https://github.com/OpenCover/opencover) and since then use it in my side projects and also propose in my work environment, even if we have other commercial options available.

`open-cover.cmd` run all the tests and measure the code coverage producing an `open-cover.xml` file with the analysis result.

![open cover xml](https://raw.githubusercontent.com/mamcer/atm-fun/master/doc/open-cover.xml.png)

### Report Generator

[Report Generator](https://github.com/danielpalme/ReportGenerator) is a lighting fast must companion of OpenCover. It takes the xml report file generated by OpenCover and convert it to different *human friendly* formats, html for example. 

The file `report-generator.cmd` takes the open-cover.xml results file as input and using ReportGenerator generates a fully navegable html site in the source root coverage-report/ folder.

You can then host it with any web server and navigate the results

![report generator](https://raw.githubusercontent.com/mamcer/atm-fun/master/doc/report-generator.png)

In a Windows development environment you most probable have IISExpress already installed. By default it is installed in `C:\Program Files (x86)\IIS Express`

If you include it in your path environment variable then in an admin console you can run:

    iisexpress /path:[full-path-to-coverage-report-folder] -port:1010

An excelllent tool to have a quick local results site of the code coverage of a project. Or how it improves (if you are including new tests)

In some projects since OpenCover+ReportGenerator is a relatively fast process I even include it in a the Continuous Integration build. Deploying the results to a web server on every build. Leaving the full static code analysis and the rest of the automatic tests to a Nightly SonarQube metrics build.

### Web Deploy

[Web Deploy](https://www.iis.net/downloads/microsoft/web-deploy) is an amazing tool. The default and recommended option to deploy your .NET web applications, but it doesn stop there. You can use it for packaging, deployment, migrate servers, synchronize, etc. In my case I have just scratched the surface deploying web applications and synchronizing databases using a dacpac file.

One of the cool features from WebDeploy regarding web applications is that you can generate a deployment package once (with MSBuild) and deploy to N environments through parameter files. 

The batch file `web-deploy.cmd` included in the source code deploys the web application using WebDeploy taking a default Dev parameters file. You can add new parameter files that target another environments and deploy the same package changing just a few lines in the script.

### SonarLint

[SonarLint](https://www.sonarlint.org/) is in my opinion an example of a in theory great idea but the implementation limitations makes it something to evaluate before adopting. 

Have the opportunity to validate your code locally in your IDE against SonarQube rules is a great idea. Otherwise you commonly only have that kind of feedback daily, after a nightly long running build who calculates all the metrics and publish the results in a SonarQube server. 

Limitatons: at the moment of this write it doesn't support all the rules supported by the SonarQube server and it includes a very considerable build time penalty. 

I my case I prefer to have fast builds and run SonarQube analysis on demand from command line or from a build server plan.

![sonar lint](https://raw.githubusercontent.com/mamcer/atm-fun/master/doc/sonar-lint.png)

### SonarQube

If you have a Continuous Integration process in place, implement code reviews and have invested (or plan to invest) in automated tests. Then [SonarQube](https://www.sonarqube.org/) is your next natural step. Also you should consider automate your deployment.

Unless you are implementing a Continuous Inspection process, where every commit triggers a metric analysis in SonarQube. It is usually implemented as a single nightly build that provides the latest results at the beginning of the next day.

![sonar-qube](https://raw.githubusercontent.com/mamcer/atm-fun/master/doc/sonar-qube.png)

In the source code there is a `sonar-qube.cmd` file which triggers a SonarScanner.MSBuild.exe analysis to the SonarQube server. This is usually not neccesarily, you just need to configure a metrics build in your build server.

### Final Thoughts

This is just a potential sample solution that I was evolving trough years. All my side projects tend to follow variants of this structure and it is working fine so far. Hope you found some inspiration or new ideas to adopt in your development process.

Also all the .cmd files can be integrated in a build server to implement Continuous Integration (`build.cmd` + `vstest-console.cmd`), Continuous Delivery/Deployment (`web-deploy.cmd` + `db-deploy.cmd`) and Continuous Inspection (same as CI + `open-cover.cmd` + `sonarqube-scanner.cmd`). If you can run it in a console/terminal you can automate the process in a build server.