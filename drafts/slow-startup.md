


# The Problem

The software I am building has really slow startup times. Specifically when I start Visual Studio to run the website but also right after an Azure App Service deploy.

After deploy, the very first page load is extremely slow. In addition, the first page load of each page (excluding dynamic things like /foo/1 vs /foo/2) is very slow.

My hunches are:

- It's JIT costs
- It's loading assemblies into memory
- It's compiling the Razor page (not Razor Pages)

One solution here is to deploy to a Staging slot for Azure App Service then write something that hits every page once. This works but it's a pain to maintain.

I am trying to dig a bit deeper to see what's going on.

# Profilers

We can use a few different performance profilers:

- Visual Studio built in one
- ANTS
- dotTrace
- PerfView

Item 1 doesn't do the trick, I tried it. At least not how I had it setup.

Options 2 and 3 are maybe the easiest but they are also expensive to purchase (around $400/yr). Maybe from a time ROI perspective though that does make sense. It depends how much time I waste on PerfView!

Specifically, I want to trace from the very start of the Application, not just a specific page load. Then we can see both the initial statup issue and the per page issue, hopefully.

# Hunches

There are some pre-compilation commands we might be able to use to make this better but I want to get data first and make sure we are solving the right problem.

# Setup PerfView (maybe move this to another post?)

(Yes this is probably suboptimal. I am using .NET Framework instead of .NET Core. which has other tools that are easier to use. Upgrading is not an option right now.)

You can download PerfView from [here](https://github.com/microsoft/perfview/releases).

1. Run PerfView as an Administrator
1. Click `Collect data machine wide` from the main PerfView page
1. Click `Start Collection`

If you see the following:

ASP.NET provider activated but ASP.NET Tracing not installed.
No ASP.NET events will be created.
To fix: DISM /online /Enable-Feature /FeatureName:IIS-HttpTracing
See `ASP.NET events` in help for more details.

Go to an Administrator Comppand Prompt and run:
`DISM /online /Enable-Feature /FeatureName:IIS-HttpTracing`

Then stop and start another PerfView Collection.

After you may see this error:

```
The search locations for symbolic information has not been set up.
You have two choices.
•	Use the Microsoft symbol server. This will allow you to see the symbolic names for code developed by Microsoft, including the operating system DLLs as well as the .NET framework. To do this PerfView try to contact the Microsoft symbol server on the web when 'Lookup Symbols' command are executed to retrieve the necessary symbolic infom This is the recommended option.
•	Use the Empty Symbol Path. Choosing this option will mean that 'Lookup Symbols' commands will fail on any Microsoft code. However this does insure that PerfView does not issue network operations during symbol lookup.
```

TODO: Can we get the PerfView split out per page request? For now I will just request the initial page then stop it.

Maybe need this:

```
By default PerfView chooses a set of events that does not generate too much data but it useful for a variety of investigations. However wall clock investigations require events that are too voluminous to collect by default. Thus if you wish to do a wall clock investigation, you need to set the 'Thread Time' checkbox in the collection dialog.
```

And here:
https://htmlpreview.github.io/?https://github.com/Microsoft/perfview/blob/main/src/PerfView/SupportFiles/UsersGuide.htm
Under 'Starting a CPU Analysis'

In the folder PerfView is installed, open `PerfViewData.etl`.

TODO: What are `PerfViewData.clrRundown.etl` and `PerfViewData.kernel.etl`?

(Show as list, say which one to open)

Can't find the process. I think what's better is `Run a command`

How do we run `IIS Express` and a locally built website? It must be the same command that Visual Studio runs. What is it?

We need to find it using `%PROGRAMFILES%\IIS Express` in Explorer. For me that goes to `C:\Program Files\IIS Express` and inside there is `iisexpress.exe`.

Also the path to my web app is: `C:\azuredevops\SchoolSync\BlackbaudIntegration`

So we can try the following to test the command:

`C:\Program Files\IIS Express>iisexpress /path:C:\azuredevops\SchoolSync\BlackbaudIntegration`

And we can test it by going to `http://localhost:8080/`

It loads but there are issues because of SSL. How do we set that up for IIS Express?

https://learn.microsoft.com/en-us/answers/questions/1330161/iis-express-ssl-certificate-bindings-are-empty?cid=kerryherger

And then maybe after we can run PerfView from the CLI, which is ideal:

PerfView.exe "/DataFile:PerfViewData.etl" /BufferSizeMB:256 /StackCompression /CircularMB:1000 /KernelEvents:ThreadTime /Providers:"IIS: Active Server Pages (ASP)::Verbose,Microsoft-Windows-IIS" /NoGui collect​

https://learn.microsoft.com/en-us/shows/perfview-tutorial/tutorial-14-investigating-wall-clock-responce-time-in-aspnet-scenarios

# ANTS

Tried this. Downloaded app. Tried to get VS Ectension. I don't think one exists for VS 2022...

https://forum.red-gate.com/discussion/88790/windows-11-and-visual-studio-2022-support

And I can't find one in Marketplace. Crazy.

# dotTrace

Installed the app and VS 2022 plugin. Let's see.

https://www.jetbrains.com/help/profiler/2024.1/Starting_Local_Profiling_Session_VS.html#project

-- Message #0
-----------------------------------------------------------------

Unable to start profiling: One or more ETW providers failed to start. Please contact dotTrace support.
Details: Unable to start ETW provider

-----------------------------------------------------------------

INSTEAD

Go to the menu and do Tracing. Maybe this is wrong. At least it ran. Timeline didnt.

Get Snapshot and Wait (or let it end). This then opens dotTrace with a snapshot.

Looks like GetCryptographyClient() is taking a ton of time, is that the issue everywhere?

Thread #13 - 30% from GetCryptographyClient(). 
- Rest from this thread is in microsoft.applicationinsights.extensibility.perfcountercollector.quickpulse.quickpulsetelemetrymodule very slow

Thread #17 - Serilog.Sinks.Batch.BatchProvider slow
Serilog.Sinks.Batch.BatchProvider.timerpump slow why?

- Main Thread, I think we mostly care about this?
azure.security.keys.keyclient.getkeys very slow
Can we not use this? We need it to securely store keys... right? Maybe we can use a different mechanism for key storage in Azure App Service?







TODO: Load another page for the first time and the perf, after loading main page (can this be shown separately?)



























1. 
1. 
1. 
1. 
1. 
1. 
1. 
1. 
1. 
1. 







# Profiling























