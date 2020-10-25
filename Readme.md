# Project Architecture Guide

A guide for organizing .NET C# projects in different repositories that depend on each other

## Purpose

Many of my projects use common code, and rather than making multiple copies of the same source code, or using submodules, I've decided to organize them as separate projects. The idea is to consider each project, when it's consumed by another, as a third-party source.

This guide lists the information I have accumulated on how to do that in the context of GitHub and other connected Continuous Integration services.

## Creating the project

I'm using Visual Studio to create projects. First we create either a WPF App (.NET Core), or a WPF Library (.NET Core), depending on the type of project. Let's not place solution and project in the same folder.

[X] Use the solution Configuration Manager to create a new x64 platform for the project, and the solution.
[X] Delete Any CPU platforms for the project and solution.

Save the project and open the .csproj file with an editor.

[X] Replace `<TargetFramework`> with `<TargetFrameworks>` and set target framework(s) as needed by your project. For example `<TargetFrameworks>netstandard2.1;netcoreapp3.1;net48</TargetFrameworks>`.
[X] Add `<LangVersion>8.0</LangVersion>` and `<Nullable>enable</Nullable>` to use C# 8.0 and turn on support of nullable.
[X] Add `<GenerateDocumentationFile>true</GenerateDocumentationFile>` if appropriate.

Restart Visual Studio for the solution. Open the project properties.

[X] In the Application page, set the default namespace to what's appropriate.
[X] In the Package page set Version, Description, Authors, Copyright, Repository URL.
[X] Set the assembly version and file version to 1.0.0.1 to create tags for them.

Close the property page.

Go to Tools / Nuget Package Manager / Manage Nuget Packages for Solution...
Click the Browse tab and install:

+ Microsoft.CodeAnalysis.FxCopAnalyzers
+ Microsoft.CodeQuality.Analyzers
+ Microsoft.NetCore.Analyzers
+ StyleCop.Analyzers

Copy .editorconfig from an already created project in the root folder of the solution. Visual Studio asks if you want to make it a solution item. Answer Yes and save the solution.

In the solution context menu, select Create Git Repository, then in the Team Explorer page select Sync, then Publish to GitHub.

Commit and sync your project as necessary afterward.

## Adding helpers

My projects include several helpers that check or modify the final binaries.

### Version Builder

This tool runs at the begining of a build and will check all project files. If a file has changed, the last number of the assembly and assembly file versions is incremented.

To activate this tool:

+ Copy `updateversion.bat` from an already created project in the root folder of the solution
+ Copy the PreBuild from an already created project in the root folder of the solution and add this project to the solution. The reason calling the batch file is to do it once per build, not once per framework.
+ Make all projects in the solution to depend on PreBuild for the build order.

### Commit ID

All binaries are tagged with the repository and commit and ID they were build from. To do this:

+ Copy `updatecommit.bat` from an already created project in the root folder of the solution
+ Open the .csproj file with an editor and add or merge the following tags:

````xml
<Target Name="PostBuild" AfterTargets="PostBuildEvent" Condition="'$(SolutionDir)'!='*Undefined*'">
<Exec Command="if exist &quot;$(SolutionDir)updatecommit.bat&quot; call &quot;$(SolutionDir)updatecommit.bat&quot; &quot;$(SolutionDir)&quot; &quot;$(TargetPath)&quot;" />
</Target>
````

This will tag the final binary at the end of each build for each framework. The `Condition="'$(SolutionDir)'!='*Undefined*'"` condition is to prevent this during a dotnet restore (explicit or implicit).

### Digital signature

It is possible to sign binaries with a digital certificate. To do this:

+ Copy `signfile.bat` from an already created project in the root folder of the solution
+ Open the .csproj file with an editor and add or merge the following tags:

````xml
<Target Name="PostBuild" AfterTargets="PostBuildEvent" Condition="'$(SolutionDir)'!='*Undefined*'">
<Exec Command="if exist &quot;$(SolutionDir)signfile.bat&quot; call &quot;$(SolutionDir)signfile.bat&quot; &quot;$(SolutionDir)&quot; &quot;$(Configuration)-$(Platform)&quot; &quot;$(TargetPath)&quot;" Condition="'$(Configuration)|$(Platform)'=='Release|x64'" />
</Target>
````

## Adding dependency on other project

