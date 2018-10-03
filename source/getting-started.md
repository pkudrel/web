---
_disableBreadcrumb: true
jr.disableMetadata: true
jr.disableLeftMenu: true
jr.disableRightMenu: true
uid: getting-started
title: Getting Started
---

# Getting Started

This article gives a tour through the most essential parts to know when working with NUKE.

## Introduction

NUKE follows the **principle of low ceremony**, which means that builds are executed with little to no requirements - which includes required knowledge as well as required tools. Since builds are written as simple C# console applications, they ensure **native integration with existing IDEs** like VisualStudio, JetBrains Rider or VSCode. Features like debugging, navigation and refactorings will just work out-of-the-box. Also NuGet packages can be consumed the usual way, by adding package references. There are simply no limitations compared to any other C# project. The most traditional way to execute a build is to invoke one of the **generated bootstrapping scripts** (_build.ps1_ or _build.sh_). More information can be found in the [bootstrapping](#bootstrapping) and [build invocation](#build-invocation) sections.

### Extended Tooling

Although NUKE integrates natively with existing tooling, we provide additional tools and extensions to offer the best user-experience for a build system. The global tool `Nuke.GlobalTool` helps with the initial setup and also allows for a more convenient:

```
# For global availability
dotnet tool install Nuke.GlobalTool --global

# For one-time availability
dotnet tool install Nuke.GlobalTool --tool-path ./nuke
```

Extensions can be installed for some of the most popular IDEs:

