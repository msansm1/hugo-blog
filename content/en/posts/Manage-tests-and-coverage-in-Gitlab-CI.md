---
title: "Manage tests and coverage in Gitlab-CI"
description: "How to manage tests and display coverage in Gitlab-CI."
# 1. To ensure Netlify triggers a build on our exampleSite instance, we need to change a file in the exampleSite directory.
theme_version: '2.8.2'
date: 2021-05-11
publishDate: 2021-05-11
cascade:
  featured_image: '/images/manage_test_coverage/manage_test_coverage_Pipeline_results.png'
---

When you have a CI, you want it to run your tests and show the test results, and what failed if that’s the case.
At Slickteam our CI runs on Gitlab-CI, and we manage our tests with it.
It helps us to find more easily what has failed with the integration inside Gitlab UI.
We also have displayed the global coverage result for all our tests.

Here is how we do it.

At first you must have tests in your code (who doesn’t ?) and run them with a build tool or something like that.
We use Gradle as our main build tool, so all my examples will have Gradle commands.

I have created unit and integration tests in the example project, and divided the run of the tests in 2 different stages in the pipeline configuration:
Stages configuration in gitlab-ci.yml file

For each stage we have the command to run the tests (script) if we want to allow failures on it and to get the artifacts.
Here I have allowed unit tests to fail as I want the 2 stages to be run on each pipeline run.

The artifact part is where you tell Gitlab-CI where your reports are, and how to manage it.
We set “when: always” as we want the reports to be always saved, and so we can have them on the UI for each run of the pipeline.
Gitlab-CI supports JUnit reports format, so nothing more to configure here.

In the IT_test stage I use a service, you can see in this article why I need it for integration tests.

Now, if you have a failed test, you can see the results like this in the UI:
![Gitlab-CI UI test result page with failed test and the stacktrace of the error](/images/manage_test_coverage/manage_test_coverage_Failed_test_UI.png "Failed test result in the UI, with the stack trace.")

Like that you will have the stack of the error directly in the UI, so you can find the origin of the issue more easily.

When you begin to have a lot of tests on your project, sometimes you want to know the coverage of these tests, to check how the code is covered and if some part are not enough tested.
We use JaCoCo on our Java projects to measure test coverage, and we configured our CI to get the global coverage result in the UI.

First, you must get the results from your tests report.
Since we have 2 stages for testing, we want to have the global results with unit and integration test coverage merged.
In the gitlab-ci.yml file, we have set JaCoCo report folders as artifacts for the 2 stages, and declare the stage unit_test as a dependency of IT_test.
Thus, the IT_test will get the artifacts from unit_test stage and JaCoCo will be able to calculate a global coverage for all the tests.

You can see that I rename 2 files in the script part of the stage, this is to avoid JaCoCo to overwrite the results from the previous stage.

Then you must set your tests result format for coverage in Gitlab-CI.
In Setting -> CI/CD, inside General pipelines section, you can set the format here:
![Gitlab-CI UI parameter page to set coverage format](/images/manage_test_coverage/manage_test_coverage_Result_format_UI.png "Manage your coverage results format in the UI.")

For my project, with JaCoCo, I use this regular expression:

```js
Total.*?([0–9]{1,3})%
```


With all this configured, you will have something like this in the UI:
![Gitlab-CI UI pipeline result page showing coverage results on the last test stage](/images/manage_test_coverage/manage_test_coverage_Pipeline_results.png "Pipeline result in Gitlab-CI showing coverage results on the last test stage.")

If we get the JaCoCo report from stage artifacts, it shows the same result:
![HTML report from JaCoCo execution showing the same result as in Gitlab-CI](/images/manage_test_coverage/manage_test_coverage_Result_report.png "HTML report from JaCoCo execution showing the same result as in Gitlab-CI.")

With that, I hope it will help you to have better information on your pipeline runs in Gitlab-CI, to make even better software :).

You can find the example project on github or on gitlab.
