---
layout: post
title: Dynamic share params
---

This was my travel down finding a reliable way to duplicate parameter sets over 100 functions and using default values & aliases along the way.  If you don't want the details, you can jump down to the [TL;DR](#tldr) script for details.

# Quick intro

In case you couldn't tell, I am not a prolific writer; it's only been 2 years since my last post. However, I was recently playing around with an API wrap for PowerShell and put a few things together that I thought might be useful & thus will share.

Just as preface, a lot of what I'm doing could be done in C# or other languages possibly easier, but this particular API has been wrapped in lots of different languages and I'm trying to do as much as I can within PowerShell. However, if someone has tips or tricks of better ways, I'm **always** happy to hear them.

# What we're doing

So the adventure begins with the fact that I'm building over 100 cmdlets in PowerShell and they are intended to be ways for someone to call an API endpoint with a native PowerShell command and not have to go lookup all the API details.  Part of that ends up being there are VERY common parameters used over and over again in these (ie: profile, ID, etc.).  I was getting around this by just using a template function and VSCode autocomplete to paste it in.  They look something like this:

```powershell
function Get-MyResource {
    [cmdletbinding()]
    param(
        [parameter(ValueFromPipelineByPropertyName)]
        [string]$MyProfile=(Get-MyDefaultProfile),
        [parameter(ValueFromPipelineByPropertyName,ValueFromPipeline)]
        [Alias('ResourceID')]
        [string[]]$MyID
    )

    Process {
        Write-Debug "Using $MyProfile to get $($MyID -join ',')"
        If ($MyID) {
            Get-APIEndpoint -Endpoint "Resource" -Profile $MyProfile -APIParam @{ "Ids" = $MyID -join ','}
        } else {
            Get-APIEndpoint -Endpoint "Resource" -Profile $MyProfile
        }
    }
}
```

# Validation leads to Dynamic Param

However, I realized that I really wanted to do validation on one of these params after I created a lot of these. On top of that, I realized I wanted it to be dynamically validated based on available profiles. Therefore, I started doing building Dynamic Param to use and would go back and update them.

For those that aren't familiar with Dynamic Params, I'll link better articles later, but basically it allows you to have parameters based on dynamic values in your environment, even based on what has been typed on the command line for other parameters.  You can do farily simple ones, but a generic one I reuse commonly is this general form:

```powershell
DynamicParam {
    $RuntimeParamDic  = New-Object System.Management.Automation.RuntimeDefinedParameterDictionary
    $StandardProps = @('Mandatory','ValueFromPipeline','ValueFromPipelineByPropertyName','ParameterSetName','Position')
    $Attrib = [ordered]@{
        'ParamName' = @{
            'AttribType' = [string]
            'Mandatory' = $true
            'ValueFromPipeline' = $true
            'ValueFromPipelineByPropertyName' = $true
            'ParameterSetName' = 'ThisParamName'
            'Position' = 1
            'ValidSet' = (ValidFunctionSet)
        }
    }
    
    ForEach ($AttribName in $Attrib.Keys) {
        #[string]$AttribName = $Key.ToString()
        $ThisAttrib = New-Object System.Management.Automation.ParameterAttribute
        ForEach ($Prop in $StandardProps) {
            If ($null -ne $Attrib.$AttribName.$Prop) {
                $ThisAttrib.$Prop = $Attrib.$AttribName.$Prop
            }
        }
        $ThisCollection = New-Object  System.Collections.ObjectModel.Collection[System.Attribute]
        $ThisCollection.Add($ThisAttrib)

        If ($Attrib.$AttribName.ValidSet) {
            $ThisValidation = New-Object  System.Management.Automation.ValidateSetAttribute($Attrib.$AttribName.ValidSet)
            $ThisCollection.Add($ThisValidation)
        }

        $ThisRuntimeParam  = New-Object System.Management.Automation.RuntimeDefinedParameter($AttribName,  $Attrib.$AttribName.AttribType, $ThisCollection)
        $RuntimeParamDic.Add($AttribName,  $ThisRuntimeParam)
    }

    return  $RuntimeParamDic
    
}
```

The key part of this setup is this hashtable:

```powershell
$Attrib = [ordered]@{
    'ParamName' = @{
        'AttribType' = [string]
        'Mandatory' = $true
        'ValueFromPipeline' = $true
        'ValueFromPipelineByPropertyName' = $true
        'ParameterSetName' = 'ThisParamName'
        'Position' = 1
        'ValidSet' = (ValidFunctionSet)
    }
}
```

You can use this ordered hashtable to do multiple different parameters and the following code iterates through the table and adds the appropriate object structures to the `$RuntimeParamDic`, and then returns it.

The other piece in this set is the `ValidFunctionSet` which just returns the valid array of values the parameter, but can dynamically calculated too.

