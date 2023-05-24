---
layout: post
title: Scripting Machine Setup
categories: [Developer Environment]
tags: [developer command prompt, developer console, powershell]
comments: true
---

Lately, I've found myself setting up multiple computers, and with [Microsoft DevBox](https://aka.ms/devbox){:target="_blank"} on the horizon, I anticipate working with "fresh" machines more frequently. Like many developers, I thrive in a familiar environment with my preferred tools and settings, as muscle memory kicks in and I can efficiently tackle any task. Unfortunately, the process of setting up a new machine can be quite cumbersome. To address this challenge, I took matters into my own hands and developed a script that streamlines the entire setup process for me.

Note that I previously wrote about a [roaming developer console]({% post_url 2018-02-10-setting-up-a-roaming-developer-console %}), but it was not as robust as I needed, and a lot has changed since then, for example the release of `winget`.

You can find my completed script which I use for my personal setup on [GitHub](https://github.com/dfederm/MachineSetup){:target="_blank"}. I'd recommend forking it and tuning it for your own personal preferences.

## Requirements

A key requirement for this project, especially since I expected to iterate on it quite a bit at first, was to ensure the script was idempotent. The goal was to run and re-run the script multiple times while consistently achieving the desired machine state. This flexibility allowed me to make changes and easily apply them. As a result, I could even schedule the script to automatically incorporate any modifications I had made.

To enhance user-friendliness, I aimed for the script to skip unnecessary actions. Instead of blindly setting a registry key, I designed it to first check if the key was already in the desired state. This approach served two purposes: it provided valuable logging information to indicate what the script actually changed and avoided unnecessary elevation prompts.

Furthermore, I prioritized security and implemented a strategy to handle elevation. The script was designed to run unelevated by default, and only specific commands would require elevation if necessary. This adherence to the [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege){:target="_blank"} improved security measures and mitigated potential issues related to file creation as an administrator. Admittedly, this has debatable value since this script is personal and so should be deemed trustworthy before executing it.

Overall, these considerations, including idempotence, skipping unnecessary actions, and managing elevation, played crucial roles in making the script more robust, user-friendly, and secure.

## Defining Machine-Specific Paths

To ensure compatibility with different machine setups, the script begins by defining two crucial paths that might vary based on the drive topography of each machine. For example, on my personal machine I have a separate OS drive and data drive, while on my work machine I have a single drive. Specifically, these paths are the `CodeDir` and `BinDir`.

`CodeDir` represents the root directory for all code, where I typically clone git repositories and store project files. `BinDir` is the designated location for scripts and standalone tools.

The setup script initiates a prompt to determine the locations of `CodeDir` and `BinDir`, assuming they haven't been defined previously. Once the user provides the necessary input, the script proceeds to set these paths as user-wide environment variables. Additionally, `BinDir` is added to the user-wide `PATH`, ensuring convenient access to scripts and tools from anywhere within the system.

## Configuring Windows

Configuring Windows revolves around making modifications to the registry. The setup script encompasses several essential registry tweaks and configuration adjustments, including:

* Configuring cmd Autorun to run `%BinDir%\init.cmd` (more on that later)
* Showing file extensions in Explorer
* Showing hidden files and directories in Explorer
* Restore classic context menu
* Disabling Edge tabs showing in Alt+Tab
* Enable Developer Mode
* Enable Remote Desktop
* Enable Long Paths
* Opting out of Windows Telemetry
* Excluding `CodeDir` from Defender

I will certainly be adding more to this list as time goes on.

## Uninstalling Bloatware

When it comes to debloating scripts and tools, it's important to strike a balance. I find that many available scripts tend to be overly aggressive, removing applications that might actually be useful or causing unintended harm to the system. In my personal experience, I find it unnecessary to uninstall essential applications like the Edge browser or OneDrive. Additionally, it's worth noting that Microsoft [discourages](https://support.microsoft.com/en-us/topic/microsoft-support-policy-for-the-use-of-registry-cleaning-utilities-0485f4df-9520-3691-2461-7b0fd54e8b3a){:target="_blank"} the use of registry cleaners due to potential malware risks, and honestly orphaned registry keys take up virtually no disk space and don't slow the system down in any way.

Nevertheless, I do believe there is value in uninstalling a few specific applications that come bundled with Windows. These include:

* Cortana: Personally I don't find Cortana useful.
* Bing Weather: It's not my preferred method for checking the weather
* Get Help: I haven't found this app useful.
* Get Started: I haven't found this app useful.
* Mixed Reality Portal: I don't use virtual reality experienced on my desktop computer (or at all for that matter).

Beyond that, a clean install of Windows should be relatively free of bloatware applications.

## Installing Applications

Many applications these days can be installed and updated via [winget](https://github.com/microsoft/winget-cli){:target="_blank"}. Winget can easily be scripted to install a list of applications, and for me that list includes:

* 7zip: A versatile file compression tool.
* DiffMerge: For file or folder comparisons.
* Git: For version control
* HWiNFO: For monitoring CPU/CPU temperatures and clock speeds
* ILSpy: A decompiler for .NET assemblies.
* Microsoft Teams: For work
* MSBuild Structured Log Viewer: A tool for [debugging MSBuild]({% post_url 2020-12-13-debugging-msbuild %}).
* .NET 7 SDK: For developing with .NET
* Node.js: For developing with JavaScript
* Notepad++: One of my favored text editors
* NuGet: Package manager for .NET
* NuGet Package Explorer: A UI for inspecting NuGet packages
* PowerShell: Better than Windows Powershell
* PowerToys: Various useful utilities
* Remote Desktop Client: modern version of `mstsc`
* Regex Hero: Helpful for working with regular expressions
* SQL Server Management Studio: For working with SQL databases
* Sysinternals Suite: Various useful utilities
* Telegram: Favored communication app
* Visual Studio Code: One of my favored text/code editors
* Visual Studio 2022 Enterprise: Code editor
* Visual Studio 2022 Enterprise Preview: Daily driver code editor
* WinDirStat: For viewing disk usage
* Windows Terminal: Better than the stock one

While most applications can be installed via Winget, there are a few exceptions. In those cases, the script takes care of installing those applications separately. One such app is the Azure Artifacts Credential Provider (for Azure Feeds), and WSL. Note that installing WSL involves enabling some Windows Components which require a reboot to fully finish installing.

## Configuring Applications

Once the applications are installed, the setup script proceeds to configure them. Some applications are configured by the registry while other use environment variables and some even use configuration files. The following configurations are performed by the script:

* Setting git config and aliases
* Enable WAM integration for Git: Promptless auth for Azure Repos
* Force NuGet to use auth dialogs: Avoid device code auth for Azure Feeds in favor of a browser popup window
* Configure the NuGet cache locations: The defaults are under the user profile but I find a path under the `CodeDir` to be more appropriate.
* Opting out of VS Telemetry: Prioritizing privacy
* Opting out of .NET Telemetry: Prioritizing privacy
* Copying Windows Terminal settings (more on this later)

## Bootstrapping

A keen eye may have noticed that the setup script installs Git, but the script lives on GitHub, so there is a bootstrapping problem. How can we download the script and other assets from GitHub?

Luckily it's fairly easy to download an entire GitHub repository as a zip file. The following PowerShell will download the zip, extract it, and run it:

```ps
$TempDir = "$env:TEMP\MachineSetup"
Remove-Item $TempDir -Recurse -Force -ErrorAction SilentlyContinue
New-Item -Path $TempDir -ItemType Directory > $null
$ZipPath = "$TempDir\bundle.zip"
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri https://github.com/dfederm/MachineSetup/archive/refs/heads/main.zip -OutFile $ZipPath
$ProgressPreference = 'Continue'
Expand-Archive -LiteralPath $ZipPath -DestinationPath $TempDir
$SetupScript = (Get-ChildItem -Path $TempDir -Filter setup.ps1 -Recurse).FullName
& $SetupScript @args
Remove-Item $TempDir -Recurse -Force
```

That's a bit much to copy and paste though, so I saved that as a `bootstrap.ps1` script in the repo, so the full bootstrapping is a one-liner:

```ps
iex "& { $(iwr https://raw.githubusercontent.com/dfederm/MachineSetup/main/bootstrap.ps1) }" | Out-Null
```

It's a bit roundabout but the one-liner will download and execute `bootstrap.ps1`, which will in turn download the entire repo as a zip file, extract it, and run the setup script.

## `BinDir` and autorun

With the bootstrap process in place, we can finally complete the picture with the aforementioned `init.cmd` autorun script and the `BinDir`. The repo contains a `bin` directory which is copied to `BinDir` and contains the `init.cmd` autorun and other necessary scripts or files.

The `init.cmd` autorun is described in more detail in my [previous blog post]({% post_url 2018-02-10-setting-up-a-roaming-developer-console %}), but essentially it's a script that runs every time `cmd` is launched. I use it primarily to set up `DOSKEY` macros like `n` for launching Nodepad++. Note that if you prefer PowerShell, you can set up similar behavior using Profiles (`%UserProfile%\Documents\PowerShell\Profile.ps1`).

Additionally, the reason why `BinDir` is on the `PATH` is because any other helpful scripts can be added there and be invoked anywhere.

Finally, a backup of my Windows Terminal `settings.json` is in this directory so that the setup script can simply copy it to configure Windows Terminal.

## Conclusion

Setting up a new machine doesn't have to be a cumbersome process. By adopting this setup script and following the outlined steps, you can significantly reduce the time and effort required to configure new machines while ensuring a consistent and optimized working environment. With the power of automation and the flexibility of customization, the setup script presented in this blog post offers a practical solution to streamline the machine setup experience. Embrace the script, tailor it to your preferences, and let it handle the heavy lifting for you, allowing you to focus on what matters most—writing code and building remarkable software.