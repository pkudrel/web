---
_disableBreadcrumb: true
_disableAffix: true
jr.disableMetadata: false
jr.disableLeftMenu: false
jr.disableRightMenu: true
uid: authoring-builds-parameter-declaration
title: Parameter Declaration
---

# Parameter Declaration

Build automation often incorporates dynamic arguments, which 




One common practice would be to define a `Configuration` parameter that returns `Debug` for local builds and `Release` for CI builds, but which is still configurable via `--configuration`: 
 
```c#
[Parameter]
readonly string Configuration { get; } = IsLocalBuild ? "Debug" : "Release";
```

Incooperation with Requires
