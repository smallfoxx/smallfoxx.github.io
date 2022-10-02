---
layout: post
title: Events with PowerShell Automation
---

If you're looking for a PowerShell script to help you plan out or schedule your event, this is not the post you're looking for; maybe another time.

However, this post was inspired by [a Reddit post](https://www.reddit.com/r/PowerShell/comments/xssra8/powershell_script_issue/) where u/anacondaonline was having an issue with [a script](https://pastebin.com/dLGSL2S9) that they were using to move files placed in a folder. This is actually a really great use of events and was something I don't see enough from novice PowerShell users so was really glad to see a really valid attempt and wanted to help out. Unfortunately, [my response](https://www.reddit.com/r/PowerShell/comments/xssra8/comment/iqn0gzj/?utm_source=share&utm_medium=web2x&context=3) was hardly explinative and was too hard to follow.  Rather than try to constrain an explination to a Reddit post, welcome to my blog spew!!!  If you don't want the details, you can jump down to the [TL;DR](#tldr) script for details.

## Event driven world

As a quick primer, Events are simply things that happen in a computing system you can watch and/or respond to.  As a PowerShell admin, we're often Windows Admins and Windows uses lots of Events to drive activities in the system. Its not quite a true Event Driven OS as it still does some Time Sharing, but it provides the ability to registry with events and trigger your activities rather than constantly running a program and constantly polling for details.

Events can be anything from a mouse click, new file, program starting to a timed event or measured value from a sensor. The idea here is that you register as a watcher of an event with the OS and when the OS encounters the matching event, it triggers your action.

However, we can encounter these outside of Windows, too. You can have events from a Webhook, a web PUSH update, file upload, or data threshold. Utilizing events lets you keep from wasting resources when there is not something happenning and responding to events as they happen. This can also help with scaling, too, because if you are getting lots of events, you can have each spawn their own response and have some an inherant multithreading with very little effort.

## Premise

In this scenario, we're looking to copy all files out of a directory when some one copies a file into it.  This was the code in question:

```powershell
Unregister-Event FileCreated
$folder = 'C:\pstest'
$filter = '*.*'
$fsw = New-Object IO.FileSystemWatcher $folder, $filter 
$fsw.IncludeSubdirectories = $true 
            
$fsw.NotifyFilter = [IO.NotifyFilters]'FileName, LastWrite'
$onCreated = Register-ObjectEvent $fsw Created -SourceIdentifier FileCreated -Action {
    $name = $Event.SourceEventArgs.Name
    $path = $Event.SourceEventArgs.FullPath
    Write-Host "The file '$name' '$path'"
    cmd.exe /c 'move-files.bat'
}
```

In a basic file copy this works fine, but the OP ran into some issues. So lets explain what this script does, then talk about the challenge, and finally talk about some options.

### Explination

The first thing to understand with events in Windows, is there are events going off all the time, but nothing happens unless there is something registered with the event.  Usually, we'll have something like a watcher that gets triggered and then performs in action.

In this case, we want to watch a folder, so they created a watcher for the file system and then registered it with the Creation event and an action to then call a batch file.

The first line is just to remove an existing registration so we don't run into a conflict and clear out what is there:

```powershell
Unregister-Event FileCreated
```

We actually create the watcher object ([`FileSystemWatcher`](https://learn.microsoft.com/en-us/dotnet/api/system.io.filesystemwatcher.-ctor#system-io-filesystemwatcher-ctor(system-string-system-string)) (FSW)) first and included the parameters of the folder we want to watch and a filter for all the files:

```powershell
$fsw = New-Object IO.FileSystemWatcher $folder, $filter 
```

In this case, the `$folder` is `c:\pstest.txt` and the `$filter` is telling it to include all files (`*.*`).  Also, the [`IO.`](https://learn.microsoft.com/en-us/dotnet/api/system.io) is because this is part of the Input-Output of `System`.

The next two lines are to setup some parameters for the FSW:

```powershell
$fsw.IncludeSubdirectories = $true 
            
$fsw.NotifyFilter = [IO.NotifyFilters]'FileName, LastWrite'
```

The first one is probably pretty easy to get, but basically we're including not only files within the current folder, but in files in folder within the current folder.

The NotifyFilter is where we can tell the system only to trigger if the FileName or LastWrite time are changed.  This is good for detecting when a file is created, renamed, or changed in some way.

The next 5 lines is actually registering with the event and specifying what code we want it to execute when it does (the `-Action`)

```powershell
$onCreated = Register-ObjectEvent $fsw Created -SourceIdentifier FileCreated -Action { <#next 4 lines#> }
```

The `$oncreated` variable is just storing the registered event. If we were doing more it might be relevant, but we could have also used `$null` to limit memory usage as we never reference it again.

The `$fsw` is our InputObject and the name is `Created`.  The `-SourceIdentifier` is the unique identifier for each register; `FileCreated` in this case. The `-Action` parameter defines the ScriptBlock, or code, we use to execute when the event occurs.

The Action code in this case was capturing variables for display, but the real activity was being down by calling the command shell (`cmd`) and then running `move-files.bat`.

## The Issue

This is a good basic event trigger, but it runs into a problem with any real activity. The main issue is code block calls a batch file to move all the files in a folder, but the event is triggered ever time a file is put in there. So if you put in 5 files, the event is triggered for the first file and the batch file attempts to move everything in the directory while potentially the other 4 files are still being copied.  The event will also trigger 4 more times, each one trying to move the full content of the folder.

1. Possible error moving files while in use
2. Innefficiently repeat activity multiple times

So we have mutliple triggers of a mass action; this is actually a common scenario we run into in an event driven environment

## Resolution

There are two realistic solutions to this particular issue:

1. As each file triggers the event, move that individual file
2. Queue up the files and move them all at once

The first one wouldn't be too bad. Just use a Move-Item to move the individual file as the Action:

```PowerShell
Move-Item -Path $Event.SourceEventArgs.FullPath -Destination 'C:\Dest'
```

However, this could be really inneficient especially if our destination is remote and we have to go through a whole authentication for each file. It would be more efficiently and hey give us the chance to play with more tech if you could do a queue.

### Let's Trigger a Queue

To free this up, lets decouple the main two things we're actually doing here:

1. Capturing file names that were copied
2. Move files to the destination

This is the same type of things we try to do when we move away from straight scripts into function based code. If we can encapsulate a single action into a function, we can make our code more easily reusable and flexible.

The big issue we'd run into is we have to get the file names and some communicate them to the move.  If we continue to do a 1-to-1 calling a move for each file name that was copied we haven't improved anything. What we can do is put the names of the files into a buffer or a queue.

There are a lots of different mechanisms out there we could use for a queue, but for simplicity sake, we'll use a text file and put all the file paths into that text file.  Then we can have something come back and process that queue file to move all the file paths in it.

We could put this file anywhere, but we have a temporary folder variable in PowerShell: `$env:TEMP`. We can use that as a storage folder and just have our move process look for our file in that folder; let's call it `PSQueue.txt`.

Calling a batch file or `Move-Item`, we can use `Add-Content` to add the paths to the file.  If the file doesn't exist, it will create it; if it does exist, it will just add it as a new line on the file.  So for our Action code block we can just do:

```powershell
    $filePath = $Event.SourceEventArgs.FullPath
    $queueFile = "$env:TEMP\PSQueue.txt"
    Write-Host "Adding '$filePath' to '$queueFile'"
    Add-Content -Path $queueFile -Value $filePath
```

Now, we could just have a script that just ran in a constent loop looking for changes to `$env:TEMP\PSQueue.txt`, we could also have a scheudled task run every 5 minutes to see if that file was there, or have someone manually come in and drag & drop the queue file onto a script to execute.  However, those aren't really helping us automate the process.

If only there were some way we could have it trigger a code block when the queue file gets created?

#### Eureka!!

We were just doing a file trigger for moving files out of a folder. How about we trigger when the queue file gets created??

Since we have a different folder, different activity, and different goals, we'll put this into a separate script file.  We can reuse our trigger file code but we can move our path and filter to focus on just that folder and file:

```powershell
$folder = $env:TEMP
$filter = 'PSQueue.txt'
$fsw = New-Object IO.FileSystemWatcher $folder, $filter 
$fsw.IncludeSubdirectories = $false 
$fsw.NotifyFilter = [IO.NotifyFilters]'FileName, LastWrite'
```

We'll use a unique name for our `-SourceIdentifier` and `-name` for our event object to register to be unique.

```powershell
$null = Register-ObjectEvent $fsw -EventName CreatedQueue -SourceIdentifier QueueFileCreated -Action { <#code block here#> }
```

For our action, we can get the contents of the queue file as an array of strings with companion cmdlet to Add-Content: `Get-Content`.  We can then use that array and feed it directly to `Move-Item` to move the files to a destination.

```powershell
#Action codeblock
$destFolder = "c:\PSDest"
$queueFile = $Event.SourceEventArgs.FullPath
$filesInQueue = Get-Content -Path $queuefile
Write-Host "Moving $($filesInQueue.count) files to $destFolder"
Move-Item -Destination $destFolder -Path $filesInQueue
```

Works great right? Everything is glorious?

Not exactly. Everytime we see a file in the source folder, our triggered event is adding it to the queue file, but our queue watcher is triggered to only when the queue file is **created**.  Moreover, we're never clearing out the content of the queue, so the queue is just going to get bigger and bigger, but we're only going to trigger it the first file put in there.

That's actually an easy fix. After we've gotten all the files out of the queue, we can simply signal we're going to take care of those files by clearing out the queue by just removing the file:

```powershell
#Action codeblock
$destFolder = "c:\PSDest"
$queueFile = $Event.SourceEventArgs.FullPath
$filesInQueue = Get-Content -Path $queuefile
Remove-Item -Path $queueFile
Write-Host "Moving $($filesInQueue.count) files to $destFolder"
Move-Item -Destination $destFolder -Path $filesInQueue
```

Everything is great, right? Well, this means its going to trigger everytime a file is placed in the source and probably every one or two files; hardly optimized.

Since the trigger will happen as soon as the queue is created, we can have our trigger wait a bit to let the queue fill up before it grabs the contents of the file and clears it out.  So we'll use `Start-Sleep` to have the file stop for a few seconds after being triggered before continuing on:

```powershell
#Action codeblock
$destFolder = "c:\PSDest"
$queueFile = $Event.SourceEventArgs.FullPath
Write-Host "Waiting 5 seconds for queue to build and copies to finish..."
Start-Sleep -Seconds 5
$filesInQueue = Get-Content -Path $queuefile
Remove-Item -Path $queueFile
Write-Host "Moving $($filesInQueue.count) files to $destFolder"
Move-Item -Destination $destFolder -Path $filesInQueue
```

## Closing up thoughts

This is a simple trigger -> queue -> process built on a Windows file system, but this type of mentality can be brought to several different systems. Queues and Hubs are common resources to track jobs and pending activities in many distributed computing systems. Its a quick way to get multitasking involved because you can have multiple processes grab things off the queue, but you can also have thousands of processes dropping things in the queue.

We can utilize these same techniques in our automation processes. Its normal for us to think in a very linear process: A causes B and results in C.  However, if B is slower than A, or C, we suddenly have a bottleneck. If we can decouple them and not have one monolithic process or script, we can start utilizing multiple resources.

The idea is that A does its task and relies on B to complete its task and send it on to C. Therefore, in a production environment, we'd want to do more error handling and checking, but there are plenty of options around this.

This is just another tool to put in your toolbox to build off of. Its not for every solution and every automation out there, but its possible.

You might even have a powershell script drop off a job in an event hub and feed it into a function to trigger a web hook before you know it. ;)

## Links

- [Register-ObjectEvent help](https://learn.microsoft.com/en-us/powershell/module/Microsoft.PowerShell.Utility/Register-ObjectEvent)
- [FileSystemWatcher](https://learn.microsoft.com/en-us/dotnet/api/system.io.filesystemwatcher)
- [Add-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/add-content)

## TL;DR

So just a quick summary, if you need to move files from one folder as their placed into it, you can use an event trigger to capture their names into a queue and then use another trigger to respond and process that queue:

Queue watcher script:

```powershell
# Queue watcher script

## Clear out any existing event, but keeping going if we didn't already have one
Unregister-Event -SourceIdentifier QueueFileCreated -ErrorAction Continue

$folder = $env:TEMP
$filter = 'PSQueue.txt'
$fsw = New-Object IO.FileSystemWatcher $folder, $filter 
$fsw.IncludeSubdirectories = $false 
$fsw.NotifyFilter = [IO.NotifyFilters]'FileName, LastWrite'

$null = Register-ObjectEvent $fsw -EventName CreatedQueue -SourceIdentifier QueueFileCreated -Action {
    $destFolder = "c:\PSDest"
    $queueFile = $Event.SourceEventArgs.FullPath
    Write-Host "Waiting 5 seconds for queue to build and copies to finish..."
    Start-Sleep -Seconds 5
    $filesInQueue = Get-Content -Path $queuefile
    Remove-Item -Path $queueFile
    Write-Host "Moving $($filesInQueue.count) files to $destFolder"
    Move-Item -Destination $destFolder -Path $filesInQueue
}
```

Folder watcher script:

```powershell
# Folder watcher script

## Clear out any existing event, but keeping going if we didn't already have one
Unregister-Event -SourceIdentifier FileCreated -ErrorAction Continue

$folder = 'C:\pstest'
$filter = '*.*'
$fsw = New-Object IO.FileSystemWatcher $folder, $filter 
$fsw.IncludeSubdirectories = $true 
$fsw.NotifyFilter = [IO.NotifyFilters]'FileName, LastWrite'

$null = Register-ObjectEvent $fsw -EventName Created -SourceIdentifier FileCreated -Action {
    $filePath = $Event.SourceEventArgs.FullPath
    $queueFile = "$env:TEMP\PSQueue.txt"
    Write-Host "Adding '$filePath' to '$queueFile'"
    Add-Content -Path $queueFile -Value $filePath
}
```
