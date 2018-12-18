---
title: Aliases to Resolve Multiple Type Errors for Transitive Nuget Dependencies
date: 2018-12-18 22:18:03
desc: Extern alias for transitive nuget dependencies using msbuild targets
tags:
 - dotnet
 - nuget
 - extern
 - alias
 - build
 - msbuild
 - multiple
 - type
 - error
 - assembly
 - namespace
---

Duplicate type exceptions are the result of the linker finding the same fully qualified (with namespace) typename in two different strong named assemblies. This most often occurs as a result of a combination of nuget packages and their dependencies. In the case where you have a direct reference to one of the assemblies involved, you can set an alias in the properties menu via the solution explorer -\> references.

However, in the case where both assemblies are transitive dependencies of nuget packages the properties menu doesn’t function. This seems to be a result of using the new `​Microsoft.Net.SDK`​ project format.

Fortunately, there is an excellent workaround that was originally suggested by @gertjvr here: https://github.com/NuGet/Home/issues/4989\#issuecomment-310565840

The workaround relies on running a build target just before references are resolved. The reference path is conditionally matched against the assembly strong name you want to alias. If the reference is a match, your alias(es) are applied. Note that this will override any existing aliases if you use this method to apply aliases to assemblies that you’ve already managed to configure via the UI.

```xml
<Target Name="ApplyAlias" BeforeTargets="FindReferenceAssembliesForReferences;ResolveReferences">
    <ItemGroup>
        <ReferencePath Condition="'%(FileName)' == '{Assembly.Strong.Name.Goes.Here}'">
            <Aliases>alias1;alias2;alias3</Aliases>
        </ReferencePath>
    </ItemGroup>
</Target>
```

I found this extremely useful in solving a problem I had with creating a shim to drop a slow dependency out from a project. Other packages still referenced this library as a transitive dependency giving me this headache. The above fix allowed me to carry on with my original plan - which largely involved wrapping `​Microsoft.Extensions.DependencyInjection`​ with a minimal subset of `​Ninject`​’s API. I hope to cover the reasons for this shim in subsequent post.