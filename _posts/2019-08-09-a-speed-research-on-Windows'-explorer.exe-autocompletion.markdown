# Motivation
It was a couple of days ago at work. I pressed the _Winkey+R_ combination to open the Windows' Run dialog-box and started typing a file-system path to open. While doing so, the application suggested a couple of paths for me to choose from. Among them, I noticed, was a path to a folder I had previously deleted.
Why is this "widget" suggesting a non-existing path, and how can I delete this path?
This was the question that triggered my mini-research. 

<Picture>

This post's goals are to both share my research process & results and introduce the readers to the tools I used. However, I think a good side-effect is to encourage the readers to follow and satisfy their curiosity.

# First Thing - First Aid
To be honest, when I had the above-mentioned question in mind I had absolutely no idea how to approach it. The run dialog-box was some magical popup that I knew only as an end-user. I just didn't know where to start. 
Luckily, Daniel my team-leader is a Windows robotrick - given a question, 95% chance he has an answer. Otherwise, he just knows how obtain one.
I simply asked him "What would you do if you wanted to know how the Run dialog-box autocompletes your typed strings?". In response, he gave me a short intro to Windows' windows (ha), showed me some tools and set me off.

# The Research Process
In hindsight, I can divide the process into 3 stages:
1. **Identify the process** which is responsible for the Run dialog-box;
2. **Trace the autocomplete-related actions** performed by this process;
3. **Get to a conclusion**.

## 1. Identifying the Run dialog-box's Process 
As a user, the Run dialog-box was for me a UI window. As a researcher, I needed to think of it as a process. 
[Sysinternals suite's](https://google.com) has (at least) two tools to help with window-to-process mapping: _Process Explorer_ and _Process Moniter_. Both have a nice feature that lets you point at a UI window object and get the process which created it.

<gif>

Another tool named _Spy++_ comes bundled with Microsoft Visual Studio. Compared to Procmon and Procexp, _Spy++_ is much more windows-oriented and actually displays the hierarchy of windows per running process. I decided to go for _Spy++_ because it was both new to me and more designated for the mission. I used _Spy++_ to identify the Run dialog-box's process. It turned out that this window is named _Run_ and is belonged to the _explorer.exe_ process.

<gif showing how I got the process ID from Spy++>

## 2. Tracing explorer.exe's Events (...that are relevant to the autocomplete feature)
The classical tool for tracing the events of a process is, for me, _Process Monitor_. Once I knew to concentrate on explorer.exe's, I looked at the events this process created in _Process Monitor_. The problem is, explorer.exe creates shit-tons of events, all the time, non-stop. I went back to _Spy++_ looking for more information on the _Run_ window, and saw its thread ID (TID). Having this datum, the task of filtering out irrelevent explorer.exe events turned much easier.

<Some grpahic/fig showing the thread filtering>

At this point I had much fewer events to look at, or in other words, much less text to pop into my sight. It was then when I saw a registry-write event* referring to a key named _**RunMRU**_. Seeing the word "Run", and recalling that _MRU_ stands for "most-recently used", I figured this might be the place I searched for. 

*_RegSetValue_ is the Windows API function for writing data to the registry. 

## 3. Reaching Conclusions
I went straight to the registry path `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU\MRUList` where I saw this amazing thing:

<picture of the MRU list data>

This was a list of paths which I had most-recently used. Indeed, my no-longer-existent path was there -- and this is why it the Run dialog-box kep suggesting it! Woohooooooooo!
Nevertheless, there was still something unclear. I noticed two autocomplete scenarios:
1. _Run_ suggests **non-existing paths** that I **onced typed** - explained by the MRU list;
2. _Run_ suggests **existing paths** that I **never-in-my-life typed** - still needed a clarification.

TODO Show the autocomplete COM object referral.

# Summing Up
This short blog post shows how I did a quick research on some mechanism in Windows that I found interesting. I found out that file-paths autocompletion is done based on:
- real-time traversal of the file system;
- a list of most-recently used file paths.

I believe these questions, that arise from one's curiosity, are the ones it is most satisfying to answer :)
I hope you always stay curious enough to ask questions and restless enough to answer them. 