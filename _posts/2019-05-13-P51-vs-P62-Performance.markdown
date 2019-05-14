---
layout: post
title: "Windows Powershell vs Core: Of Course Core is Faster...But Really?" 
date:   2019-05-13 09:03:37 -0400
categories: powershell
tags: powershell, powershell core, pwsh
image: https://avatars2.githubusercontent.com/u/20189855
---

## Look, Don't Leap

When Powershell Core first came out, the AD module was not supported. That's non-starter for me, 
so I hadn't really done anything with it. After attending [PSHSummit 2019](https://twitter.com/hashtag/PSHSummit?src=hash),
I decided to give it another look, and was really surprised at how much faster it turned out to be in certain circumstances. 

## The Showdown

I recently had to audit some Exchange settings, to the tune of ~70,000 mailboxes. I have a job that pulls down a list of all
the mailboxes, with attributes I use across a number of jobs. The report I had to write concerned several mailbox attributes, but 
for the sake of simplicity, let's create a csv file with two fields, "UserID" and "Policy".

### Generate

Let's make our source data file.

```powershell
PS> $users = 0..69999 | Foreach {   `
    $i = "{0:00000}" -f $_          `  
    [pscustomobject]@{              `
        UserId = "User_$i"          `
        Policy = 'Default'          `
    }                               `    
}

# Let's change some values
PS> $users[20001].Policy = 'SuperSecure'
PS> $users[10001].Policy = 'SuperSecure'
PS> $users[35001].Policy = 'SuperSecure'
PS> $users[19001].Policy = 'LooseyGoosey'
PS> $users[59999].Policy = 'LooseyGoosey'
PS> $users[69999].Policy = 'LooseyGoosey'

# ...and write to file.
PS> $users | export-csv users.csv
```

### Testing it out

Sweet. Now say we need to find anyone with a 'LooseyGoosey' policy, so we can revisit that policy decision 
or take action on the mailbox. 

```powershell
# Powershell 5.1
PS> $users = import-csv users.csv

# First, using the .Where Method
PS> Measure-Command { $users.Where( {$_.Policy -eq 'LooseyGoosey'} ) }

Days              : 0
Hours             : 0
Minutes           : 2
Seconds           : 2
Milliseconds      : 159
Ticks             : 1221594732
TotalDays         : 0.00141388279166667
TotalHours        : 0.033933187
TotalMinutes      : 2.03599122
TotalSeconds      : 122.1594732
TotalMilliseconds : 122159.4732

# Now, let's try piping this business
PS> Measure-Command { $users | Where-Object { $_.Policy -eq 'LooseyGoosey' } }

Days              : 0
Hours             : 0
Minutes           : 2
Seconds           : 4
Milliseconds      : 942
Ticks             : 1249421529
TotalDays         : 0.00144608973263889
TotalHours        : 0.0347061535833333
TotalMinutes      : 2.082369215
TotalSeconds      : 124.9421529
TotalMilliseconds : 124942.1529
```

Oof... 2+ minutes. Let's do this in Core (6.2)

```powershell
# Powershell 6.2
PS> $users = import-csv users.csv

# .Where Method
PS> Measure-Command { $users.Where( {$_.Policy -eq 'LooseyGoosey'} ) }

Days              : 0
Hours             : 0
Minutes           : 0
Seconds           : 0
Milliseconds      : 804
Ticks             : 8041803
TotalDays         : 9.30764236111111E-06
TotalHours        : 0.000223383416666667
TotalMinutes      : 0.013403005
TotalSeconds      : 0.8041803
TotalMilliseconds : 804.1803

# Now with pipes
PS> Measure-Command { $users |  Where-Object {$_.Policy -eq 'LooseyGoosey'} }
    
Days              : 0
Hours             : 0
Minutes           : 0
Seconds           : 1
Milliseconds      : 730
Ticks             : 17307062
TotalDays         : 2.00313217592593E-05
TotalHours        : 0.000480751722222222
TotalMinutes      : 0.0288451033333333
TotalSeconds      : 1.7307062
TotalMilliseconds : 1730.7062

```

2 minutes down to ~1 second. Pretty crazy. The original script I wrote was more complex, making use of a couple 
of data sources and checking against multiple conditions. Executing the script with pwsh took the runtime from ~11 
minutes, down to about 12 seconds, without changing any code. Now I'm looking at swapping in Powershell Core on a
number of data crunching jobs.