The function where you place the `DynamicParam` you can then reference the value from `$PSBoundParameters.ParamName`.  So the implementation of the `DynamicParam` in that very basic funciton would look something like this:

```powershell
function Get-MyResource {
    [cmdletbinding()]
    param(
        [parameter(ValueFromPipelineByPropertyName,ValueFromPipeline)]
        [Alias('ResourceID')]
        [string[]]$MyID
    )

    DynamicParam {
        $RuntimeParamDic  = New-Object System.Management.Automation.RuntimeDefinedParameterDictionary
        $StandardProps = @('Mandatory','ValueFromPipeline','ValueFromPipelineByPropertyName','ParameterSetName','Position')
        $Attrib = [ordered]@{
            'MyProfile' = @{
                'AttribType' = [string]
                'Mandatory' = $false
                'ValueFromPipelineByPropertyName' = $true
                'Position' = 1
                'ValidSet' = (Get-MyProfile)
            }
        }
        
        ForEach ($AttribName in $Attrib.Keys) {
            #[string]$AttribName = $Key.ToString()
            $ThisAttrib = New-Object System.Management.Automation.ParameterAttribute
            ForEach ($Prop in $StandardProps) {
                If ($null -ne $Attrib.$AttribName.$Prop) {
                    $ThisAttrib.$Prop = $Attrib.$AttribName.$Prop
                }
            }
            $ThisCollection = New-Object  System.Collections.ObjectModel.Collection[System.Attribute]
            $ThisCollection.Add($ThisAttrib)

            If ($Attrib.$AttribName.ValidSet) {
                $ThisValidation = New-Object  System.Management.Automation.ValidateSetAttribute($Attrib.$AttribName.ValidSet)
                $ThisCollection.Add($ThisValidation)
            }

            $ThisRuntimeParam  = New-Object System.Management.Automation.RuntimeDefinedParameter($AttribName,  $Attrib.$AttribName.AttribType, $ThisCollection)
            $RuntimeParamDic.Add($AttribName,  $ThisRuntimeParam)
        }

        return  $RuntimeParamDic
      
    }

    Process {
        $MyProfile = $PSBoundParameters.MyProfile
        Write-Debug "Using $MyProfile to get $($MyID -join ',')"
        If ($MyID) {
            Get-APIEndpoint -Endpoint "Resource" -Profile $MyProfile -APIParam @{ "Ids" = $MyID -join ','}
        } else {
            Get-APIEndpoint -Endpoint "Resource" -Profile $MyProfile
        }
    }
}
```

# Compartimentalizing the code with functions

Hopefully you see that this is a whole lot of code in a little function. And I'm looking at repeating this over 100 times.  Time to find some code reuse.  There are things within the `DynamicParam{}` that I could put into a function, but if you notice, `DynamicParam` itself uses braces `{` `}`... because its a code block.  Its a code block I'm repeating everywhere that just returns a dictionary object. So lets just encapsulate the whole thing in a function that. Actually, lets break out the dictionary build from the main function and just pass the dictionary build a table.

```powershell
function New-MyDynamicParam {
    param([System.Collections.Specialized.OrderedDictionary]$ParamDic,
        [string[]]$StandardAttribs = @('Mandatory','ValueFromPipeline','ValueFromPipelineByPropertyName','ParameterSetName','Position'))

    Begin {
        $RuntimeParamDic  = New-Object System.Management.Automation.RuntimeDefinedParameterDictionary
    }
    Process {
        ForEach ($ParamName in $ParamDic.Keys) {
            #[string]$AttribName = $Key.ToString()
            $ThisAttrib = New-Object System.Management.Automation.ParameterAttribute
            ForEach ($Prop in $StandardAttribs) {
                If ($null -ne $ParamDic.$ParamName.$Prop) {
                    $ThisAttrib.$Prop = $ParamDic.$ParamName.$Prop
                }
            }
            $ThisCollection = New-Object  System.Collections.ObjectModel.Collection[System.Attribute]
            $ThisCollection.Add($ThisAttrib)

            If ($ParamDic.$ParamName.ValidSet) {
                $ThisValidation = New-Object  System.Management.Automation.ValidateSetAttribute($ParamDic.$ParamName.ValidSet)
                $ThisCollection.Add($ThisValidation)
            }

            $ThisRuntimeParam  = New-Object System.Management.Automation.RuntimeDefinedParameter($ParamName,  $ParamDic.$ParamName.AttribType, $ThisCollection)
            $RuntimeParamDic.Add($ParamName,  $ThisRuntimeParam)
        }
    }
    End {
        return  $RuntimeParamDic
    }
}
function MyCommonParams {

    $Attribs = [ordered]@{
        'MyProfile' = @{
            'AttribType' = [string]
            'Mandatory' = $false
            'ValueFromPipelineByPropertyName' = $true
            'Position' = 1
            'ValidSet' = (Get-MyProfile)
        }
    }
    
    New-MyDynamicParam -ParamDic $Attribs
}
```

