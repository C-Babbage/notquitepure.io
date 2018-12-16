---
title: T4 Templates at Build Time with Dotnet Core
date: 2018-12-13 08:21:35
desc: Using T4 build time code generation templates with dotnet core cross platform
tags:
 - dotnetcore
 - dotnetcli
 - T4
 - T5
 - build
 - msbuild
 - templates
 - csproj
---
Currently, T4 templates can only be run at build time as part of a `​dotnet msbuild`​ command. `Microsoft.TextTemplating.Targets` is only available on machines with Visual Studio installed, and can’t be resolved from a `​dotnet build`​ command.

The workaround is to use T5: <https://github.com/atifaziz/t5>

T5 is an open source implementation of the T4 processor designed to work with Mono. It happily works with dotnet core. Though the “4” in “T4” isn’t a version number (it's shorthand for 4 T's), if it really bothers you just pretend that T5 stands for Text Template Transformation Toolkit Two.

To use it in a `​Microsoft.NET.Sdk`​ project file, include the following:

```xml
<ItemGroup>
    <DotNetCliToolReference Include="T5.TextTransform.Tool" Version="1.1.0-*"/>
    <TextTemplate Include="**\*.Generated.tt"/>
    <Generated Include="**\*.Generated.cs"/>
</ItemGroup>

<Target Name="TextTemplateTransform" BeforeTargets="BeforeBuild">
    <ItemGroup>
        <Compile Remove="**\*.cs"/>
    </ItemGroup>
    <Exec WorkingDirectory="$(ProjectDir)" Command="dotnet tt %(TextTemplate.Identity)"/>
    <ItemGroup>
        <Compile Include="**\*.cs"/>
    </ItemGroup>
</Target>

<Target Name="TextTemplateClean" AfterTargets="Clean">
    <Delete Files="@(Generated)" />
</Target>
```

There’s quite a bit going on in this short snippet, so I’ll break it down a bit more so we can see what’s happening.

The first item group does 3 things:

1. References the T5 nuget package as a dotnet CLI tool, allowing us to later invoke it using `​dotnet tt`​
2. Defines an item called `​TextTemplate`​ which includes all files ending with `​.Generated.tt`
3. Defines an item called `​Generated`​ which includes all the generated files matching `​*.Generated.cs`​ (if you’re generating other file types you can specify more `​Generated`​ items)

```xml
<ItemGroup>
    <DotNetCliToolReference Include="T5.TextTransform.Tool" Version="1.1.0-*"/>
    <TextTemplate Include="**\*.Generated.tt"/>
    <Generated Include="**\*.Generated.cs"/>
</ItemGroup>
```

Next, we define a build target to contain all of the tasks related to template transformation.

```xml
<Target Name="TextTemplateTransform" BeforeTargets="BeforeBuild">
```

The new SDK projects very kindly include all source files by default. Unfortunately, this means if we’ve already generated files in a previous build our task will throw a duplicate compile item error when we go to add the new ones. Further, if there are no generated files in existance (or they’ve changed) the build fails.

One work around would be to disable auto inclusion, but I quite like the feature. So a much nicer solution is to start our target with a task item group that excludes all `​**\*.cs`​ files from compile. We re-add these after we’ve transformed our templates.

```xml
<ItemGroup>
    <Compile Remove="**\*.cs"/>
</ItemGroup>
```

This next item does the heavy lifting. The `​%(TextTemplate.Identity)`​ syntax tells the build system to run this task for every unique `​TextTemplate`​ item that was found by `​<TextTemplate Include="**\*.Generated.tt"/>`​. This is called “batching" in msbuild nomenclature if you want to find more on the topic. We also have to remember to set the working directory to the project level to find the T5 dotnet tool.

```xml
 <Exec WorkingDirectory="$(ProjectDir)" Command="dotnet tt %(TextTemplate.Identity)"/>
```

The last item group for this target adds the files we removed (plus the newly generated files) back into the upcoming compile task.

```xml
<ItemGroup>
    <Compile Include="**\*.cs"/>
</ItemGroup>
```

It would be really nice to have the generated files deleted after running `​dotnet clean`​. One final target with a delete task should be enough. This is where we make use of the `​Generated`​ item we included at the start. You may have to tweak the globbing if you use a different naming strategy for generated files (personally I like the `​*.Generated.cs`​ style because it’s very explicit and makes including generated files in msbuild tasks trivial).

```xml
<Target Name="TextTemplateClean" AfterTargets="Clean">
    <Delete Files="@(Generated)" />
</Target>
```

With this configuration running `​dotnet build`​ at the solution level will succeed. Any new templates you create are auto included, cleaning the solution picks up generated code, and everything works as you think it should. Yay templates!

