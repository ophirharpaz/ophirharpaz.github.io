---
layout: post
title:  "A Speed-Research on Windows Explorer's Auto-Completion"
---

It was a couple of days ago at work. I pressed the keyboard shortcut _Winkey+R_ to open Windows' _Run_ dialog-box and 
started typing a file-system path to open. While doing so, the application suggested a couple of paths for me to choose 
from. 

![Windows Run dialog-box auto-completing text](/images/2019_08_09_run_dialog_box.png "Windows Run dialog-box auto-completing text")

Among these paths, I noticed, was a path to a folder I had previously deleted.

> <span style="font-size:32px;">Why is _Run_ suggesting a non-existing path? and how can I remove this path from the drop-down list?</span>

This was the question that triggered my research, whose process and findings I will share in this post.

### First Thing - First Aid
Having the above-mentioned question in mind, I didn't know where to start. 
The run dialog-box was some magical popup that I knew only as an end-user. I had absolutely no idea how to approach this problem.
Luckily, [Daniel](https://twitter.com/ace__pace) my team-leader has the answer (or the path to it) to any Windows question.

I simply asked him: "What would you do if you wanted to know how the Run dialog-box auto-completed your typed strings?". 
In response, he gave me a short intro to Windows' windows (:neutral_face:), told me to start by finding the Run window 
object and set me off.

### The Research Process
In hindsight, I can split the research into 3 stages:
1. **Identify the process** which is responsible for the _Run_ dialog-box;
2. **Find the autocomplete-related events** performed by this process;
3. **Conclude the answer** to my question.

# 1. Identifying the Run dialog-box's Process 
In order to understand where _Run_ stored its saved paths, I needed to first find the process responsible for its dialog-box. 
Both _Process Explorer_ and _Process Monitor_ from [Sysinternals Suite](https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite) 
help with window-to-process mapping; they have a nice feature that lets you point a special cursor at a UI window 
and get the process which created it.

![Process Explorer enables to locate a window's process by hovering over it with a designated cursor](/images/procexp_cursor.png "Process Explorer Find Window's Process")

Another tool named [_Spy++_](https://docs.microsoft.com/en-us/visualstudio/debugger/introducing-spy-increment?view=vs-2019) 
comes bundled with Microsoft Visual Studio. Compared to _Process Monitor_ and _Process Explorer_, _Spy++_ is much more 
window-oriented and displays the complete window hierarchy of the current session. 
I decided to go with _Spy++_ as it was both new to me and felt more accurate for the mission.

Using _Spy++_ I identified the _Run_ dialog-box's process. It turned out that this window is named _Run_ and belongs to 
the _explorer.exe_ process (the gif below shows only the process ID; I inferred the process name using _Process Explorer_).

![Spy++ enables mapping between windows and their processes](/images/spy_process.gif "Spy++ enables mapping between windows and their processes")

# 2. Tracing explorer.exe's Auto-Complete Events
My go-to tool for monitoring a process is, well, _Process Monitor_. 
Once I knew I should concentrate on _explorer.exe_, I started capturing events in _Process Monitor_, opened the _Run_ 
dialog box and started typing arbitrary paths. Then I used a filter to display only _explorer.exe_ events.
The problem is, _explorer.exe_ creates shit-tons of events, non-stop. 

I went back to _Spy++_ looking for more information on the _Run_ window, and noticed its thread ID (TID). 
With this information, the task of filtering out irrelevant _explorer.exe_ events became much easier.

Even after filtering, there were still many events. Briefly scanning them, I saw a registry-write event referring to a 
key named _**RunMRU**_. Seeing the word "Run", and recalling that _MRU_ stands for "most recently used", I figured this 
might be the place I was searching for.

![Process Monitor shows registry-write events, accessing a key named RunMRU](/images/mru_in_procmon.png "Registry Writes to MRU Keys")


# 3. Reaching Conclusions
I went straight to the registry path `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU\MRUList` where I 
found my answer:

![Windows Registry contains the most-recently used paths typed in the Run dialog-box](/images/registry.png "MRU Paths in the Registry")


This key listed my most recently used paths, and was clearly "feeding" the _Run_ auto-complete feature.
The path to the non-existent folder was there -- and _this_ is why the _Run_ dialog-box kept suggesting it!
Woohooooooooo!

I tested my theory by evicting the non-existent path from the MRU list. I simply entered new paths to the text box until
the non-existent path was pushed out. Once the path was no longer in the registry, it never appeared again in the 
drop down list.


Actually, the MRU mechanism is pretty sweet so I'll elaborate a bit in the box below.
```
explorer.exe saves 26 paths at each given moment. 
Why 26? Because each path is saved in a key whose name is a letter from a to z. 
The "most" part in the "MRU" is encoded in the MRUList key. 
This is a string consisting of all 26 alphabet letters, 
the first letter being the newest entry, the last letter being the oldest. 
This string determines the order of the paths that appear when you hit the down-arrow 
in the Run dialog-box. 
```

# 4. One Last Investigation
Even with the MRU, there was still something unclear. I noticed two "autocomplete scenarios":
1. _Run_ suggests **non-existing paths** that I **once typed**;
2. _Run_ suggests **existing paths** that I **never-in-my-life typed**.

The first scenario was explained by the MRU list, but the second still needed clarification.

TODO Show the autocomplete COM object referral.

### Summing Up
This short blog post shows how I did a quick research on a mechanism in Windows that I found interesting. 
I found out that file-paths auto-completion is done based on:
- a list of most-recently used file paths;
- real-time traversal of the file system.

I believe these questions, that arise from one's curiosity, are the ones that are most satisfying to answer :)
I hope you always stay curious enough to ask questions and restless enough to answer them. 