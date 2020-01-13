---
layout: post
title: Classes, Nested Modules, and Sharing Values
---

- [Summary](#summary)
- [Goals](#goals)
- [Gross oversimplification summary of PowerShell features](#gross-oversimplification-summary-of-powershell-features)
  - [Sharing data](#sharing-data)
    - [Scope](#scope)
  - [Classes](#classes)
- [Exploiting Scopes](#exploiting-scopes)
  - [Basic implementation](#basic-implementation)
  - [Coded Properties](#coded-properties)
  - [Using your Class](#using-your-class)
- [Example Code](#example-code)

## Summary ##

As I've been working on a recent Module package, I've been separating out the module into nested module files. The main purpose was to keep the functions and commandlets organized based on their purpose or goals.  Eventually, I'd like to separate them out a bit more into interdependent, child modules similar to the Az module with Azure.

```powershell
# MyModule.psd1
@{
  NestedModules = @( 'Alpha.psm1', 'Beta.psm1' )
  FunctionsToExport = 'Get-Alpha','Set-Alpha',
    'Get-Beta','Set-Beta'
}
```

This means that the **MyModule** module has two file modules incorporated with it the two nested module **Alpha** and **Beta**.  This works great and no problems here.

## Goals ##

My real goal that I wanted to have was this:

- Consistent value across all nested modules
- Complete obfuscation outside of the module
- Ability to control input and output of data flows
- Black box visibility between modules
- Ease of use between modules
- Work within PowerShell framework

## Gross oversimplification summary of PowerShell features ##

> I should probably go into each of these in more detail each, but I'll hopefully come back to them in later posts and just retcon this article.
>
> **-A. Fool, last words**

### Sharing data ###

However, assume that there is a value in Alpha that needs to be shared with Beta.  There are a few ways to deal with this.

| **Method** | **Code** | **Pro** | **Con** |
| --- | --- | --- | --- |
| Global private variable | ```New-Variable -Name &lt;name&gt; -Value &lt;value&gt; -Visibility Private -Scope Global``` | Its privately available within your module | Its existence outside your module is visible with the proper call, even if value is hard to find |
| Get-/Set- Functions | ```Function Get-&lt;name&gt; { $script:&lt;name&gt; }Function Set-&lt;name&gt; {$script:&lt;name&gt; = $args[0] }``` | Value manipulation totally controlled by calls | - Functions exposed outside module- Have to call them potentially every time a value is updated- Very taxing when dealing with external data sources |
| Reference Object | ```[ref]$&lt;name&gt;``` | Gets dynamic content from single memory space | Have to constantly use .Value property and always use [ref] in type definition |
| Custom class | ```class MyClass {     $&lt;name&gt;     Function MyClass() { }}``` | Consistent structure throughout usage | Values unique to each instantiation. |

With all that, Global Private variables or Get-/Set- functions would probably do most of it, but I still didn't like the potential exposure and implementation.  I had no real silver bullet, so I started tinkering around some.

#### Scope ####

For anyone not working with PowerShell before, there are really only 4 scopes:

| **Scope** | **Purpose** |
| --- | --- |
| Global | available across all scopes |
| Local | only available within your local scope (ie: function, script, session, etc.) |
| Script | available within your script(file) |
| Using | Referencing current scope when calling a script block or other area. |

None of these do exactly what I'm after either.

### Classes ###

Classes are useful because you can setup your own structure:

```powershell
Class cSharing {
    [string] $MyValue

    cSettings () {
    }

}
```

Classes really have two main parts: Properties and Methods. 

While I could create a variable then in both Alpha and Beta of the type cSharing, each would have its own unique .MyValue as each is just its own instantiation of the type of structure.  Also, would have to publish the class making it visibly externally, again, not really looking to do that in this case.  There are some classes I would want public, but internal settings isn't really one of them.

## Exploiting Scopes ##

If you've gotten through the fast pace remedial, now this is where we start using scopes to our advantage.  Quite frankly, I stumbled on this.  It makes sense and I'm assuming someone has done this before, but I wanted to make sure it was shared. Its not an exploit of a vulnerability, just using the features provided to achieve our goals.

Going back to Classes, if we make a Method we use commands to do whatever we want: (consider this all within a ```class { }``` definition)

```powershell
[string] GetData () {
    $FunVariable
}
[void] SetData () {
    $FunVariable = $args[0]
}
```

This is useful, but $FunVariable would just be scoped to within each Method and really no use.  With a class, we can reference our current instantiation of the class with the variable $this and reference its properties, even private ones:

```powershell
[private][string]$FunVariable
[string] GetData () {
    $this.FunVariable
}
[void] SetData () {
    $this.FunVariable = $args[0]
}
```

This is helpful, but each instance of the class is going to have its own value of FunVariable.

Now, consider the scope Script:.  This means any value within the script.  A fun thing about this is while the values within a class are scoped to that instance of the class and that instance is within the scope of where it was created, the class itself has its own scope of where it was defined.  In this case, I had put it in its own file Classes.psm1.

If you move the variable outside of the class, it is now in the scope of the Script and you can reference it with $script:FunVariable .

```powershell
[string]$FunVariable
Class cSharing {
    [string]$MyVariable
    [string] GetData () {
        $script:FunVariable
    }
    [void] SetData () {
        $script:FunVariable = $args[0]
    }
}
```

What's great, is because now that the $FunVariable is in the scope of the Classes.psm1, no matter how many instances of ```cSharing``` you create they all reference the same ```$script:FunVariable``` object.

### Basic implementation ###

As an example, if we make our Alpha.psm1 and Beta.psm1 nested modules look like this:

```powershell
$AlphaSettings = New-Object cSettings

Function Get-Alpha {
    Write-Host "`tMyValue: $($AlphaSettings.MyValue)"
    Write-Host "`tShared: $($AlphaSettings.GetData())"
}

Function Set-Alpha {
    param($Value)

    $AlphaSettings.SetData($Value)
    $AlphaSettings.MyValue = $Value
}
```

Then do the same thing for Beta but replace '```Beta```' wherever you find '```Alpha```'.

Now if we import that module (Import-Module MyModule), we can run a sequence of commands and see the values change through the different scopes:

```powershell
PS> Get-Alpha
        MyValue: 
        Shared: 
PS> Get-Beta
        MyValue: 
        Shared: 
PS> Set-Alpha -Value "Set by alpha"
PS> Get-Alpha
        MyValue: Set by alpha
        Shared: Set by alpha
PS> Get-Beta
        MyValue: 
        Shared: Set by alpha
PS> Set-Beta -Value "Set in Beta"
PS> Get-Alpha
        MyValue: Set by alpha
        Shared: Set in Beta
PS> Get-Beta
        MyValue: Set in Beta
        Shared: Set in Beta
```

This illustrates that the Shared value changes with each setting even across modules without global exposure or private filtering.

However, things are a little less than ideal.  Methods work for reading and setting values, and they're a little clunky.  Furthermore, the class still has to be exported across module.

### Coded Properties ###

In many object oriented programs, classes can properties have code behind them for Get and Set routines.  However, PowerShell's class definition uses variables for properties and script blocks for methods; you can't do script blocks with properties.

Enter our PowerShell cmdlet friend: Add-Member ScriptProperty.  Not really sure why you can't do this in a class definition since PowerShell obviously supports this with the ScriptProperty member type.  With the Add-Member call the Value parameter is the Get script block and Value2 is the Set script block.

But, since we can't do this within the definition of the class structure, we have to run Add-Member against each object as its constructed. Luckily, PowerShell classes do have a constructor method you can overload and is called whenever an object is created.

```powershell
[string]$FunVariable
Class cSharing {
    [string]$MyVariable
    cSharing () {
        $this | Add-Member ScriptProperty Data { $script:FunVariable } { $script:FunVariable = $args[0]
    }
}
```

Now we just update our code to use the property Data instead of the methods GetData() and SetData().  In our example:

```powershell
Function Get-Alpha {
    Write-Host "`tMyValue: $($AlphaSettings.MyValue)"
    Write-Host "`tShared: $($AlphaSettings.Data)"
}

Function Set-Alpha {
    param($Value)

    $AlphaSettings.Data = $Value
    $AlphaSettings.MyValue = $Value
}
```

This looks much more familiar to PowerShell code and not quite so foreign.  Remove our module, re-import to get the update, and run our code to make sure it still works:

```powershell
PS> Get-Alpha
        MyValue: 
...
PS> Set-Alpha -Value "Set by alpha"
PS> Get-Alpha
        MyValue: Set by alpha
        Shared: Set by alpha
PS> Get-Beta
        MyValue: 
        Shared: Set by alpha
PS> Set-Beta -Value "Set in Beta"
PS> Get-Alpha
        MyValue: Set by alpha
        Shared: Set in Beta
PS> Get-Beta
        MyValue: Set in Beta
        Shared: Set in Beta
```

Perfect!

### Using your Class ###

The final piece I need is getting access to the Class definition and its script without publishing the Class to everyone and their bug.  This shouldn't be consider a major security consideration as they can still open your PowerShell and figure out what its doing, but rather just a way to minimize accidental access and protect the user from themselves.

In PowerShell v5, the command ```using``` was introduced to allow the importing of a module to obtain access to its Classes.  So rather than exporting the class and making it public, we can just use using:

```powershell
using module "C:\Path\To\Modules\GlobalSettings\Class.psm1"
```

This works fine as long as I know where the module path is.  However, that path can easily change depending on where  the module is installed, but we can fix that by using the $PSScriptRoot.

```powershell
using module "$PSScriptRoot\Class.psm1"
```

That fails because ```using``` will not accept a variable in the path.  Now we have to get a little creative, but we can do this by turning the string into a script block and then use dot sourcing to process it.

```powershell
$useBlock = [ScriptBlock]::Create("using module '$PSScriptRoot\Classes.psm1'")
. $useBlock
```

Now the module loads the class from the Classes.psm1!

This means that we're now sharing select object between modules without having to constantly pull and update the object every time we want to use it or make a change to it.  At the same time, we can coexist with unique properties among each iteration.

## Example Code ##

Feel free to exam the example code I made around this and use it as you like.  Also, feel free to reach out with any ideas, or comments around this! Just keep it civil here on the interwebs.

[GlobalSettings example module on GitHub](https://github.com/smallfoxx/Tools/tree/master/TestModule/GlobalSettings)

- [GlobalSettings.psd1](https://github.com/smallfoxx/Tools/raw/master/TestModule/GlobalSettings/GlobalSettings.psd1)
- [Alpha.psm1](https://github.com/smallfoxx/Tools/raw/master/TestModule/GlobalSettings/Alpha.psm1)
- [Beta.psm1](https://github.com/smallfoxx/Tools/raw/master/TestModule/GlobalSettings/Beta.psm1)
- [Classes.psm1](https://github.com/smallfoxx/Tools/raw/master/TestModule/GlobalSettings/Classes.psm1)