I realize I can also put the `ID` parameter into the dynamic set as well and then I can get rid of it from all the functions:

```powershell
function MyCommonParams {

    $Attribs = [ordered]@{
        'MyProfile' = @{
            'AttribType' = [string]
            'Mandatory' = $false
            'ValueFromPipelineByPropertyName' = $true
            'Position' = 0
            'ValidSet' = (Get-MyProfile)
        }
        'MyID' = @{
            'AttribType' = [string[]]
            'Mandatory' = $false
            'ValueFromPipeline' = $true
            'ValueFromPipelineByPropertyName' = $true
            'Position' = 1
        }
    }
    
    New-MyDynamicParam -ParamDic $Attribs
}

```

This leaves the template to look like this:

```powershell
function Get-MyResource {
    [cmdletbinding()]
    param()

    DynamicParam{
        MyCommonParams
    }

    Process {
        $MyProfile = $PSBoundParameters.MyProfile
        $MyID = $PSBoundParameters.MyID
        Write-Debug "Using $MyProfile to get $($MyID -join ',')"
        If ($MyID) {
            Get-APIEndpoint -Endpoint "Resource" -Profile $MyProfile -APIParam @{ "Ids" = $MyID -join ','}
        } else {
            Get-APIEndpoint -Endpoint "Resource" -Profile $MyProfile
        }
    }
}

```

# Missing a few things

