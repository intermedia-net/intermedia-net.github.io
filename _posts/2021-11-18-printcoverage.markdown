---
layout: single
title:  "Unit Test Coverage Enhancements for Azure DevOps Pull Requests"
date:   2021-11-18
categories: Android Mobile
---

Hello again from the Intermedia Mobile Team!

As the title suggests, in this article we are going to talk about unit test coverage calculations when Azure DevOps is used as your main VCS for Android projects. As part of this discussion, we will introduce you to a new tool we developed that solves for a common problem – the ability (or, in this case, the inability) to display unit tests coverage results right on the Azure DevOps pull request page.

Measuring unit test coverage for a newly submitted pull request (PR) is very important. It provides a clear understanding of how many new tests were written and if this amount was enough for covering changes introduced in the PR. If the code coverage decreased this means that more tests need to be written to cover new changes. On the other hand if the coverage increased or at least remained unchanged, we are most likely good to go and have a sufficient amount of new tests.

We use [Azure DevOps Repos][azure-devops-repos] as our main
VCS service for our Android projects. At the moment, there is no option to
display the percentage of unit tests coverage on the pull request page. However, Azure DevOps
does provide an API that allows one to [post various statuses][pull-request-status-api]
on the pull request page. We began using this API to publish our unit tests coverage badge for every new PR.

Here is how it looks like:
![PR coverage badge example][how-it-looks]

The most popular tool for collecting test coverage data in Java projects is [JaCoCo][jacoco].
Android projects use it too. It can generate a coverage report in XML format from which we can extract the percentage of unit tests coverage and publish a status with this percentage
using the [Azure DevOps API][pull-request-status-api].

## JaCoCo C0 counter
As mentioned, JaCoCo publishes reports in [xml format][jacoco-xml-report-format].
The report contains information about various counters which allow us to evaluate
code coverage from various perspectives. The simplest counter is C0 or "Instructions" counter.

> The smallest units JaCoCo counts are single Java byte code instructions.

To configure JaCoCo [xml reports using gradle][jacoco-gradle-report-configuration]
we need to define the `jacocoTestReport` task attribute:
```groovy
jacocoTestReport {
  reports {
    xml.required = true
    xml.destination file(
      "${project.buildDir}/outputs/jacoco/report.xml"
    )
  }
}
```

It produces `report.xml` file which looks like this:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!DOCTYPE report
  PUBLIC '-//JACOCO//DTD Report 1.1//EN'
  'report.dtd'>
<report name="<gradle-project-name>">
  <sessioninfo dump="1637724811203" id="<jacoco-session-id>" start="1637724520065"/>
  <package name="<package-name>">
    <class name="<class-name>" sourcefilename="FileName.kt">
      <method desc="(Lkotlin/jvm/functions/Function0;)V" line="94" name="&lt;init&gt;">
        <counter covered="9" missed="0" type="INSTRUCTION"/>
        <counter covered="1" missed="0" type="LINE"/>
        <counter covered="1" missed="0" type="COMPLEXITY"/>
        <counter covered="1" missed="0" type="METHOD"/>
      </method>
      <method desc="()Lkotlin/jvm/functions/Function0;" line="94" name="getBlock">
        ...
      </method>
      <method desc="()Z" line="97" name="execute">
        ...
      </method>
      <counter covered="9" missed="9" type="INSTRUCTION"/>
      <counter covered="2" missed="0" type="LINE"/>
      <counter covered="1" missed="2" type="COMPLEXITY"/>
      <counter covered="1" missed="2" type="METHOD"/>
      <counter covered="1" missed="0" type="CLASS"/>
    </class>
    ...
  </package>
  ...
</report>
```

We will use `INSTRUCTION` counter from this report to estimate the unit test coverage:

```kotlin
var missed = 0 // sum of all INSTRUCTION missed
var covered = 0 // sum of all INSTRUCTION covered
...
val coverage = 100.0 / (missed + covered) * covered
```

## Gradle plugin
After obtaining the coverage data from the JaCoCo report we need to somehow display this information on the Azure DevOps pull request page. As previously noted, we couldn’t find a suitable tool that would allow this to happen…so…we decided to create our own.

We developed the [printcoverage][printcoverage] Gradle plugin.
Configuring it is quite simple:

```groovy
printCoverage {
  setPrinterFactory(
    new AzurePrinterFactory(
      new AzureRepo(
        "<azure token>",
        "<azure base url>",
        "<azure organization>" ,
        "<azure project>",
        "<azure repo>"
      )
    )
  )
  setJacocoReportFile(file("${project.buildDir}/outputs/jacoco/report.xml"))
}
```

You need to set [Azure access token][personal-access-token] in order to allow the plugin to use Azure API, configure your [Azure DevOps][azure-devops-repos] project information and specify the path to JaCoCo report file.

You then need to run the plugin after tests execution (so that the 
JaCoCo report has already been generated):

```shell
./gradlew test printCoverage
```

Of course, we wouldn’t be sharing this information if we weren’t planning to share our tool with the community! This Print Coverage plugin source code is [available on our GitHub page][printcoverage] under MIT license!

Please let us know of any success or issues you experience with it!

[azure-devops-repos]: https://azure.microsoft.com/en-us/services/devops/repos/
[pull-request-status-api]: https://docs.microsoft.com/en-us/rest/api/azure/devops/git/pull-request-statuses/create?view=azure-devops-rest-6.0
[personal-access-token]: https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page
[how-it-looks]: {{ site.url }}/assets/printcoverage/how-it-looks.jpg
[jacoco]: https://www.jacoco.org/jacoco/
[jacoco-counters]: https://www.jacoco.org/jacoco/trunk/doc/counters.html
[jacoco-xml-report-format]: https://www.jacoco.org/jacoco/trunk/coverage/report.dtd
[jacoco-gradle-report-configuration]: https://docs.gradle.org/current/userguide/jacoco_plugin.html#sec:jacoco_report_configuration
[printcoverage]: https://github.com/intermedia-net/printcoverage