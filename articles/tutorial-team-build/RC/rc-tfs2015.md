<properties pageTitle="Ionic Tutorial" 
  description="This is an article on ionic tutorial" 
  services="" 
  documentationCenter=""
  authors="bursteg" />

#Using Tools for Apache Cordova with Team Foundation Services 2015 RC
**This article is specific to using Tools for Apache Cordova 2015 with TFS 2015 *RC*. See [this article for details on Visual Studio Online or TFS 2015 RTM or later](../tfs2015.md).**

Tools for Apache Cordova is designed to work with a number of different team build systems since the projects it creates are standard [Apache Cordova Command Line interface](http://go.microsoft.com/fwlink/?LinkID=533773) (CLI) projects. Team Foundation Services 2015 provides a new [cross-platform agent](http://go.microsoft.com/fwlink/?LinkID=533789) and [Gulp](http://go.microsoft.com/fwlink/?LinkID=533742lp) based build capabilities that enables TFS to build directly on Windows or OSX which is a critical capability Cordova based development. In addition, Gulp also enables you to easily add in a large number of "[plugins](http://go.microsoft.com/fwlink/?LinkID=533790)" to perform useful build tasks in environments that you do not control directly like Visual Studio Online.

For these reasons, this tutorial will focus on the use of the cross-platform agent and Gulp rather than MSBuild as the primary build language for Cordova apps. If you still need to use the legacy XAML / MSBuild based approach, see the [TFS 2013](http://go.microsoft.com/fwlink/?LinkID=533770) tutorial for details on setup. The instructions generally still apply to TFS 2015.

**Troubleshooting Tip: Be aware that a bug in VS templates in VS 2015 RC included four json files that can cause issues if added to source control: plugins/android.json, plugins/remote_ios.json, plugins/windows.json, and plugins/wp8.json.** Adding these files to source control can result in a build that appears to succeed but is missing plugin native code. They should only be included if the "platforms" folder is also checked in which is not recommended. Simply remove these files from source control to resolve the issue.

##Initial Setup
Since the build process we will describe here is not directly dependent on MSBuild or Visual Studio for Android, you have two options for installing pre-requisites on Windows:

1.  Install Visual Studio 2015 and select the Tools for Apache Cordova option and let it install the pre-requisites for you

2.  Manually install only the pre-requisites needed for the specific platforms you intend to build. For example, you do not need to install Visual Studio at all if you only intend to target Android. See "Installing Dependencies" in the [Building Cordova Apps in a Team / Continuous Integration Environment](http://go.microsoft.com/fwlink/?LinkID=533743) tutorial for details.

Next you will need to install the Windows build agent to build Android, Windows, or Windows Phone, and the [cross-platform build agent](http://go.microsoft.com/fwlink/?LinkID=533789) on an OSX machine if you intend to build iOS. See [TFS 2015 documentation](http://go.microsoft.com/fwlink/?LinkID=533772) for detailed instructions on configuring the agent for use with an on premise TFS 2015 instance or Visual Studio Online.

**Troubleshooting Tip:** See ["Internet Access & Proxy Setup" in the general CI tutorial](http://go.microsoft.com/fwlink/?LinkID=533743) if your build servers have limited Internet connectivity or require routing traffic through a proxy.

###Visual Studio Online
As of this writing, you can build Cordova apps targeting Android, Windows, and Windows Phone using the Hosted Agent Pool in Visual Studio Online. This allows you to build without setting up a Windows build agent on premise. iOS builds are not yet available.

When using the Hosted Agent Pool in Visual Studio Online (VSO), all pre-requisites will already be installed and Node.js will be in your path. However, Apache Ant, required for building Android with Cordova 4.3.0 and below, will not be in your path. Fortunately you can resolve this issue in a few simple steps. We will cover these details step-by-step later in this tutorial.

###Meet the Cross-Platform Build Agent
Since it is a new capability, let's pause and briefly highlight the new TFS [cross-platform build agent](http://go.microsoft.com/fwlink/?LinkID=533789) we will be using in this tutorial for building iOS on OSX since setup is different than traditional TFS build agents. The agent is a Node.js based service that uses a HTTPS connection to your TFS 2015 server to fetch work. As a result, your OSX machine only needs to have HTTP access to your TFS instance but not the other way around. This makes setup and configuration quite simple. The agent is for use with TFS 2015 and Visual Studio Online's [next generation build system](http://go.microsoft.com/fwlink/?LinkID=533772), not the legacy XAML/MSBuild based system.

The pre-requisites in this case are simple: Your OSX machine needs to have Node.js and Xcode installed. Simply open the OSX Terminal app and follow the [setup instructions](http://go.microsoft.com/fwlink/?LinkID=533789). The agent will automatically register itself with TFS when you start up the agent for the first time.

Because of its design, you can also easily use an **on-premise OSX machine with Visual Studio Online.** The OSX machine simply needs to have HTTP access to your VSO domain URI. You do not need a VPN connection and VSO does not need access to the OSX machine. Simply enter the your VSO project's domain URI when prompted during agent setup (Ex: "https://myvsodomain.visualstudio.com"). All other setup instructions apply directly.

##Environment Variables
You should set the following environment variables if they have not already been configured. Note that you can also set these in the "Variables" section of your build definition if you would prefer.

| **Variable**       | **Required For**                         | **Purpose**                              | **Default Location (Visual Studio 2015)** |
|:-------------------|:-----------------------------------------|:-----------------------------------------|:------------------------------------------|
| **ANDROID\_HOME**  | Android                                  | Location of the Android SDK              | %PROGRAMFILES(x86)%\\Android\\android-sdk |
|**JAVA\_HOME**     | Android                                  | Location of Java                         | %PROGRAMFILES(x86)%\\Java\\jdk1.7.0\_55 |
| **ANT\_HOME**      | Android when building using Ant (not Gradle) | Location of Ant                          | %PROGRAMFILES(x86)%\\Microsoft Visual Studio 14.0\\Apps\\apache-ant-1.9.3 |
| **GRADLE\_USER\_HOME**      | Optional | Overrides the default location Gradle build system dependencies should be installed when building Android using Cordova 5.0.0+ | If not specified, uses %HOME%\\.gradle on Windows or ~/.gradle on OSX |
| **CORDOVA\_CACHE** | Optional                                 | Overrides the default location used by the [sample build module](http://go.microsoft.com/fwlink/?LinkID=533736) to cache installs of multiple versions of Cordova. | If not specified, uses %APPDATA%\\cordova-cache on Windows and ~/.cordova-cache on OSX |

Note that you can opt to pre-populate the GRADLE_USER_HOME and CORDOVA_CACHE locations with the versions of Cordova and its dependencies you want to use by updating the [prep-cache PowerShell script](https://github.com/Chuxel/taco-team-build/tree/master/samples/prep-cache) in the samples directory of the sample build module GitHub repo.

###Setting Your Path
The following will also need to be in your path:
- **Node.js** should already be in your path on OSX simply by the fact that you've setup the cross-platform build agent, but if it is not in your path on Windows you will want to be sure it is configured for use. The default location of Node.js on Windows is **%PROGRAMFILES(x86)%\nodejs**.
- **%ANT_HOME%\bin** should be added to your path if you are using a version of Cordova < 5.0.0 or have specified the "--ant" build option

When using the Hosted Agent Pool in Visual Studio Online (VSO), all other pre-requisites will already be installed and these environment variables will be set. Node.js will be in your path but Apache Ant, required for building Android with Cordova 4.3.0 and below, will not be in your path. Fortunately you can resolve this issue in a few simple steps. We will cover these details in the "Build Definition for Windows" section below.

##Project Setup & Build Definitions
###Adding Gulp to Your Project
Using Gulp in a team environment is fairly straight forward as you can see in the detailed [Gulp tutorial](http://go.microsoft.com/fwlink/?LinkID=533742). However, to streamline setup, follow these steps:

1.  Take the sample "gulpfile.js" and "package.json" file from the "samples/gulp" folder [from this GitHub repo](http://go.microsoft.com/fwlink/?LinkID=533736) and place them in the root of your project

2.  Check these two files into source control with your project

From here you can modify gulpfile.js and add other gulp plugins. The [Gulp tutorial](http://go.microsoft.com/fwlink/?LinkID=533742) provides additional detail on what the gulpfile does and how to wire Gulp tasks as "hooks" into Cordova build events.

###Creating Your Build Definitions
We'll assume for the purposes of this tutorial that we want to build our Cordova app for Android, iOS, and Windows. The Windows Cordova platform can only be built on Windows and iOS can only be built on OSX. As a result, we'll need the ability to be able to queue a build that can target one of these two operating systems.

There are two ways that this can be accomplished:

1.  Setting up separate build queues for OSX vs Windows machines and then queueing the same build definition in the appropriate build queue based on the desired platform

2.  Using the concept of a "demand" in two separate build definitions to route the work to the correct OS from the same queue

For the sake of this tutorial, we'll cover option 2. The sample "gulpfile.js" assumes you want to build Android, Windows, and Windows Phone on Windows and iOS on OSX. Technically you could also opt to have Android built on OSX but we will not cover that in detail in this tutorial.

#### Build Definition for Windows
**Note:** *For this tutorial we have opted to use the "Command Line" build task since as of this writing the "Npm install" and "Gulp" build tasks were not available from the Windows agent. You can opt to use these build tasks instead and this tutorial will be updated as these features come on-line.*

Detailed instructions on creating build definitions in TFS 2015 can be found in [its documentation](http://go.microsoft.com/fwlink/?LinkID=533772), but here are the specific settings you will need to use to configure a build.

1.  Depending on the version of TFS 2015 you are using, you may need to click on the **BUILD.PREVIEW** menu option to access the next generation TFS build system.

	![BUILD.PREVIEW](media/rc-tfs2015/tfs2015-0.png)

2.  Create a new build definition and select "Empty" as the template. We'll start out targeting platforms that can be built on Windows so give the build definition a name that indicates that this is the case.

3.  Now we will configure the build definition to install any Gulp or npm package dependencies your build may have. 
	1.  Under the "Build" tab, add a new build step and select **Command Line** from the **Utility** category
	2.  Use the following settings:
		- **Tool:** cmd
		- **Arguments:** /c npm install
		- **Advanced =\> Working Directory**: Location of the Cordova project itself inside your solution (not the solution root).
		- **Advanced =\> Fail on Standard Error**: Unchecked.

	![Windows Build Definition - npm](media/rc-tfs2015/tfs2015-1.png)

	We need to use cmd.exe here due as npm is a batch file rather than an executable and thus is not found by the Command Line build task. This will be resolved in future updates.

4.  Next we'll configure Gulp itself. 

    1.  Under the "Build" tab, add a new build step and select **Command Line** from the **Utility** category.
	2.  Use the following settings:
		- **Tool:** node
		- **Arguments:** node_modules/gulp/bin/gulp.js
		- **Advanced =\> Working Directory**: Location of the Cordova project itself inside your solution (not the solution root).
		- **Advanced =\> Fail on Standard Error**: Unchecked.

	![Windows Build Definition - gulp](media/rc-tfs2015/tfs2015-2.png)

5.  Next we need to ensure that this particular build runs on Windows rather than OSX. Under the "General" tab, verify a demand that "Cmd" exists is present. If not, add one.

	![Windows Build Definition - Demand](media/rc-tfs2015/tfs2015-3.png)

6.  As an optional step, you can configure your build to upload the resulting build artifacts to your TFS or VSO instance for easy access. The sample gulpfile.js script places the resulting output in the "bin" folder to make configuration simple. **TFS 2015 RC on-premise installs vary here slightly.** 

    For **Visual Studio Online and post-RC TFS 2015** releases:

    1. Under the "Build" tab, add a new build step and select **Publish Artifact** from the **Build** category.
	2. Use the following settings:
		- **Copy Root**: Location of the Cordova project itself inside your solution (not the solution root).
		- **Contents:** bin/*
		- **Artifact Name:** bin
		- **Artifact Type:** Server

	![Windows Build Definition - Drop location](media/rc-tfs2015/tfs2015-4.png)

 	For **TFS 2015 RC** on-premise installs, go to the "Options" tab and enter the following information:
	- **Copy to Staging Folder:** Checked
    - **Copy to Staging Folder =\> Search Pattern:** \*/bin
    - **Create Build Drop:** Checked
    - **Create Build Drop =\> Drop location:** Server


Finally, click the "Queue build..." button to validate your setup. You'll see a real-time console view of your build progressing so you can quickly fine tune your definition.

That's it for Windows! You're now able to build using the Android, Windows, and Windows Phone 8 (WP8) Cordova platforms.

####Using the Hosted Agent Pool in Visual Studio Online
As of this writing, you can build Cordova apps targeting Android, Windows, and Windows Phone using the Hosted Agent Pool in Visual Studio Online. This allows you to build without setting up a Windows build agent on-premise. iOS builds are not yet available. 

When using Hosted build agents in Visual Studio Online (VSO), Node.js will already be in your path but Ant will not. Fortunately you can resolve this issue in a few simple steps as all of the other required environment variables will be set for you.

1. Create a batch file called "setenv.cmd" and place it in a solution folder with the following contents:

	~~~~~~~~~~~~~~~~~~~~~~~~~~
	@SET PATH=%PATH%;%ANT_HOME%\bin
	~~~~~~~~~~~~~~~~~~~~~~~~~~

2. Check this into source control with the rest of your project.

	![setenv.cmd in solution](media/rc-tfs2015/tfs2015-8.png)

3. Next, under the "Build" tab in your Windows Build Definition, add a new build step, select **Batch Script** from the **Utility** category, and enter the following settings:
	
	- **Path:** setenv.cmd
	- **Modify Environment:** Checked.

4. Make this the first build step in your build definition.

	![Windows Build Definition - setenv.cmd](media/rc-tfs2015/tfs2015-7.png)

You are now ready to go in VSO!

#### Build Definition for OSX
Now let's create a version of this same build definition to target iOS that will run on a configured cross-platform agent on OSX.

1. Right click on the Windows build definition and select "Clone." Once you save you should give this definition a name that indicates it's the OSX build.

2. Next we need to update our call to "npm install" to be OSX friendly.
	1.  Select the existing "Run cmd" Command Line build step
	2.  Update the following settings:
		- **Tool:** npm
		- **Arguments:** install

   ![OSX Build Definition - Update npm install](media/rc-tfs2015/tfs2015-6.png)

3.  Now we need to add a demand that will route builds to OSX machines rather than Windows. Under the "General" tab, remove the "Cmd" demand if present and add a demand that "xcode" exists.

	![OSX Build Definition - Demand](media/rc-tfs2015/tfs2015-5.png)

You are now all set! You can configure either of these build definitions further as you see fit including having them automatically fire off on check-in or adding other validations.

**Troubleshooting Tip:** See ["Troubleshooting Tips for Building on OSX" in the general CI tutorial](http://go.microsoft.com/fwlink/?LinkID=533743) for tips on resolving common build errors that can occur when building Cordova projects on that operating system.

## More Information
* [Learn about other Team Build / CI options](../tutorial-team-build-readme.md)
* [Read tutorials and learn about tips, tricks, and known issues](../../cordova-docs-readme.md)
* [Download samples from our Cordova Samples repository](http://github.com/Microsoft/cordova-samples)
* [Follow us on Twitter](https://twitter.com/VSCordovaTools)
* [Visit our site http://aka.ms/cordova](http://aka.ms/cordova)
* [Read MSDN docs on using Visual Studio Tools for Apache Cordova](http://go.microsoft.com/fwlink/?LinkID=533794)
* [Ask for help on StackOverflow](http://stackoverflow.com/questions/tagged/visual-studio-cordova)
* [Email us your questions](mailto:/vscordovatools@microsoft.com)