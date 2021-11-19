---
layout: single
title:  "Coverage enhancement for Azure DevOps Pull Requests"
date:   2021-11-18
categories: Android Mobile
---

Measuring unit test coverage in a pull request is an important tool. It allows 
understanding if enough tests are written in the new changes. If the code 
coverage decreased, it means that more tests need to be written. If the code 
coverage increased or remained unchanged, the pull request contains a sufficient 
number of tests.

We use [Azure DevOps Repos][azure-devops-repos] as our main
VCS service for our Android projects. At the moment, there is no possibility in it to
display the percentage of coverage by tests on the pull request page. However, he has
an open API that allows [post statuses][pull-request-status-api]
on the pull request page. We will use it to publish the coverage badge.

![PR coverage badge example][how-it-looks]

The most popular tool for evaluating test coverage in Java projects is [JaCoCo][jacoco],
Android projects use it too. It can publish a coverage XML report,
which we will use to extract the percentage of coverage and create a status with this percentage,
using the [Azure DevOps API to publish statuses][pull-request-status-api].

## JaCoCo C0 counter
[JaCoCo][jacoco] publishes reports in [xml format][jacoco-xml-report-format].
It contains information about various counters, which allow you to ultimately evaluate
a coverage. The simplest counter is C0 or Instructions.

> The smallest unit JaCoCo counts are single Java byte code instructions.

We will use it to estimate the unit-test coverage.

## Gradle plugin
For easy integration of the unit-tests coverage publication tool,
we created the [printcoverage][printcoverage] Gradle plugin. 
Setting it up is quite simple:

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

The plugin receives the [JaCoCo][jacoco] report XML file, the project coordinates in
[Azure DevOps Repos][azure-devops-repos], and [personal access token][personal-access-token],
which will be used to authorize HTTP queries into the Azure DevOps API.

You need to run the plugin after the tests have run (so that the 
[JaCoCo][jacoco] report has already been generated):
```shell
./gradlew test printCoverage
```

Plugin source code [available on GitHub][printcoverage]!

[azure-devops-repos]: https://azure.microsoft.com/en-us/services/devops/repos/
[pull-request-status-api]: https://docs.microsoft.com/en-us/rest/api/azure/devops/git/pull-request-statuses/create?view=azure-devops-rest-6.0
[personal-access-token]: https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page
[how-it-looks]: {{ site.url }}/assets/printcoverage/how-it-looks.jpg
[jacoco]: https://www.jacoco.org/jacoco/
[jacoco-counters]: https://www.jacoco.org/jacoco/trunk/doc/counters.html
[jacoco-xml-report-format]: https://www.jacoco.org/jacoco/trunk/coverage/report.dtd
[printcoverage]: https://github.com/intermedia-net/printcoverage