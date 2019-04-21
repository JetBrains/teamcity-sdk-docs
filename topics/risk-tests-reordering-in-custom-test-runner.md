[//]: # (title: Risk Tests Reordering in Custom Test Runner)
[//]: # (auxiliary-id: Risk+Tests+Reordering+in+Custom+Test+Runner.html)

In TeamCity, you can instruct the system to [run risk group tests before any others](https://www.jetbrains.com/help/teamcity/?running-risk-group-tests-first).

To implement the risk group tests reordering feature for your own custom test runner, TeamCity provides the following special system properties:
* __teamcity.tests.runRiskGroupTestsFirst__: this system property value contains groups of tests to run before others. Accordingly, there are two groups: __recentlyFailed__ and __newAndModified__. If more than one group is specified, they are separated with a comma. This property is provided only if corresponding settings are selected on the build runner page.
* __teamcity.tests.recentlyFailedTests.file__: this system property value contains the full path to a file with the recently failed tests. The property is provided only if the __recentlyFailed__ group is selected. The file contains tests separated by a new line. For Java\-like tests, full class names are stored in the file (without the test method name). In other cases, the full name of the test will be stored in the file as it was reported by the tests runner.
* __teamcity.build.changedFiles.file__: this system property is useful if you want to support running of new and modified tests in your tests runner. This property contains the full path to a file with the information about changed files included in the build. You can use this file to determine whether any tests were modified and run them before others. The file contains new\-line separated files: each line corresponds to one file and has the following format:
```shell
<relative file path>:<change type>:<revision>

```
where:
    * `<relative file path>` is the path to a file relative to the current checkout directory.
    * `<change type>` is a type of modification and can have the following values: `CHANGED`, `ADDED`, `REMOVED`, `NOT_CHANGED`, `DIRECTORY_CHANGED`, `DIRECTORY_ADDED`, `DIRECTORY_REMOVED`
    * `<revision>` is a file revision in the repository. If the file is a part of change list started via the [remote run](https://www.jetbrains.com/help/teamcity/?remote-run), then the `<personal>` string will be written instead of the file revision.

* __teamcity.build.checkoutDir_: this system property contains the path to the build checkout directory. It is useful if you need to convert relative paths to modified files to absolute ones.

<note>

TeamCity will pass the __teamcity.tests.runRiskGroupTestsFirst__, __teamcity.tests.recentlyFailedTests.file__ and __teamcity.build.changedFiles.file__ properties to the build process, but if the process starts an additional JVM or other processes, these properties won't be passed to them automatically.

For example, if you are using an Ant runner, you will have access to these properties from the Ant build.xml. But if your build.xml starts a new JVM (or `<junit/>` task with `fork="yes"` attribute), and you want to access these properties from this JVM, you'll have to modify your build script and pass them explicitly.
</note>

#### Known Limitations

If you have a package specified in the `TestNG xml` suite, reordering will not work: in this case TestNG itself searches for classes in packages and TeamCity cannot affect the way it sorts these classes. However, reordering will work if you specify concrete Test classes in the `xml` suite. Also, if you have several xml suites, reordering will work on the per\-suite basis.

 Having single classes in the XML suite may impose some inconveniences, e.g. developers have to remember to include classes in the suites. At the same time, this should speedup the tests startup, as the process of the searching classes by package is not that fast.
