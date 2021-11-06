---
blurb: After several tragedies befell my Minecraft server, I decided to setup some automated tasks to help me manage it. Here is how.
title: Scheduled Tasks for Minecraft servers
path: scheduled-tasks-minecraft-servers
tags:
    - Minecraft
    - Cron
    - Jobs
    - Linux
published: 2021-05-03
lastEdited: 2021-06-14
length: 5
---

There are some absolutely incredibly fun stories I have had playing with my friends in Minecraft. On the other hand there are times when things went terribly wrong.

We (the admins) allowed a few players to join from our friends online community, we lived in relative peace for a time. One day our buildings were being burned down and covered in lava, it was clear these players were responsible. We also found they were using various cheats to disrupt the community. We banned them from the server of course, but the damage had already been done and there was substantial effort required to restore buildings, items and precious building materials. This happened more than once unfortunately; access is now more restricted.

Another time my admin friend decided to play a prank on me using admin commands to change ownership of my pet wolf ot himself. This ended in disaster, he was inexperienced using complex admin commands and the end result was apocalyptic. Lighting was summoned on my approximately 100 wolves spread across the entire world, some being deep underground. The results were devastating; destroying structures, causing fires to break out, much of the community was in ruins. The server itself was in an unrecoverable state; it was eating up memory and lagging intensely. I had been doing manual backups of the server at the time and had to do a complete restore, we lost more than a weeks worth of work.

You never know where tragedy is going to strike, but we can at least be prepared. To prevent incidents like this happening in the future or losing our world altogether, I decided to use my software knowledge to solve this problem. A set of PowerShell scripts to perform backup, restart and settings updates were scheduled via cron jobs. I had never setup cron jobs before, but they were much easier than I expected.

## Prerequisites

* Linux Machine
* Powershell 6+
* Minecraft Server
* Supervisor

## Managing Backups

Performing a backup is easy, just zip up the contents of the Minecraft directory, you can download the zip file to wherever you want. Make sure the `Backups` folder is not a child of the folder you are archiving otherwise you have backups inside your backups (yo dawg).

```powershell
[CmdletBinding()]
param (
    [Parameter(Position = 0)]
    [ValidateNotNullOrEmpty()]
    [String]
    $ServerDirectory = "/home/minecraft/Server",

    [Parameter(Position = 1)]
    [ValidateNotNullOrEmpty()]
    [String]
    $BackupDirectory = "/home/minecraft/Backups"
)

if(!(Test-Path $BackupDirectory)) {
    New-Item -Path $BackupDirectory -ItemType Directory
}

$BackupName = "Minecraft-World-Backup-" + (Get-Date -Format "dd-MM-yyyy")
zip -r "$BackupDirectory/$BackupName.zip" $ServerDirectory
```

## Scheduled Restart & Settings

The `server.properties` file determines the launch settings of the Minecraft server. If I can modify this file using regex, the new settings will be applied each time the server is restarted. I found out the hard way, you have issues if you make changes on the live server. Therefore, perform changes on a temporary file `server_new.properties` and replace the settings on each restart. I will cover how to fully setup Minecraft in a different article, but for now this is all I need to do.

```powershell
[CmdletBinding()]
param (
    [Parameter(Position = 0)]
    [ValidateNotNullOrEmpty()]
    [String]
    $ServerDirectory = "/home/minecraft/Server"
)

# restart the server with the new properties.
supervisorctl stop minecraft
Copy-Item "${ServerDirectory}/server_new.properties" "${ServerDirectory}/server.properties"
supervisorctl start minecraft
```

The scripts I created to manage my Minecraft server and are available on [Github](https://github.com/kaelanhr/MinecraftManagementScripts){target="__blank"}, I chose PowerShell because I prefer it over bash. I know I could have used OOP to take all settings and output the file but it really was not necessary, there are only a small number of properties I need to change and I did not need to over engineer, features are only great if they are actually useful. In this case I only need to change `difficulty`, `monsters spawns` and `motd` properties.

## Creating cron jobs

Cron jobs are automated scheduled tasks. To create cron jobs, type `crontab -e` into the terminal. This will open the cron jobs file in a text editor e.g. nano. You can then setup scripts to run using the following format:

```zsh
# At 00:00 every Sunday, perform a backup.
0 0 * * 0 pwsh -f /home/minecraft/MMS/Scripts/Backup.ps1

# At 03:00 every day, restart the server and apply new config.
0 3 * * * pwsh -f /home/minecraft/MMS/Scripts/RestartServer.ps1

# At 02:00 every day, select Random Quote for the following day.
0 2 * * * pwsh -f /home/minecraft/MMS/Scripts/Set-Motd.ps1

# At 02:00 every Friday, set the difficulty to hard
0 2 * * 5 pwsh -f /home/minecraft/MMS/Scripts/Set-Difficulty.ps1 -Difficulty hard

# At 02:00 every Monday, turn Monsters on for the week.
0 2 * * 1 pwsh -f /home/minecraft/MMS/Scripts/Set-Monsters.ps1 -MonstersEnabled:true

# At 02:00 every Saturday, turn monsters off
0 2 * * 6 pwsh -f /home/minecraft/MMS/Scripts/Set-Monsters.ps1 -MonstersEnabled:false
```

I would highly recommend using [crontab guru](https://crontab.guru/){target="__blank"} to assist your own cron job configuration.

It is possible there is another solution to this problem which sends the property updates to the live running server, but with my current setup I could not figure out how to implement this without adding unnecessary complexities. I hope this article has given you an idea of just how useful cron jobs can be for every day tasks.