This is working..... but, missing a few things. On the original layout, I had a default value for `$MyProfile` (`[string]$MyProfile=(Get-MyDefaultProfile)`, and I had an Alias (`[Alias('ResourceID')]`) for the `$MyID`.  However, the dynamic param doesn't have that yet.

Fear not!! For there is a way!!!

## Adding in an Alias

For the alias, its not too bad. We can use the `System.Management.Automation.AliasAttribute` object type to add an array as an alias.  So first we add an `Alias` property to the parameter table.

```powershell
'MyID' = @{
    'AttribType' = [string[]]
    'Mandatory' = $false
    'ValueFromPipeline' = $true
    'ValueFromPipelineByPropertyName' = $true
    'Position' = 1
    'Alias' = @('ids')
}
```

Also, some resources use a unique name for their type of resource. Since we have a function, we can pass the resource type to `MyCommonParams` as `$IDType` and add it to the array.

```powershell
$Attribs.MyID.'Alias' += "$($IDType)ID"
```

We also need to add a bit to the `New-MyDynamicParam` to process and add the alias.

```powershell
if ($ParamDic.$ParamName.Alias) {
    $ThisAlias = New-Object -Type `
        System.Management.Automation.AliasAttribute -ArgumentList @($ParamDic.$ParamName.Alias)
    $ThisCollection.Add($ThisAlias)
}
```

## Default value has a bit of a twist

I tried looking around for a default value for a dynamic parameter, but I couldn't find one documented one. If someone out there happens to have one, I would love to hear, so time to start exploring the object values out there.  Looking through them, I found the dictionary of the dynamic parameters was just this:

```powershell
Key       Value
---       -----
MyProfile System.Management.Automation.RuntimeDefinedParameter
MyID      System.Management.Automation.RuntimeDefinedParameter
```

If I look at the value associated with `MyProfile`, you'll see this:

```powershell
Name          : MyProfile
ParameterType : System.String
Value         :
IsSet         : False
Attributes    : {__AllParameterSets, System.Management.Automation.ValidateSetAttribute}
```

That `Value` property is very appealing!  So I setup the `New-MyDynamicParam` to look for a `DefaultValue` in the hashttable and setting the `Value` if it exists and adding it to the hashtable in `MyCommonParams` to have a default value.

```powershell
If ($ParamDic.$ParamName.DefaultValue) {
    $ThisRuntimeParam.Value = $ParamDic.$ParamName.DefaultValue
}
```

Put all that in, call the function without parameters so it will use default values, and......

Nothing.

No default values.

Doing some debugging and find that the problem is `$PSBoundParameter.MyProfile` has no value..... because there is no bound paramter because there was no parameter put on the command line.  K, so how do I get the value of the dynamic value and use it if the bound parameter is missing?

Luckily we have a function that returns a dictionary of the dynamic parameters: `MyCommonParams`.  We can iterate through the keys of the dictionary and for any values that are empty, grab the value and assign it the value.

```powershell
Begin {
    $CommParams = MyCommonParams
}
Process {
    ForEach ($Comm in ($CommParams.Keys)) {
        Set-Variable -Name $Comm -Value $PSBoundParameters.$Comm
        If (-not [string]::IsNullOrEmpty((Get-Variable -Name $Comm))) {
            Set-Variable -Name $Comm -Value $CommParams.$Comm.Value
        }
    }
    Write-Debug "Using $MyProfile to get $($MyID -join ',')"
    # more code here ...
}    
```

By doing a `ForEach` loop through the dictionary keys, I can also use this regardless of which variables are put in the common dynamica param table and reference them as the name

# Links

- [Dynamic Parameters in PowerShell](https://powershellmagazine.com/2014/05/29/dynamic-parameters-in-powershell/) by Ben Ten
- [Tips and Tricks to Using PowerShell Dynamic Parameters](https://jeffbrown.tech/tips-and-tricks-to-using-powershell-dynamic-parameters/) by Jeff Brown
- [RuntimeDefinedParameter Class](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.runtimedefinedparameter?view=powershellsdk-7.0.0)

# TL;DR

So just a quick summary, if you want to use Dynamic Parameters with Aliases and/or Default Values, you can use this script as a template:

```powershell
function New-MyDynamicParam {
    param([System.Collections.Specialized.OrderedDictionary]$ParamDic,
        [string[]]$StandardAttribs = @('Mandatory','ValueFromPipeline','ValueFromPipelineByPropertyName','ParameterSetName','Position'))

    Begin {
        $RuntimeParamDic  = New-Object System.Management.Automation.RuntimeDefinedParameterDictionary
    }
    Process {
        ForEach ($ParamName in $ParamDic.Keys) {
            #[string]$AttribName = $Key.ToString()
            $ThisAttrib = New-Object System.Management.Automation.ParameterAttribute
            ForEach ($Prop in $StandardAttribs) {
                If ($null -ne $ParamDic.$ParamName.$Prop) {
                    $ThisAttrib.$Prop = $ParamDic.$ParamName.$Prop
                }
            }
            $ThisCollection = New-Object  System.Collections.ObjectModel.Collection[System.Attribute]
            $ThisCollection.Add($ThisAttrib)

            If ($ParamDic.$ParamName.ValidSet) {
                $ThisValidation = New-Object  System.Management.Automation.ValidateSetAttribute($ParamDic.$ParamName.ValidSet)
                $ThisCollection.Add($ThisValidation)
            }

            if ($ParamDic.$ParamName.Alias) {
                $ThisAlias = New-Object -Type `
                    System.Management.Automation.AliasAttribute -ArgumentList @($ParamDic.$ParamName.Alias)
                $ThisCollection.Add($ThisAlias)
            }

            $ThisRuntimeParam  = New-Object System.Management.Automation.RuntimeDefinedParameter($ParamName,  $ParamDic.$ParamName.AttribType, $ThisCollection)
            If ($ParamDic.$ParamName.DefaultValue) {
                $ThisRuntimeParam.Value = $ParamDic.$ParamName.DefaultValue
            }
            $RuntimeParamDic.Add($ParamName,  $ThisRuntimeParam)
        }
    }
    End {
        return  $RuntimeParamDic
    }
}
function MyCommonParams {
    param([string]$IDType)

    $Attribs = [ordered]@{
        'MyProfile' = @{
            'AttribType' = [string]
            'Mandatory' = $false
            'ValueFromPipelineByPropertyName' = $true
            'Position' = 0
            'DefaultValue' = (Get-MyDefaultProfile)
            'ValidSet' = (Get-MyProfile)
        }
        'MyID' = @{
            'AttribType' = [string[]]
            'Mandatory' = $false
            'ValueFromPipeline' = $true
            'ValueFromPipelineByPropertyName' = $true
            'Position' = 1
            'Alias' = @('ids')
        }
    }
    
    If ($IDType) {
        $Attribs.MyID.'Alias' += "$($IDType)ID"
    }

    New-MyDynamicParam -ParamDic $Attribs
}

function Get-MyResource {
    [cmdletbinding()]
    param()

    DynamicParam{
        MyCommonParams
    }

    Begin {
        $CommParams = MyCommonParams
    }
    Process {
        ForEach ($Comm in ($CommParams.Keys)) {
            Set-Variable -Name $Comm -Value $PSBoundParameters.$Comm
            If (-not [string]::IsNullOrEmpty((Get-Variable -Name $Comm))) {
                Set-Variable -Name $Comm -Value $CommParams.$Comm.Value
            }
        }
        Write-Debug "Using $MyProfile to get $($MyID -join ',')"
        If ($MyID) {
            Get-APIEndpoint -Endpoint "Resource" -Profile $MyProfile -APIParam @{ "Ids" = $MyID -join ','}
        } else {
            Get-APIEndpoint -Endpoint "Resource" -Profile $MyProfile
        }
    }
}

function Get-APIEndpoint {
    param($EndPoint,
        $Profile,
        $ApiParam)
}

function Get-MyProfile {
    @('Bob','Alice')
}

function Get-MyDefaultProfile {
    (Get-MyProfile)[0]
}
```