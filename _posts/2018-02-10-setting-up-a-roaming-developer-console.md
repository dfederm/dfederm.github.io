---
layout: post
title: Setting up a Roaming Developer Console
categories: [Developer Environment]
tags: [autorun, cmd, command prompt, console, developer command prompt, developer console, onedrive, powershell]
comments: true
---
Have you ever wanted to have your personal scripts and aliases just always available to you in any console session? Well, it's possible!

In my setup I'll be using OneDrive, but the concept applies to any cloud storage that syncs to your disk like Google Drive or Dropbox.

## Setting it up
The Command Prompt has a fairly unknown feature called Autorun, which allows for running a command every time `cmd.exe` starts. This is done via a registry key, but to make setup easy, you can write a script and put it in your cloud storage. You can take this a step further can have that Autorun script be in your cloud storage as well. This is the basis by which we'll be creating the roaming dev console.

First, create a `setup.cmd` in your cloud storage. This script will need to be run once per machine you log into. In my example, I'm putting it under my OneDrive folder under `Code\Scripts\setup.cmd`.

```bat
@echo off

setlocal

REM The script to run each time cmd starts.
set initScript=%HOMEDRIVE%%HOMEPATH%\OneDrive\Code\Scripts\init.cmd

if not exist "%initScript%" (
    echo "%initScript%" does not exist!
    exit /B 1
)

REM Configure the current user's autorun
echo reg add "HKCU\Software\Microsoft\Command Processor" /v "Autorun" /d "\"%initScript%\"" /t REG_EXPAND_SZ /f
call reg add "HKCU\Software\Microsoft\Command Processor" /v "Autorun" /d "\"%initScript%\"" /t REG_EXPAND_SZ /f

echo Autorun configured!

REM init now to avoid the need to restart the console
echo Running "%initScript%" now
"%initScript%"

endlocal
```

This setup script simply adds the registry key which controls Command Prompt's Autorun and points it to an init script under your cloud storage.

For those who prefer PowerShell, there is an equivalent concept to Autorun called Profiles. Instead of setting the registry key, add a PowerShell file at `%UserProfile%\My Documents\WindowsPowerShell\profile.ps1`. This will run initially in any PowerShell window you open. To make the roaming part work, have this `profile.ps1` file call an `init.ps1` under your cloud storage.

You can actually take this even further and have your setup script also configure other settings (user-wide git config), install various applications (git, nodejs, etc.), and anything else you may want to do once per computer you use.

## The init script
Now that init script is configured to run in each Command Prompt or PowerShell window, what're the kinds of things to add here?

One warning about this is that Autorun/Profile runs before **_every_** invocation of `cmd` or `powershell`. Thus it's not recommended to do long-running work there or perhaps more importantly, don't emit any output. Any application which may spawn child processing using `cmd` or `powershell` will incur the cost of your init script, and if it parses the output of the process it launched, it will see any output from your init script. So just keep it quiet, short, and sweet.

Here's an example of what part of my `init.cmd` file looks like. In my example, I'm putting it under my OneDrive folder under `Code\Scripts\init.cmd`.

```bat
@echo off

SET PATH=%PATH%;%~dp0

DOSKEY ..=cd ..
DOSKEY n="%ProgramFiles(x86)%/Notepad++/notepad++.exe" $*
```

Again, for PowerShell users, you can just as easily put similar things in your `init.ps1`, like global functions.

While this doesn't look like much, it sets up a few aliases for me, and adds my `Code\Scripts` OneDrive folder to my `PATH`. I have other convenience scripts
and tools which don't require an install under this path as well:

![Example tools](/assets/devconsole-scripts-path-300x223.png)

To give an idea of some of the things I have in mine:
*   [`handle.exe`](https://docs.microsoft.com/en-us/sysinternals/downloads/handle){:target="_blank"} - Helps identify which stubborn process has an open handle on a file.
*   [`kill.exe`](https://docs.microsoft.com/en-us/sysinternals/downloads/pskill){:target="_blank"} - Renamed from `PsKill.exe`. Easily kills processes
*   [`nuget.cmd`](https://www.nuget.org/downloads){:target="_blank"} - Calls `Nuget\nuget.exe`, the NuGet CLI
*   [`procexp.exe`](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer){:target="_blank"} - A better task manager.
*   [`procmon.exe`](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon){:target="_blank"} - Shows real-time file and registry accesses, process spawns, etc.

The beauty about this is that if you add a new tool or alias, your cloud storage will automatically sync to other machines and so will just be available automatically there!