Say you want to include tools from the [Contracts](https://github.com/dlebansais/Contracts) project.

+ Copy `nuget.config` from an already created project in the root folder of the solution.
+ Reload Visual Studio if necessary.
+ Go to Tools / Nuget Package Manager / Manage Nuget Packages for Solution...
+ Select github as the Package source.
+ Select the `Contracts` package and install it.
+ Close Visual Studio and open the .csproj file with an editor. There should be a `<PackageReference Include="Contracts" Version="..." />` line in one of the <ItemGroup> tags.
+ Change the line as follow (keep the version number):

````xml
<PackageReference Include="Contracts-Debug" Version="1.0.8" Condition="'$(Configuration)|$(Platform)'=='Debug|x64'" />
<PackageReference Include="Contracts" Version="1.0.8" Condition="'$(Configuration)|$(Platform)'!='Debug|x64'" />
````

+ This will load the Contracts-Debug package in the debug version, and the Contracts package in the release version as well as during donet restore (explicit or implicit).
+ Close the editor and start Visual Studio again.

## Adding support for Continous Integration with appveyor

To enable Appveyor, copy `appveyor.yml` from an already created project in the root folder of the solution.
This file is organized as follow:

````yaml
before_build:
  - nuget restore %APPVEYOR_PROJECT_NAME%.sln
  - |-
    printf "<?xml version=\"1.0\" encoding=\"utf-8\"?>\n" > build_all.xml
    printf "  <Project xmlns=\"http://schemas.microsoft.com/developer/msbuild/2003\">\n" >> build_all.xml
    printf "    <Target Name=\"Build\">\n" >> build_all.xml
    printf "      <MSBuild Projects=\"%APPVEYOR_PROJECT_NAME%.sln\" Properties=\"Configuration=Debug;Platform=x64\"/>\n" >> build_all.xml
    printf "      <MSBuild Projects=\"%APPVEYOR_PROJECT_NAME%.sln\" Properties=\"Configuration=Release;Platform=x64\"/>\n" >> build_all.xml
    printf "    </Target>\n" >> build_all.xml
    printf "</Project>\n" >> build_all.xml
````

This will create `build_all.xml`, a script to build the `Debug` and `Release` configurations in the same job. The purpose is to deploy a single zip file with both configurations to GitHub Release. 

````yaml
  - nuget install Packager -DependencyVersion Highest -OutputDirectory packages
  - ps: $folder = Get-ChildItem -Path packages/Packager.* -Name | Out-String
  - ps: $firstline = ($folder -split '\r\n')[0]
  - ps: $fullpath = ".\packages\$firstline\lib\net48\Packager.exe"
````

This downloads and install the latest version of Packager, a program that generates a .nuspec file for each project in a solution.

````yaml
  - ps: '& $fullpath --debug'
  - nuget pack nuget-debug\%APPVEYOR_PROJECT_NAME%-Debug.nuspec
  - dotnet pack --configuration Release /p:Platform=x64 %APPVEYOR_PROJECT_NAME%/%APPVEYOR_PROJECT_NAME%.csproj
````

This creates a nuspec file for a package with the `-Debug` suffix, packages it and also packages the Release configuration using the more traditional approach.
  
````yaml
  - ps: |-
        $xml = [xml](Get-Content .\$env:APPVEYOR_PROJECT_NAME\$env:APPVEYOR_PROJECT_NAME.csproj)
  - ps: $version = $xml.Project.PropertyGroup.Version
  - ps: set version_tag v$version
  - ps: $env:VERSION_TAG=$version_tag
````

This fills `VERSION_TAG` with the version being compiled, and will be used during deployment.

````yaml
artifacts:
  - path: $(APPVEYOR_PROJECT_NAME)/bin/
    name: $(APPVEYOR_PROJECT_NAME)
  - path: $(APPVEYOR_PROJECT_NAME)-Debug.*.nupkg
    name: $(APPVEYOR_PROJECT_NAME)-Package-Debug
    type: NuGetPackage
  - path: $(APPVEYOR_PROJECT_NAME)/bin/x64/Release/$(APPVEYOR_PROJECT_NAME).*.nupkg
    name: $(APPVEYOR_PROJECT_NAME)-Package-Release
    type: NuGetPackage
````

This creates 3 artifacts, one for GitHub Release and two for GitHub packages. These are the packages that will be consummed in other projects.

## Including code and symbols in the debug package.

The debug package, as created by Packager, doesn't contain any file. You need to add them in the /lib subdirectory of the nuget-debug directory. This is done in a PostBuild section of the project:

+ Open the .csproj file with an editor.
+ Include or append the following lines in a PostBuild `Target` tag:

````xml
  <Target Name="PostBuild" AfterTargets="PostBuildEvent" Condition="'$(SolutionDir)'!='*Undefined*'">
    <Exec Command="if not exist &quot;$(SolutionDir)nuget-debug\lib\$(TargetFramework)&quot; mkdir &quot;$(SolutionDir)nuget-debug\lib\$(TargetFramework)&quot;" Condition="'$(Configuration)|$(Platform)'=='Debug|x64'" />
    <Exec Command="if exist &quot;$(TargetPath)&quot; copy &quot;$(TargetDir)*&quot; &quot;$(SolutionDir)nuget-debug\lib\$(TargetFramework)\&quot; &gt; nul" Condition="'$(Configuration)|$(Platform)'=='Debug|x64'" />
    <Exec Command="if not exist &quot;$(SolutionDir)nuget\lib\$(TargetFramework)&quot; mkdir &quot;$(SolutionDir)nuget\lib\$(TargetFramework)&quot;" Condition="'$(Configuration)|$(Platform)'=='Release|x64'" />
    <Exec Command="if exist &quot;$(TargetPath)&quot; copy &quot;$(TargetDir)*&quot; &quot;$(SolutionDir)nuget\lib\$(TargetFramework)\&quot; &gt; nul" Condition="'$(Configuration)|$(Platform)'=='Release|x64'" />
  </Target>
````

Because this creates new files at build, you need to exclude them in .gitignore:

	/nuget
	/nuget-debug
 

## Publishing your project for other projects to consume

Until deployment can be improved, it is done by merging the `master` branch and the `deployment` branch (`master` is always ahead). The deployment section in `appveyor.yml` is executed then, because all deployments are condtioned to the `deployment` branch being built.

To perform the merge, copy `deploy.bat` from an already created project in the root folder of the solution, and run it every time you want to publish a CI build.
 
Two packages are then published on GitHub Package:

+ The regular package using the name of the project, and containing Release configuration files (.dll assembly and doc if any).
+ The debug package, using the name of the project followed by the `-Debug` suffix, containing Debug configuration files (.dll assembly with `Assert` kept, .pdb symbols and doc if any.