- [JetBrains ReSharper](https://resharper-plugins.jetbrains.com/packages/ReSharper.Nuke/)
- [JetBrains Rider](https://plugins.jetbrains.com/plugin/10803-nuke-support)
- [VisualStudio Code](https://marketplace.visualstudio.com/items?itemName=nuke.support)

Typically, these extensions offer a convenient way to execute build targets in different modes. For instance, either in run or debug mode, or with or without dependencies. Also they include code snippets to write targets more quickly.


## Build Setup

To setup our project with NUKE, we can execute the global tool:

```powershell
# For repositories without build
nuke

# For re-invocation
nuke :setup
```

During the setup, a wizard will prompt for the following questions:

- Which **solution** should be the default?
- How should the **bootstrapping** for the build project work?
- What target **framework** should be used?
- What project **format** should the build project use?
- Which NUKE **version** should be used?
- What should be the **location** for the build project?
- What should be the **name** for the build project?

It is strongly recommended to choose **.NET Core** as the platform. This solely indicates that the build project is built with .NET Core. The build will still be able to compile projects based on the full .NET Framework.

Additionally, when selecting a default solution, the wizard will continue with more specific questions to provide an even better default build implementation. For example:

- Use either _.NET Core SDK_ or _.NET Framework/Mono_ for compilation
- Are source files located in either `./src` or `./source`
- Are packages moved to either `./output` or `./artifacts`
- ... and many more smartness!

### Effective Changes

During execution, the following changes will be applied:

- Generate a _.nuke_ marker file in the root directory
- Generate a _build.ps1_ and _build.sh_ file in the current directory
- Copy templates for the build project and [build class](https://github.com/nuke-build/nuke/blob/develop/source/Nuke.GlobalTool/templates/Build.cs)
- If a solution was selected, the build project is added

_Note: for general awareness, we recommend to review applied changes using `git diff` or similar tools. The setup can also be re-invoked via `nuke :setup` to update the bootstrapping files to the latest version._

### Bootstrapping

The bootstrapping scripts _build.ps1_  and _build.sh_ take care of compiling and executing the build project. What is actually happening, depends on the choice of either .NET Core or .NET Framework/Mono as the platform.

For **.NET Core**, the script will perform the following steps:

1. Check _global.json_ for a [.NET Core tools version](https://docs.microsoft.com/en-us/dotnet/core/tools/global-json)
2. If the global installation of .NET Core matches the version specified in (1) or no version was specified, use the global installation
3. Otherwise, install local version to temp directory:
    1. If no version was specified, download the latest version
    2. If version did not match, download the expected version
4. Compile and execute the build project

For **.NET Framework/Mono**, the script will perform the following steps:

1. Install the specified NuGet executable version into the temp directory
2. If NuGet version is _latest_, try to update
3. Install and execute the _Nuke.MSBuildLocator_, which determines the MSBuild executable
4. Compile and execute the build project

## Build Invocation

In order to invoke NUKE, we can use the global tool or one of the bootstrapping scripts that corresponds to our operating-system:

```powershell
# Global Tool (executed from anywhere below .nuke file)
nuke [parameters]

# PowerShell
./build.ps1 [parameters]

# Bash
./build.sh [parameters]
```

_Note: the global tool is in fact just a wrapper for the script invocations._

### Argument Specification

Input arguments can simply be passed to the [build invocation](#build-invocation). Below, some of the predefined arguments are explained:

- `-target`: defines the target(s) to be executed; multiple targets are separated by plus sign (i.e., `compile+pack`); if no target is defined, the _default_ will be executed; can also be positional as first argument
- `-verbosity <value>`: supported values are `quiet`, `minimal`, `normal` and `verbose`
- `-skip [targets]`: if no target is defined, only the invoked targets are executed; multiple targets are separated by plus sign (i.e, `-skip clean+push`)
- `-graph`: will generate a HTML view of target dependencies
- `-help`: will show further information about available targets and parameters

For better understanding, here are a couple of examples:

```
# Execute a single target Pack
nuke pack

# Execute two targets Pack and Test
nuke pack+test

# Execute the default target and skip all dependent targets
nuke -skip

# Execute Pack but skip Clean and Compile
nuke -skip clean+compile -target pack

# Show help (other arguments are ignored for convenience)
nuke -target compile -help
```

NUKE also provides a convenient approach to [declare additional parameters](#parameter-declaration).

## Build Authoring

Builds are written as simple console applications. Targets are implemented as _expression-bodied properties_ and the default target is defined with the `Main` method.

```c#
class Build : NukeBuild
{
    public static int Main() => Execute<Build>(x => x.MyTarget);

    Target MyTarget => _ => _
        // other fluent calls
        .Executes(() =>
        {
            Console.WriteLine("Hello from NUKE!");
        });
}
```

The fluent syntax allows to set further target specific options:

- `Description("text")`: description shown in help text
- `DependsOn(SymbolTarget)`: symbolic dependency that is executed before this target
- `DependsOn("NamedTarget")`: named dependency that is - if existent - executed before this
- `Requires(() => Field)`: field that must have been initialized prior to build execution; usually used in combination with `[Parameter]` attribute
- `Requires(() => condition)`: condition that is checked prior to build execution
- `OnlyWhen(() => condition)`: condition that tells whether the target should be executed or skipped
- `WhenSkipped(DependencyBehavior.Execute/Skip)`: whether dependency should be executed or skipped if this target is skipped

### Predefined properties

The `NukeBuild` base class provides predefined properties:

- `RootDirectory`: directory where the `.nuke` file is located; usually the repository root
- `TemporaryDirectory`: temporary directory at `/.tmp`
- `Host`: build execution host (i.e., Console, Jenkins, TeamServices, TeamCity, ...)
- `IsLocalBuild`/`IsServerBuild`: flag that indicates whether the build is running locally (console host) or on a server (CI host)
- `SkippedTargets`: targets that are skipped via the `-skip` parameter
- `InvokedTargets`: targets that are directly invoked from command-line
- `ExecutingTargets`: targets that are part of the execution list

### CLT Wrappers

NUKE ships with a lot of wrapper APIs for command-line tools like _MSBuild_, _NuGet_, _xUnit.net_  or _OpenCover_. These wrapper APIs are generated from so-called [specification files](https://github.com/nuke-build/nuke/tree/develop/build/specifications), which can either be written manually, e.g. for smaller internal company tools, or converted from other sources like we do for the [Docker](https://github.com/nuke-build/docker/) and [Azure](https://github.com/nuke-build/azure/) addons. Generating code from specifications allows to provide a rich and consistent API with minimal effort.

A call like `msbuild.exe Solution.sln /nologo /targets:Restore;Build /p:configuration=Release /maxCpuCount:2` can easily be constructed using the `MSBuildTasks` with either fluent API or string interpolation. Whatever is preferred:

```csharp
// using static Nuke.Common.Tools.MSBuild.MSBuildTasks;

MSBuild(s => s
    .SetSolutionFile(SolutionFile)
    .EnableNoLogo()
    .SetTargets("Restore", "Build")
    .SetConfiguration("Release")
    .SetMaxCpuCount(2));
    
MSBuild($"{SolutionFile} /p:configuration={Configuration} ...");
```

Such generated task classes implement the following features:

- **Tool path resolution:** whenever possible, the tool path will be set automatically:
    - **Package references:** referencing a NuGet package in the build project, either via [`PackageReference`](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files) or [packages.config](https://docs.microsoft.com/en-us/nuget/reference/packages-config), allows to resolve the executable path from the installed package.
    - **PATH variable:** using the tools `where` (Unix) and `which` (Windows) the required executable is attempted to be resolved from the `PATH` environment variable.
    - **Environment variables:** for any task class, the environment variable `[TOOL]_EXE` can be used to provide the required executable. This takes precedence over other resolution methods.
- **Argument construction:** arguments that must be passed can be constructed using a fluent API. Every call creates a new instance, thus allows easy composition. The fluent API includes the following methods:
    - Setting and resetting an argument value (`SetValue`, `ResetValue`).
    - Enabling, disabling and toggling of boolean flags (`EnableFlag`, `DisableFlag`, `ToggleFlag`).
    - Setting, adding, removing and clearing of collection-like arguments (`SetItems`, `AddItems`, `RemoveItems`, `ClearItems`).
    - Adding and removing of dictionary and multi-dictionary arguments (`AddKeyValuePair`, `RemoveKey`)
- **Process invocation:** processes are invoked with the specified working directory, tool path, arguments and environment variables. The process is awaited to be exited; if specified, with a timeout. Afterwards, the exit code is asserted to be zero. 
- **Output capturing:** the process standard and error output is captured in a collection and returned from every task alias.
- **Documentation:** task aliases and arguments include the same xml summaries as described in the individual official tools documentation.

If necessary, extension points can be used to implement:

- **Return value conversion:** the process arguments and output collection are passed to a custom implemented method that creates a proper typed return value.
- **Log level detection:** for each output entry, a custom method is called to determine its log level (error, warning, info, trace.
- **Exit code validation:** the default implementation that asserts a zero exit code can be overriden.
- **Pre/post-processing:** methods are called before and after process invocation. The pre-processing also allows for modifications of the arguments. 

### Parameter Declaration

- Separators
- Lisp-casing
- Single/double dash
- Supported types
- Typo detection

## Common Build Steps

In the following sections, we will illustrate some of the most common build steps used in the .NET ecosystem. We will also explain how NUKE can help with _default settings_ to further standardize such targets.

### Clean, Restore, Compile

This example shows how to clean, restore and compile a solution while using [GitVersion](https://gitversion.readthedocs.io/) to calculate a version number based on our git commit history.

```csharp
[GitVersion] readonly GitVersion GitVersion;

Target Clean => _ => _
    .Executes(() =>
    {
        FileSystemTasks.DeleteDirectories(GlobDirectories(SourceDirectory, "**/bin", "**/obj"));
        FileSystemTasks.EnsureCleanDirectory(OutputDirectory);
    });

Target Restore => _ => _
    .DependsOn(Clean)
    .Executes(() =>
    {
        DotNetTasks.DotNetRestore(s => s
            .SetWorkingDirectory(SolutionDirectory)
            .SetProjectFile(SolutionFile));

        // Or using static imports and default settings:
        DotNetRestore(s => DefaultDotNetRestore);
    });

Target Compile => _ => _
    .DependsOn(Restore)
    .Executes(() =>
    {
        DotNetTasks.DotNetBuild(s => s
            .SetWorkingDirectory(SolutionDirectory)
            .SetProjectFile(SolutionFile)
            .EnableNoRestore()
            .SetConfiguration(Configuration)
            .SetAssemblyVersion(GitVersion.GetNormalizedAssemblyVersion())
            .SetFileVersion(GitVersion.GetNormalizedFileVersion())
            .SetInformationalVersion(GitVersion.InformationalVersion));

        // Or using static imports and default settings:
        DotNetBuild(s => DefaultDotNetBuild);
    });

```

### Test and Coverage

This example shows how to execute tests using [xUnit.net](https://xunit.github.io/) with optionally enabling [OpenCover](https://github.com/OpenCover/opencover) for code coverage reporting.

```csharp
Target TestAndCoverage => _ => _
    .DependsOn(Compile)
    .Executes(() =>
    {
        var assemblies = GlobFiles(SolutionDirectory, $"*/bin/{Configuration}/net4*/Nuke.*.Tests.dll").NotEmpty();
        var xunitSettings = new Xunit2Settings()
            .AddTargetAssemblies(assemblies)
            .AddResultReport(Xunit2ResultFormat.Xml, OutputDirectory / "tests.xml");

        if (EnvironmentInfo.IsWin)
        {
            var searchDirectories = xunitSettings.TargetAssemblyWithConfigs.Select(x => Path.GetDirectoryName(x.Key));

            OpenCoverTasks.OpenCover(s => s
                .SetOutput(OutputDirectory / "coverage.xml")
                .SetTargetSettings(xunitSettings)
                .SetSearchDirectories(searchDirectories)
                .SetWorkingDirectory(RootDirectory)
                .SetRegistration(RegistrationType.User)
                .SetTargetExitCodeOffset(targetExitCodeOffset: 0)
                .SetFilters(
                    "+[*]*",
                    "-[xunit.*]*",
                    "-[FluentAssertions.*]*")
                .SetExcludeByAttributes(
                    "*.Explicit*",
                    "*.Ignore*",
                    "System.Diagnostics.CodeAnalysis.ExcludeFromCodeCoverageAttribute")
                .SetExcludeByFile(
                    "*/*.Generated.cs",
                    "*/*.Designer.cs",
                    "*/*.g.cs",
                    "*/*.g.i.cs")

            // Or using static imports and default settings:
            OpenCover(s => DefaultOpenCover
                .SetOutput(OutputDirectory / "coverage.xml")
                .SetTargetSettings(xunitSettings)
                .SetSearchDirectories(searchDirectories));
        }
        else
            XunitTasks.Xunit2(s => xunitSettings);
    });
```

### Packaging and Publish

This example shows how to pack and publish build artifacts. Via the `NuGet` switch we can control whether the packages should be pushed to [nuget.org](https://nuget.org/) or [myget.org](https://myget.org/). In any case, we require the `ApiKey` parameter to be specified. When pushing to nuget.org, we also require the build to be executed with the `Release` configuration. For repositories hosted at [GitHub](https://github.com/), we can also include a link to the changelog file.

```csharp
[Parameter("ApiKey for the specified source.")] readonly string ApiKey;

[GitRepository] readonly GitRepository GitRepository;

string Source => NuGet
    ? "https://api.nuget.org/v3/index.json"
    : "https://www.myget.org/F/myfeed/api/v2/package";

string Branch => GitRepository.Branch;
string ChangelogFile => RootDirectory / "CHANGELOG.md";

Target Pack => _ => _
    .DependsOn(Compile)
    .Executes(() =>
    {
        var changelogUrl = GitRepository.GetGitHubBrowseUrl(ChangelogFile, branch: "master");

        DotNetTasks.DotNetPack(s => s
            .SetPackageReleaseNotes(changelogUrl)
            .SetWorkingDirectory(SolutionDirectory)
            .SetProject(SolutionFile)
            .EnableNoBuild()
            .SetConfiguration(Configuration)
            .EnableIncludeSymbols()
            .SetOutputDirectory(OutputDirectory)
            .SetVersion(GitVersion.NuGetVersionV2));

        // Or using default settings:
        DotNetPack(s => DefaultDotNetPack
            .SetPackageReleaseNotes(changelogUrl));
    });

Target Publish => _ => _
    .DependsOn(Pack)
    .Requires(() => ApiKey)
    .Requires(() => !NuGet || Configuration.EqualsOrdinalIgnoreCase("release"))
    .Executes(() =>
    {
        GlobFiles(OutputDirectory, "*.nupkg").NotEmpty()
            .Where(x => !x.EndsWith(".symbols.nupkg"))
            .ForEach(x => DotNetNuGetPush(s => s
                .SetTargetPath(x)
                .SetSource(Source)
                .SetApiKey(ApiKey)));
```

<br/>
<br/>
**Happy building!**
