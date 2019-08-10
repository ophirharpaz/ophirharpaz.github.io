---
layout: post
title:  "A Speed-Research on Windows Explorer's Auto-Completion"
---

### Motivation
It was a couple of days ago at work. I pressed the keyboard shortcut _Winkey+R_ to open Windows' _Run_ dialog-box and 
started typing a file-system path to open. While doing so, the application suggested a couple of paths for me to choose 
from. 

![Windows Run dialog-box auto-completing text](/images/2019_08_09_run_dialog_box.png "Windows Run dialog-box auto-completing text")

Among them, I noticed, was a path to a folder I had previously deleted.

> <span style="font-size:32px;">Why is _Run_ suggesting a non-existing path? and how can I remove this path from the drop-down list?</span>

This was the question that triggered my research, whose process and findings I will share in this post.

### First Thing - First Aid
Having the above-mentioned question in mind, I didn't know where to start. 
The run dialog-box was some magical popup that I knew only as an end-user. I had absolutely no idea how to approach this problem.
Luckily, [Daniel](https://twitter.com/ace__pace) my team-leader has the answer (or the path to it) to any Windows question.

I simply asked him: "What would you do if you wanted to know how the Run dialog-box auto-completes your typed strings?". 
In response, he gave me a short intro to Windows' windows (:neutral_face:), taught me how to locate a UI window using 
_Spy++_ and set me off.

### The Research Process
In hindsight, I can divide the process into 3 stages:
1. **Identify the process** which is responsible for the _Run_ dialog-box;
2. **Find the autocomplete-related events** performed by this process;
3. **Get to a conclusion**.

# 1. Identifying the Run dialog-box's Process 
In order to understand where _Run_ stored its saved paths, I needed to first find the process responsible for its dialog-box. 
Both _Process Explorer_ and _Process Monitor_ from [Sysinternals suite's](https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite) 
help with window-to-process mapping; they have a nice feature that lets you point at a UI window object and get the 
process which created it.

![Process Explorer enables to locate a window's process by hovering over it with a designated cursor](/images/procexp_cursor.png "Process Explorer Find Window's Process")

Another tool named [_Spy++_](https://docs.microsoft.com/en-us/visualstudio/debugger/introducing-spy-increment?view=vs-2019) 
comes bundled with Microsoft Visual Studio. Compared to _Process Monitor_ and _Process Explorer_, _Spy++_ is much more 
window-oriented and displays the complete window hierarchy of the current session. 
I decided to go with _Spy++_ as it was both new to me and it felt more accurate for the mission.

Using _Spy++_ I identified the _Run_ dialog-box's process. It turned out that this window is named _Run_ and belongs to 
the _explorer.exe_ process.

![Spy++ enables mapping between windows and their processes](/images/spy_process.gif "Spy++ enables mapping between windows and their processes")

# 2. Tracing explorer.exe's Events (...that are relevant to the autocomplete feature)
The classical tool for tracing the events of a process is, for me, _Process Monitor_. 
Once I knew I should concentrate on _explorer.exe_, I looked at its events in _Process Monitor_. 
The problem is, _explorer.exe_ creates shit-tons of events, all the time, non-stop. 

I went back to _Spy++_ looking for more information on the _Run_ window, and noticed its thread ID (TID). 
With this information, the task of filtering out irrelevant _explorer.exe_ events turned much easier.

Even after filtering, there were still many events. Briefly scanning them, I saw a registry-write event referring to a 
key named _**RunMRU**_. Seeing the word "Run", and recalling that _MRU_ stands for "most recently used", I figured this 
might be the place I was searching for.

![Process Monitor shows registry-write events, accessing a key named RunMRU](/images/mru_in_procmon.png "Registry Writes to MRU Keys")


# 3. Reaching Conclusions
I went straight to the registry path `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU\MRUList` where I 
found my answer:

![Windows Registry contains the most-recently used paths typed in the Run dialog-box](/images/registry.png "MRU Paths in the Registry")


This was a list with my most recently used paths, and this list was "feeding" the _Run_ auto-complete feature.
Indeed, the no-longer-existent path was there -- and _this_ is why the _Run_ dialog-box kept suggesting it!
Woohooooooooo!

I tested this by evicting the non-existent path from the MRU list (by simply typing in new paths). Once the registry 
entry was "out" - the path never appeared again in the drop down list.

Actually, the MRU mechanism is pretty sweet so I'll elaborate a bit. 
_explorer.exe_ saves at each given moment 26 paths. 
Each path is saved in a key whose name is a letter from _a_ to _z_. 
The "most" part in the "MRU" is encoded in the _MRUList_ key. This is a string consisting of all 26 alphabet letters,
the first letter being the newest entry and the last one being the oldest. This string determines the order of the
paths appearing when you hit the down-arrow multiple times while in the _Run_ dialog-box. 

# 4. One Last Investigation
There was still something unclear. I noticed two "autocomplete scenarios":
1. _Run_ suggests **non-existing paths** that I **onced typed** - explained by the MRU list;
2. _Run_ suggests **existing paths** that I **never-in-my-life typed**.

The second scenario was still an open question to me.

TODO Show the autocomplete COM object referral.

### Summing Up
This short blog post shows how I did a quick research on a mechanism in Windows that I found interesting. 
I found out that file-paths auto-completion is done based on:
- a list of most-recently used file paths;
- real-time traversal of the file system.

I believe these questions, that arise from one's curiosity, are the ones that are most satisfying to answer :)
I hope you always stay curious enough to ask questions and restless enough to answer them. 