# prclone

Contains some notes for synchronization of Mac user files to a remote file storage using rclone

## Motivation

I had some trouble to synchronize my files to [pcloud](https://www.pcloud.com/). This provider offer very interesting services for file storage (5 TB of lifetime online storage), but the software for file synchronization does not work as expected on a Mac (did not managed to make it work correctly on a Macbook Pro M2 and on an old Macbook Air)

I managed to synchronize my data using [rclone](https://rclone.org/)

## Solution

### pcloud
Once you have setup your account, you will need to ask the support team to create for you an "application" that will permit you to access the service with their public API.

As the support told me, you can not create the application by yourself due to previous abuses. They need to control the service usage. You will be asked what is the purpose of your application and they will create it after their internal validation.

After the application is created, you will gain your credentials (a `client id` and a `client secret`)

### rclone

#### Installation and configuration
- Follow the instructions given here https://rclone.org/downloads/
- using `rclone config` command on a terminal, configure the rclone remote service for your pcloud acount: follow the documentation given here: https://rclone.org/pcloud/. During configuration, you will give a name (e.g. `pcloud-remote`) and provide your `client id` and `client Secret`

After this setup, you have a full access to your pcloud files using rclone commands. You can find help using the `man rclone` command or look at the [online documentation](https://rclone.org/commands/)

### One shot synchronization
you can synchronize your local files using the `rclone sync` command

Assuming you want to synchronize the directory `/Users/my-user-name/my-directory` to `my/pcloud/sync-folder` on your pcloud account you named `my-pcloud` during rclone setup, on a terminal, just type
```/bin/bash
rclone sync /Users/my-user-name/my-directory my-pcloud:my/pcloud/sync-folder
```

Be aware that the first synchronization can take a lot of time, depending of the size of your data and the number of files (it can take days, even weeks). Don't be afraid if you have to shutdown your computer during this synchronization. If you relauch the same command, the synchronization process will continue, starting from it's previous state.

If you want to perform synchronization of all your data on your home directory, you may don't want to save some files that don't make sense for a backup. (log files, temporary files, caches, ...). You should create an exclude file that will contain the file patterns you don't want to synchronize. The syntax for the file patterns is given [here](https://rclone.org/filtering/)

/Users/my-user-name/.rclone-exclusions.txt
```
.Trash/**
Library/Caches/**
Library/Logs/**
```

```/bin/bash
rclone sync /Users/my-user-name/my-directory my-pcloud:my/pcloud/sync-folder \
--exclude-from /Users/my-user-name/.rclone-exclusions.txt
```

Out of the box, for security reasons, some files are not readable by rclone (e.g. some files in your Library Folder). You can override this settings by giving the rclone executable the "Full Disk Access" privilege.

- find the rclone executable location (type `which rclone` on a terminal). e.g.: /usr/local/bin/rclone
- Open `System Settings > Privacy & Security > Full Disk Access` from the Apple Menu (&#xF8FF;) or `System Preferences > Security & Privacy > Privacy (onglet) > Full Disk Access` for old systems
- click on `+` to add a new application 
- use `Cmd+Shift+G` shortcut to navigate to the rclone executable location

### Constant synchronization

Once you have the right setiings for rclone, you can automate the execution of your rclone command. For this, you can use the `launch agent` utility. See some documentation here : [technical guide: building a modern launch agent on macos](https://gist.github.com/Matejkob/f8b1f6a7606f30777552372bab36c338#technical-guide-building-a-modern-launch-agent-on-macos)

First you have to describe the agent that will execute continuously your rclone command in a `plist` file.

Here is an example file: notice that there are log files (`/var/log/rclone.log`) that you can look at to see if everything works as expected.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>my-pcloud-synchronization</string>

        <key>ProgramArguments</key>
        <array>
            <string>/usr/local/bin/rclone</string>
            <string>sync</string>
            <string>/your/directory/to/synchronize</string>
            <!-- assuming my-pcloud is the name of your pcloud remote in rclone config-->
            <string>my-pcloud:my/pcloud/sync-folder</string>
            <string>--links</string>
            <string>--exclude-from</string>
            <string>/my/exclude-from-sync.txt</string>
            <string>--log-file</string>
            <string>/var/log/rclone.log</string>
            <string>--log-level</string>
            <string>INFO</string>
            <string>--log-file-max-size</string>
            <string>10M</string>
            <string>--log-file-max-backups</string>
            <string>3</string>
            <string>--log-file-compress</string>
        </array>

        <key>RunAtLoad</key>
        <true/>

        <key>KeepAlive</key>
        <true/>
    </dict>
</plist>
```

Save this file to $HOME/Library/LaunchAgents/my-pcloud-synchronization.plist

Launch the agent

```/bin/bash
launchctl load $HOME/Library/LaunchAgents/my-pcloud-synchronization.plist
```

And that's it! You will have your files synchronized in your pcloud account.

