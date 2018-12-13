---
title: T4 Templates at Build Time with Dotnet Core
date: 2018-12-13 08:21:35
tags:
 - dotnetcore
 - T4
 - T5
 - csharp
 - build
 - csproj
---
Currently, T4 templates can only be run at build time as part of a `​dotnet msbuild`​ command. `Microsoft.TextTemplating.Targets` is only available on machines with Visual Studio installed, and can’t be resolved from a `​dotnet build`​ command.

The workaround is to use T5: <https://github.com/atifaziz/t5>

T5 is an open source implementation of the T4 processor designed to work with Mono. It happily works with dotnet core. And yes, the “4” in “T4” isn’t a version number (it stands for 4 T’s: Text Template Transformation Toolkit), so if it really bothers you just pretend that T5 stands for Text Template Transformation Toolkit Two.

To use it in a `​Microsoft.NET.Sdk`​ project, include the following:

```xml
<ItemGroup>
    <DotNetCliToolReference Include="T5.TextTransform.Tool" Version="1.1.0-*"/>
</ItemGroup>

<ItemGroup>
    <TextTemplate Include="**\*.Generated.tt"/>
</ItemGroup>

<Target Name="TextTemplateTransform" BeforeTargets="BeforeBuild">
    <ItemGroup>
        <Compile Remove="**\*.Generated.cs"/>
    </ItemGroup>
    <Exec WorkingDirectory="$(ProjectDir)" Command="dotnet tt %(TextTemplate.Identity)"/>
    <ItemGroup>
        <Compile Include="**\*.Generated.cs"/>
    </ItemGroup>
</Target>
```

There’s quite a bit going on in this short snippet, so I’ll break it down a bit more so we can see what’s happening.

The first item group adds T5 as a dotnet CLI tool, allowing us to later invoke it using `​dotnet tt`​.

```xml
<ItemGroup>
    <DotNetCliToolReference Include="T5.TextTransform.Tool" Version="1.1.0-*"/>
</ItemGroup>
```

Next, we have an item group which defines our `​TextTemplate`​ metadata for the build. It’s globbed to automatically include all files from any project subdirectory ending with `​.Generated.tt`​. I like to name all of my templates this way as it makes cleaning up before and after the transform step easier.

```xml
<ItemGroup>
    <TextTemplate Include="**\*.Generated.tt"/>
</ItemGroup>
```

Now we can define a build target to contain all of the tasks related to template transformation.

```xml
<Target Name="TextTemplateTransform" BeforeTargets="BeforeBuild">
```

The new SDK projects very kindly include all source files by default. Unfortunately, this means if we’ve already generated files in a previous build our task will throw a duplicate compile item error when we go to add them later on in this task. We absolutely have to add our generated files manually because the compile items are evaluated when build is invoked, so if there are no generated files in existance (or they’ve changed) the build fails.

One work around would be to disable auto inclusion, but I quite like the feature. So a much nicer solution is to start our target with a task item group that excludes all `​.Generated.cs`​ files from compile (if your generated output file has a different extension you can add globbed `​Compile`​ items for each) .

```xml
<ItemGroup>
    <Compile Remove="**\*.Generated.cs"/>
</ItemGroup>
```

Now we’re finally able to run our command to transform our template. The `​%(TextTemplate.Identity)`​ syntax tells the build system to run this task for every unique `​TextTemplate`​ item that was found by `​<TextTemplate Include="**\*.Generated.tt"/>`​. We also have to remember to set the working directory to the project level to find the T5 dotnet tool.

```xml
 <Exec WorkingDirectory="$(ProjectDir)" Command="dotnet tt %(TextTemplate.Identity)"/>
```

Finally, we need to add the files we just generated back into the upcoming compile task. We do this with a second item group.

```xml
<ItemGroup>
    <Compile Include="**\*.Generated.cs"/>
</ItemGroup>
```

With these changes, running `​dotnet build`​ at the solution level will succeed. Any new templates you create are auto included, and everything works as you think it should. Yay templates!