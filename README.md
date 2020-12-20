# Introduction #
I'm making this demo repo and writeup because it was surprisingly and
frustratingly difficult to get Xamarin.UITest tests to run on a
Microsoft-hosted agent in an Azure DevOps pipeline.  NO App Center.  NO
self-hosted agents.  I just wanted to do everything in Azure DevOps.

So, this demo shows how to accomplish that, and some other common desires for
an Azure Devops continuous integration pipeline for the Android portion of a
Xamarin app...

* Each build gets its own `versionCode` and `versionName`.
* Build the APK.
* Sign the APK.
* Publish the APK as a pipeline artifact.
* Do unit tests (NUnit).
* Do UI tests (Xamarin.UITest), which involves several Android emulator steps.
* Publish test results.

This demo is not about getting started on unit testing or UI testing; the demo
is about getting these things to also work in an Azure DevOps pipeline.

You can see a
[successful run](https://jmegner.visualstudio.com/Demos/_build/results?buildId=4&view=results),
a
[successful job overview](https://jmegner.visualstudio.com/Demos/_build/results?buildId=4&view=logs),
[published artifacts](https://jmegner.visualstudio.com/Demos/_build/results?buildId=4&view=artifacts&pathAsName=false&type=publishedArtifacts),
and
[unit+UI test results](https://jmegner.visualstudio.com/Demos/_build/results?buildId=4&view=ms.vss-test-web.build-test-results-tab)
(also alernate view for
[unit test run](https://jmegner.visualstudio.com/Demos/_testManagement/runs?_a=runCharts&runId=1000012)
and
[UI test run](https://jmegner.visualstudio.com/Demos/_testManagement/runs?runId=1000014&_a=runCharts)).

# Notable Files #
The
[XamarinPipelineDemo.Android/AzureDevOps/](XamarinPipelineDemo.Android/AzureDevOps/)
folder has most of the notable files...
* [pipeline-android.yml](XamarinPipelineDemo.Android/AzureDevOps/pipeline-android.yml):
  the pipeline definition and heart of this demo.
* [AndroidSetVersion.ps1](XamarinPipelineDemo.Android/AzureDevOps/AndroidSetVersion.ps1):
  the script that manipulates the Android manifest file to update the
  `versionName` and `versionCode` attributes.
* [example.keystore](XamarinPipelineDemo.Android/AzureDevOps/example.keystore):
  for signing the APK.  Normally keystore files are sensitive and you wouldn't
  put them (and their passwords) in your repo, but this is a demo.

[XamarinPipelineDemo.UITest/AppInitializer.cs](XamarinPipelineDemo.UITest/AppInitializer.cs):
the autogenerated AppInitializer.cs has been modified so that you can specify
which APK file to install for testing, or which keystore to match an already
installed (signed) APK. I suggest the APK file methodology.

[uitest\_run.ps1](LocalScripts/uitest_run.ps1): script to run
UITest tests in way most similar to how the pipeline will do it.

[Screenshots](Screenshots) folder has some screenshots of the results of a
working pipeline run, and some of the web interface you need to tangle with to
get the pipeline working.

# Getting Started, Local Machine #
First, check that it works on your machine.  Open the solution in Visual Studio
2019, and deploy the Release build to an Android emulator or connected Android
device (just select Release build configuration and launch the debugger).  The
app should show you a page with a label that says "Some text.".

In Visual Studio's test explorer, run both the Nunit and UITest tests.
Everything should pass.

Also, to run the UITest tests in the way most similar to how the pipeline will
do it, install
[nunit3-console](https://github.com/nunit/nunit-console/releases), go into the
`XamarinPipelineDemo.UITest` folder and run `uitest_run.ps1`.  You'll get a
test results file `TestResult.xml` and a detailed log `uitest.log` that is
useful for troubleshooting. The script tries to use `adb` and `msbuild` from
your `PATH` environment variable and a few other locations.  You might have to
add your `adb` or `msbuild` directories to your `PATH`. Also, you might have to
set the `ANDROID_HOME` environment variable to something like `C:\Program Files
(x86)\Android\android-sdk` and the `JAVA_HOME` environment variable to
something like `C:\Program Files\Android\jdk\microsoft_dist_openjdk_1.8.0.25`

# Getting Started, Azure DevOps #
In the Pipelines => Library section of your Azure DevOps project, you need to
do a few things...

* Upload example.keystore as a secure file.
* Create a variable group named `android_demo_var_group`. In it,
  create the following variables...
    * `androidKeystoreSecureFileName: example.keystore`
    * `androidKeyAlias: androiddebugkey`
    * `androidKeystorePassword: android`
    * `androidKeyPassword: android`
* Make the `androidKeystorePassword` and `androidKeyPassword` secret by clicking the padlock icon.

Upload the repo to Azure DevOps. Create a new pipeline, choose "Azure Repos
Git", select the XamarinPipelineDemo repo, select "Existing Azure Pipelines
YAML file", and select the
`XamarinPipelineDemo/XamarinPipelineDemo.Android/AzureDevOps/pipeline-android.yml`
as the existing yaml file.

Run the pipeline and you'll have to click a few things to give the pipeline
access to these secure files and secret variables. To grant permission to the
pipeline, you might have to go to the run summary page (see `Screenshots`
folder).

# Explanation Of The Journey: Deadends, Pitfalls, Solutions #

Note that things may change after this demo was made (2020-12-19).  Some
limitations may go away, and some workaround no longer needed.  I'd love to
hear about them if you ever run across a way this demo should be updated.

## Must Use MacOS Agent ##

First of all, doing Xamarin.UITest tests on Microsoft-hosted agent in an Azure
DevOps pipeline has some important constraints.  Microsoft-hosted agents for
Window and Linux run in a virtual machine that can not run the Android
emulator, so only the MacOS agent can run the Android emulator ([MS docs
page](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/android?view=azure-devops&viewFallbackFrom=vsts#test-on-the-android-emulator)).

With a self-hosted agent that is not a virtual machine, you can use any of the
three OSes.  With App Center, the tests are run on a real device, and there is
no need to run the Android emulator, so again, you can use any of the three
OSes.

The MacOS agent has a few pitfalls to watch out for.

* MacOS must use Mono when dealing with .net framework stuff (originally made
  just for Windows).  So, .net framework stuff that works on your Windows
  machine may not do so well in the pipeline.  Try to make your project target
  .net core or .net 5 where possible.  Of special note, make your Nunit
  unit-test project .net core or .net 5.
    * Of special note, you can't use DotNetCoreCLI task on a MacOS agent to run
      test projects that target .net framework.
      [Mono's open issue 6984](https://github.com/mono/mono/issues/6984) says that you can do
      "dotnet build" on a .net framework project, but you can't "dotnet test".
* Xamarin.UITest MUST be .net framework, so you can not use DotNetCoreCLI task
  to run Xamarin.UITest tests.
* MacOS agent also doesn't support VSTest or VsBuild tasks.
* The only thing left to do for Xamarin.UITest is a MSBuild task to build it,
  then directly run nunit3-console to run the Xamarin.UITest tests.
* MacOS agents are case sensitive for path stuff while Windows is not, so make
  sure your pipeline stuff is case-appropriate.
* On Windows, you might be used to using Unix-inspired PowerShell aliases like
  "ls" and "mv".  Do not use those aliases.  In MacOS, even inside a PowerShell
  script, commands like "ls" will invoke the Unix command instead of the
  PowerShell cmdlet.

### MacOS Agent Directories ###

For pipelines, there are three major directories to think about
* `Build.SourcesDirectory`, which is often `/Users/runner/work/1/s/`.
* `Build.BinariesDirectory`, which is often `/Users/runner/work/1/b/`.
* `Build.ArtifactStagingDirectory`, which is often `/Users/runner/work/1/a/`.

The repo for your pipeline is automatically put in the
`Build.SourcesDirectory`.  The other two directories are just natural places
for you to put stuff. For instance, build outputs to `Build.BinariesDirectory`
and special files you want to download later (artifacts) to
`Build.ArtifactStagingDirectory`.  The `PublishBuildArtifacts` even defaults to
publishing everything in the `Build.ArtifactStagingDirectory`.

## Fresh Autogenerated Pipeline ##

If you make a new pipeline for a Xamarin Android app, you get an autogenerated
yaml file that...
* Triggers on your main or master branch.
* Selects desired agent type (pool's vmImage value).
* Sets buildConfiguration and outputDirectory variables.
* Does usual nuget stuff so you download the nuget packages used by your solutions.
* The XamarinAndroid task builds all "`*droid*.csproj`" projects (probably just
  one for you), generating an unsigned APK file.

That's it.  You can't even access the unsigned APK file after the pipeline
runs; you just get to know whether the agent was able to make the unsigned APK.

I'll explain how and why we add to the pipeline to accomplish the goals I
mentioned in the introduction.

## General YAML/Pipeline Advice ##

You're going to have to learn nuances of yaml.  If you don't already know yaml
and the unique quirks of pipeline yaml, it's going to trip you up somewhere.

### Pipeline Variables ###

One of the first learning hurdles for dealing with pipeline is learning enough
to use variables effectively.

The variable section in a fresh autogenerated pipeline looks like this...
```
variables:
  name1: value1
  name2: value2
```
...which is nice and compact.  But if you need to use a variable group,
[you have to](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=yaml#use-a-variable-group)
go with the more verbose way...
```
variables:
  - group: nameOfVariableGroup
  - name: name1
    value: value1
  - name: name2
    value: value2
```

I still haven't read the entire
[MS Docs page on pipeline variables](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch)
because it is so long.  Unfortunately there are
[three different syntaxes](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#understand-variable-syntax)
for referencing variables.  You can mostly use
[macro syntax](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#macro-syntax-variables),
which looks like `$(someVariable)`.

If the pipeline encounters `$(someVariable)` and doesn't recognize
`someVariable` as a variable, then the expression stays as is (because maybe
it'll be usable by PowerShell or whatever you're executing).

So, if you get errors that directly talk about `$(someVariable)` rather than
the value of `someVariable`, then `someVariable` isn't defined.  You need to
check your spelling, and if it's a variable from a variable group (defined in
Library section of web interface), you need to explicitly reference the
variable group.

My pipeline yaml uses macro syntax all over the place. The only time I
don't use macro syntax is when I use
[runtime expression syntax](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#runtime-expression-syntax)
(`$[variables.someVariable]`) in conditions and expressions, as is
[recommended](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#what-syntax-should-i-use).
You can see the runtime expression syntax in my pipeline's step conditions, just search
for "condition:" or "variables.".

Also, non-secret variables are
[mapped to environment variables](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#access-variables-through-the-environment)
for each task.

### Pipeline Triggers ###

A freshly autogenerated pipeline might a
[trigger](https://docs.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git?view=azure-devops&tabs=yaml#ci-triggers)
section...
```
trigger:
- main
```
...which will make the pipeline trigger for every change to the main branch.
But if you have multiple target platforms (android, iOS, uwp), each having
their own pipeline, then you get a lot of unnecessary builds when you update
something only relevant to one platform.

So, you probably want a
[path-based trigger](https://docs.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git?view=azure-devops&tabs=yaml#paths).
Note that wildcards are unsupported and all paths are relative to the root of
the repo.  Here's a trigger section for a hypothetical android pipeline...
```
trigger:
  branches:
    include:
    - main
  paths:
    include:
    # common
    - 'MyApp'
    - 'MyApp.NUnit'
    - 'MyApp.UITest'
    - 'Util'
    - 'XamarinUtil'
    - 'MyApp.sln'
    # platform
    - 'MyApp.Android'
```

Also, this path-based trigger stuff is why this demo's android pipeline yml
file and android version script are under
[XamarinPipelineDemo.Android/AzureDevOps](XamarinPipelineDemo.Android/AzureDevOps)
rather than under a root-level AzureDevOps folder. A change to these
android-pipeline-specific file should only trigger an android pipeline build,
and putting them under an android folder makes that easy trigger-wise.

Similarly,
[uitest\_run.ps1](LocalScripts/uitest_run.ps1) is in a LocalScripts folder
instead of the XamarinPipelineDemo.UITest folder because changes to a
local-use-only script should not trigger a pipeline build.  There is also the
option of having a XamarinPipelineDemo.UITest/LocalScripts folder and listing
that folder in the yaml's trigger-paths-exclude list.

### Pipeline Tasks ###

Some tasks support path wildcards in their inputs, some don't.  Always check the
[task reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/?view=azure-devops)
before using path wildcards.

Sometimes the task is a wrapper around some tool, and the task's documentation doesn't go into much detail into the behavior of the tool.  For instance,
[AndroidSigning](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/android-signing?view=azure-devops)
is a wrapper around
[apksigner](https://developer.android.com/studio/command-line/apksigner),
and you have to get all the way down to the `--out` option section of the
apksigner doc to learn that the _absence_ of the option leads to the APK file
being signed in place, overwriting the input APK.

## Give Each Build An Increasing Android App Version ##

If the Azure DevOps pipeline is going to be making the APKs we'll be releasing,
we need unique `versionCode` and `versionName` values for each build.

[Reminder](https://developer.android.com/studio/publish/versioning#appversioning):
`versionCode` is a positive integer that is used by Android to compare versions
and is not shown to the user.  `versionName` is text displayed to the user and that
is its only use.

Short version: The 'Set build-based Android app version' task uses the YAML
counter function on the pipeline name (`Build.DefinitionName`) to set the
`versionCode` and the `Build.BuildNumber` to set the `versionName`.  This task
is executed right before the XamarinAndroid build task and calls a PowerShell
script to modify the Android manifest file.

### How To Set The Android App Version ###

James Montemagno's and Andrew Hoefling's "Mobile App Tasks for VSTS"
([Azure DevOps plugin](https://marketplace.visualstudio.com/items?itemName=vs-publisher-473885.motz-mobile-buildtasks),
[Github repo](https://github.com/jamesmontemagno/vsts-mobile-tasks))
has an AndroidBumpVersion task does half of the job: setting the `versionCode` and `versionName`.

Some people are not allowed to use Azure DevOps plugins (perhaps for security
by their employer), so we will not use this as a plugin. Azure DevOps plugins
are run via a Node server, so the plugin would use
[`tasks/AndroidBumpVersion/task.ts`](https://github.com/jamesmontemagno/vsts-mobile-tasks/blob/master/tasks/AndroidBumpVersion/task.ts),
but thankfully James has also provided PowerShell and bash equivalents of his
plugin tasks, so you can look at those files.

I went with his PowerShell script, fixed a bug, and cleaned it up ([pull
request 39](https://github.com/jamesmontemagno/vsts-mobile-tasks/pull/39)).
The result is this demo's
[AndroidSetVersion.ps1](XamarinPipelineDemo.Android/AzureDevOps/AndroidSetVersion.ps1).

(Note: recent versions of PowerShell are cross platform, so you can run
PowerShell on MacOS and Linux. But again, be mindful of Unix commands
overriding PowerShell aliases and you can't be case-insensitive.)

The essence of the script is that the Android manifest file is XML and inside
the `manifest` element,  set the `android:versionCode` and
`android:versionName` attributes appropriately.  Thankfully PowerShell has the
[XmlDocument](https://docs.microsoft.com/en-us/dotnet/api/system.xml.xmldocument?view=net-5.0)
class and the
[Select-XML](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-xml?view=powershell-7.1#outputs)
cmdlet that gives you easy-to-manipulate
[SelectXmlInfo](https://docs.microsoft.com/en-us/dotnet/api/microsoft.powershell.commands.selectxmlinfo?view=powershellsdk-7.0.0)
objects.

### How To Choose The Version ###

The second half of the problem is how to have an increasing and meaningful
`versionCode` and `versionName`.  Azure DevOps pipelines will have [pre-defined
variables](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml),
including...
* `Build.BuildId`: a positive integer that is build id that is unique across
  your
  [organization](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/organization-management?view=azure-devops)
  and will appear in the build's URL (ex:
  `dev.azure.com/SomeOrganization/SomeProject/_build/results?buildId=123456`).
* `Build.BuildNumber`: a string (not a number, especially if you set the
  [`name` variable](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/run-number?view=azure-devops&tabs=yaml).
  The [default format](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/run-number?view=azure-devops&tabs=yaml#tokens)
  is "`$(Date:yyyyMMdd)$(Rev:.r)`", which looks like "20201231.7" and is unique
  only within the pipeline.
* `Build.DefinitionName`: the name of the pipeline.

I think that the default `Build.BuildNumber` makes sense for `versionName`;
it's unique, increasing, and easy for you to lookup the build/commit for the
version name a user sees.  I don't like `Build.BuildId` for `versionCode`
because consecutive builds will probably not have consecutive `versionCode`
values because of all the other builds in your Azure DevOps organization.
`Build.BuildId` is probably just going to be a large, meaningless number for
you.

Thankfully, Andrew Hoefling wrote “[Azure Pipelines Custom Build Numbers in
YAML
Templates](https://www.andrewhoefling.com/Blog/Post/azure-pipelines-custom-build-numbers-in-yaml-templates)”,
which shows how you can get a simple {1,2,3,...} progression for a build using
the yaml [counter
function](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/expressions?view=azure-devops#counter).
MS docs on defining pipeline variables has a [counter
example](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#set-variables-by-using-expressions)
too.

Here's a snippet that shows a simple `pipelineBuildNumber` that goes up
{0,1,2,...} and a `versionRevision` that counts up but gets reset everytime you
change the `versionMajorMinor` value.
```
variables:
  # for doing Major.Minor.Revision;
  # any time you change versionMajorMinor,
  # versionRevision uses a new counter
  - name: 'versionMajorMinor'
    value: '0.0'
  - name: 'versionRevision'
    value: $[counter(variables['versionMajorMinor'], 0)]
  # for doing simple pipeline build counter
  - name: 'pipelineBuildNumber'
    value: $[counter(variables['Build.DefinitionName'], 1)]
```

## Sign The APK File ##

### Keystore Background ###

You'll want to sign the APK file so it can be installed on users' devices and
distributed on Google Play.  This repo already comes with a keystore file
(remember: don't put your keystore in your repo; it should be more tightly
controlled and uploaded as a secure file to Azure DevOps), but you can create
your own keystore by following these [MS Docs
instructions](https://docs.microsoft.com/en-us/xamarin/android/deploy-test/signing/)
(don't do the "Sign the APK" section).

You might get confused that if you make a keystore in Visual Studio, you have
to choose a "keystore password", but not a "key password", and lots of other
places talk about the "key password".  The "[key and certificate
options](https://developer.android.com/studio/command-line/apksigner#options-sign-key-cert)"
section of the apksigner doc might help you understand.  A keystore can contain
multiple keys, each identified by a key alias.  The keystore itself
password-protected, and each key might have its own password.  This [keytool
example](https://docs.microsoft.com/en-us/xamarin/android/deploy-test/signing/manually-signing-the-apk)
makes me think a common behavior is for a key password to default to the same
as the keystore password.

One approach that has worked for me so far: when you are asked for a key
password, and you don't recall there being a key password, you can probably put
the keystore password.

Another confusion you may have is that the
[AndroidSigning@3](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/android-signing?view=azure-devops)
task has an input named "keystoreAlias" (also called "apksignerKeystoreAlias"),
but keystores do not have aliases; keys within keystores have aliases.  You
specify the keystore by the file name, then you specify the key by the key's
alias. I have reported this misnaming as a
[problem on Developer Community](https://developercommunity.visualstudio.com/content/problem/1292515/androidsigning.html).

### AndroidSigning Task ###

This is the demo's
[AndroidSigning](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/android-signing?view=azure-devops)
task (and required reference to appropriate variable group)...
```
variables:
  - group: android_demo_var_group

...

- task: AndroidSigning@3
  displayName: 'sign APK with example keystore'
  inputs:
    apkFiles: '$(outputDir)/*.apk'
    apksignerKeystoreFile: '$(androidKeystoreSecureFileName)'
    apksignerKeystoreAlias: '$(androidKeyAlias)'
    apksignerKeystorePassword: '$(androidKeystorePassword)'
    apksignerArguments: '--verbose --out $(finalApkPathSigned)'
```

The task doc says it accepts wildcards for `apkFiles`.  (Remember, don't assume
tasks accept wildcards, check the task doc). Also, the doc states that the
referenced keystore file _must_ be a secure file, which should be fine for you.
However, if you want to get around this restriction, you could use a
[PowerShell task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/powershell?view=azure-devops)
to call apksigner directly.

If you get errors like "can't find $(someVariable) secure file", that means the
`someVariable` is not defined.  Check that you are referencing the appropriate
variable group in your yaml's `variables` section, and check that
`someVariable` exactly matches what you have in your variable group.

By default, apksigner overwrites the APK file, and therefore the AndroidSigning
task overwrites the APK file, which could be fine for you.  But I wanted the
signed APK to go into the artifact staging directory (path held in predefined
variable `Build.ArtifactStagingDirectory`) with a particular file name (not the
default `com.demo.XamarinPipelineDemo.apk`), so I used
[apksigner's `--out` argument](https://developer.android.com/studio/command-line/apksigner#options-sign-general).

Note that `finalApkPathSigned` puts the `Build.BuildNumber` and
`pipelineBuildNumber` in the file name.

If you ever want to double check whether an APK has been signed, and by which
keystore, use apksigner (possibly at `C:\Program Files
(x86)\Android\android-sdk\build-tools\SOME_VERSION\apksigner.bat`). Do
`apksigner verify --print-certs THE_APK_PATH` and the first line tells you
about the key that signed the APK or `DOES NOT VERIFY` if not signed.

Likewise, for looking at keystore files, you can use keytool (possibly at
`C:\Program
Files\Android\jdk\microsoft_dist_openjdk_1.8.0.25\bin\keytool.exe`).  `keytool
-v -list -keystore KEYSTORE_PATH` will tell you about keys in the keystore,
even if you provide no keystore password.

## Publish The APK File(s) As Build Artifacts ##
TODO

## Build And Run Unit Tests ##
TODO

## Run AppCenter UI Tests ##
TODO (and remember the pitfalls of Xamarin.UITest's nuget folder (test-cloude.exe!), and you need Node v10)

## Set Up And Start Android Emulator ##
TODO

## Build UI Tests ##
TODO

## Run UI Tests ##
TODO

TODO: remember ClearAppData2 and post-ping errors; sometimes pool problem, most
likely app crash, so do your local UITest with a release version of your app,
especially helped by `uitest_run.ps1`

TODO: "Tcp transport error" happened to me once during one of Azure DevOps service degradations.  I redid the pipeline, and the error was gone.

## Problems Accessing Stuff ##

You might have someone who can't access something, like build artifacts,
regardless of permissions (and rememeber there are permissions under project
settings, then permissions for pipelines, then permissions for EACH pipeline).
The problem might be their “access level”.  If their access level is
“Stakeholder”, then it probably needs to be changed to "Basic" or better.
“Basic”.  You can check anyone’s organization-specific access level at URLs
like this: `dev.azure.com/TheAppropriateOrganization/_settings/users`

# Thanks To Those Who Helped Me #

* James Montemagno, thanks for the huge amount of educational Xamarin content
  you've made, and specifically for the [Mobile App Tasks Azure DevOps
  plugin](https://marketplace.visualstudio.com/items?itemName=vs-publisher-473885.motz-mobile-buildtasks).
    * Online presence: [web site](https://montemagno.com/),
      [GitHub](https://github.com/jamesmontemagno),
      [Twitter](https://twitter.com/JamesMontemagno),
      [YouTube](https://www.youtube.com/channel/UCENTmbKaTphpWV2R2evVz2A),
      [Twitch](https://twitch.tv/jamesmontemagno),
      [Xamarin Show](https://channel9.msdn.com/Shows/XamarinShow))
      [DevBlog](https://devblogs.microsoft.com/xamarin/author/jamesmontemagno/).
* Andrew Hoefling, thanks for the "[Azure Pipelines Custom Build Numbers in YAML Templates](https://www.andrewhoefling.com/Blog/Post/azure-pipelines-custom-build-numbers-in-yaml-templates)"
  article and your contribution to the Mobile App Tasks Azure DevOps plugin, 
  your "[Azure Pipelines Custom Build Numbers in YAML
  Templates](https://www.andrewhoefling.com/Blog/Post/azure-pipelines-custom-build-numbers-in-yaml-templates)"
  blog post, and leading the [Rochester Xamarin Meetup group](https://www.meetup.com/Rochester-Xamarin-Meetup/). I look forward to reading more of your blog.
    * online presence: [web site](https://www.andrewhoefling.com/),
      [GitHub](https://github.com/ahoefling),
      [Twitter](https://twitter.com/andrew_hoefling).
* Dan Siegel, thanks for your [Prism](https://prismlibrary.com/) work, your
  educational content, personally helping me with Prism, and pointing me to a
  [UITest-in-pipeline example](https://github.com/PrismLibrary/Prism/tree/master/build) to work from.
    * online presence: [web site](https://dansiegel.net/),
      [LinkedIn](https://twitch.tv/dansiegel),
      [GitHub](https://github.com/dansiegel),
      [StackOverflow](https://stackoverflow.com/users/5699454/dan-s),
      [Twitter](https://twitter.com/DanJSiegel),
      [YouTube](https://www.youtube.com/dansiegel),
      [Twitch](https://twitch.tv/dansiegel).
* Jerome Laban thanks for making the UITest-in-pipeline example, and
  making/[explaining](https://twitter.com/jlaban/status/1338923127615741956)
  the additional example at UnoPlatform.
    * online presence: [web site](https://jaylee.org/), [GitHub](https://github.com/jeromelaban), [Twitter](https://www.twitter.com/jlaban).
* Brian Lagunas, thanks for giving me SO MUCH Prism help and your work on Prism. My coworkers and I 
