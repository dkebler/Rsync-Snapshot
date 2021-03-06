Rsync Snapshot
==============

[![NPM](https://nodei.co/npm/rsync-snapshot.png)](https://nodei.co/npm/rsync-snapshot/)

A Node.js implementation of incremental full system backups using rsync based on [rsync - Arch Linux Wiki](https://wiki.archlinux.org/index.php/rsync#Snapshot_backup). Currently only tested on Linux.

See [Incremental Bacukps with rsync](https://medium.com/@mattlyons0/incremental-backups-with-rsync-9c8e3f14c2c) for details on environment configuration

### Features
- Full System Backup
  - Backup of `/` (or any other folder) with permissions and other attributes preserved
  - Encrypted networked backups over the SSH Protocol
  - Transfer file deltas only (No need to transfer entire files, just the changes)
- Incremental History
  - Using Hardlinks incremental history is stored
  - Auto deletion of oldest snapshots after specified number of snapshots is exceeded
- Logging
  - Multiple output modes (json, text and raw rsync output)
  - Log to file
  - Multiple logging levels supported
- Script Hooks
  - Execute scripts before or after backup

### Requirements
- NodeJS v7.6 or later - *async/await is used in codebase*
- Rsync must be installed on the client and the server
- One machine (client) must have SSH access to the other (server) if backing up over network, without a password (pubkey)
  - This script is designed to be run from the machine data is being backed up from (the client)
  - This script requires root access on the client to backup `/` and root access on the server to be able to set correct file ownership
    - In order to run this script passwordless sudo on `rsync`, `rm` and `mv` from the SSH user is required. This must be manually configured on the server. See [Editing /etc/sudoers](https://unix.stackexchange.com/questions/92123/rsync-all-files-of-remote-machine-over-ssh-without-root-user/92125#comment139116_92125) for details
    - If root is not used on server all files will be owned by the user logged in via ssh which will lead to errors on restore

### Usage
- Install Globally `npm install -g rsync-snapshot`
- Execute the backup
  - Locally `rsync-snapshot --dst /media/MyBackup`
  - Remotely `rsync-snapshot --shell ssh --dst username@myserver.com:/media/MyBackup`
- It is recommended to schedule this command to run regularly in cron or alike
  - When scheduling this script run it is best to update `rsync-snapshot` regularly
  - Execute `npm install -g rsync-snapshot` to update to latest version
- Backups will be in a folder named by time and date (ex: `2018-02-21.19-06-26`)
  - Anything may be appended (manually) to folder names to add user friendly info (ex: `2018-02-21.19-06-26.createdDatabase`) as long as `.incomplete` is not appended (which is reserved for backups in progress, failed or canceled)
  - `ls -1 | sort -r` can be used to sort backups (most recent to least recent)
- A symbolic link `latest` will always point to the most recent backup

### Breaking Changes
Although this package will attempt to create very few breaking changes, if said changes do occur the [Breaking Changes Notification Thread](https://github.com/mattlyons0/Rsync-Snapshot/issues/20) will be updated. Subscribe to the [thread](https://github.com/mattlyons0/Rsync-Snapshot/issues/20) in order to be notified of breaking changes.

#### Parameters
*Note: To wrap strings double quotes must be used. Ex: `--shell "ssh -p 2222"` must be used to specify ssh parameters. Single quotes will not be parsed correctly.*
##### Rsync
- `--src PATH` *Default:* `/*`
  - Source path to backup
- `--dst PATH`
  - Destination folder path for backup
  - If using `--shell ssh` format is `username@server:destinationPath`
  - Folders will be created in this directory for incremental backup history
- `--shell SHELL`
  - Remote shell to use
  - *Note: Remote shell is assumed to be a ssh compatible client if specified*
    - Ex: `ssh` or `"ssh -p 2222"`
- `--exclude PATH` *Can be used multiple times*
  - *Note: Unless `--excludeFile` is set [default exclude list](https://github.com/mattlyons0/Rsync-Snapshot/blob/master/data/defaultExclude.txt) will be used in addition to specified excludes*
  - Syntax
    - Include empty folder in destination: `/dev/*`
    - Do not include folder in destination: `/dev`
    - Glob style syntax: `*/steam/steamapps` (Will exclude any file/folder ending with /steam/steamapps)
    - [See Filter Rules](https://linux.die.net/man/1/rsync) for more information
- `--excludeFile EXCLUDEFILE` *Default:  [defaultExclude.txt](https://github.com/mattlyons0/Rsync-Snapshot/blob/master/data/defaultExclude.txt)*
  - Similar to `--exclude` but is passed a text file with an exclude rule per line
  - For exclude rule syntax see `--exclude` documentation
- `--checksum`
  - Change default transfer criteria from comparing modification date and file size to just comparing file size
  - This means the file size being the same is the only requirement needed to generate a checksum and transfer potential file differences
    - Enabling this flag will incur a performance penalty as many more checksums may be generated
- `--accurateProgress`
  - Recurse all directories before transferring any files to generate a more accurate file tree
  - *Note: This will increase memory usage substantially (10x increase is possible)*
- `--noDelete`
  - Don't delete existing files in `--dst`
- `--noDeleteExcludes`
  - Don't delete existing files in `--dst` that are excluded
  - Can be useful for restores where excluded files are hardware specific
- `--rsyncPath PATH` *Defaults to* `"sudo rsync"`
  - Command to execute rsync
  - If using SSH `"sudo rsync"` is recommended however it requires additional setup as a password prompt can not be asked (/etc/sudoers file must be modified to set NOPASSWD for rsync, see [Rsync over ssh without root](https://unix.stackexchange.com/questions/92123/rsync-all-files-of-remote-machine-over-ssh-without-root-user/92125#92125))
- `--setRsyncArg ARGUMENT=VALUE` *Can be used multiple times*
  - Specify an argument to be passed to rsync and (optionally) its value
  - Ex: `--setRsyncArgument checksum` or `--setRsyncArgument block-size=1024`
- `--unsetRsyncArg ARGUMENT` *Can be used multiple times*
  - Unset an argument which was already passed to rsync
##### Snapshot Management
- `--maxSnapshots NUMBER`
  - Maximum number of snapshots
  - Once number is exceeded, oldest snapshots will be deleted until the condition is met
##### Script Hooks
  Script Hooks can be used to run scripts before or after backup on the client while using the same log file as the backup process. Script hooks are not run in parallel.
- `--runBefore EXECUTABLE` *Can be used multiple times*
  - Script to run on client before backup (file will be executed directly and output will be logged)
  - Can be useful for taking backups of data that requires consistency (ex: running pg_dump) and putting it in a folder that will be transfered by Rsync in the backup
- `--runAfter EXECUTABLE` *Can be used multiple times*
  - Script to run on client after backup
  - Hook will only trigger if backup is successful
  - Can be useful for deleting temporary data after it is successfully transferred
##### Logging
- `--logFormat FORMAT` *Default:* `text`
  - Format used to log output
  - Supported formats:
    - `json` - Rsync process output in JSON format
    - `text` - An easy to read rsync process output
    - `raw` - Output directly from rsync process
- `--logFile PATH`
  - Path to file used to write output in `logFormat`
  - If file already exists it will be appended, otherwise it will be created
- `--logFileLevel LEVEL` *Default:* `ALL`
  - Level of output to write to log file
  - Supported levels:
    - `ALL` Log Progress, Warnings, Errors and Summary
    - `WARN` Log Warnings, Errors and Summary
    - `ERROR` Log Errors and Summary
##### Restore
  - `--restore`
    - Clone files from `--src` to `--dst`
      - *Note: Since there is no snapshot management done in restore mode, restores can be done from server to client or client to server by switching the destination and source arguments*
    - If this flag is used snapshots are not used, this flag enables a simple rsync copy from the source to destination
      - All snapshot management flags will be ignored
##### Debug Info
  - `--version`
    - Print package version and exit
  - `--printCommand`
    - Print rsync command used

### Data Consistency & Integrity
Rsync Does **NOT** Ensure Consistency | Rsync **May** Ensure Integrity
- Consistency
  - Rsync can not take snapshots like certain file systems (ZFS, LVM...) this means **if there are changes to files between this script starting and finishing the files could have been copied in any state**
  - Rsync first builds a list of files then transfers only the deltas of each file from the client to the server
  - This means that files created after rsync has built a list of files will not be transfered
  - Because consistency is not ensured, **this backup solution is not sufficient for database backups**
    - It is recommended that database dumps are taken using database specific technology (ex: pg_dump for postgres)
    - The same applies to any write heavy application
- Integrity
  - Rsync **will always ensure transferred files are correctly reconstructed** in memory
  - Rsync will then write the data in memory to the disk
  - If the OS indicates a successful write, rsync will proceed
    - There is no checksum done post write to disk as write correctness to be handled by the OS
  - Rsync determines files to be transferred by default by comparing file size and modification date
    - **Checksums are only generated for potentially transferred files**
    - The criteria to potentially transfer files can be changed to comparing file size only using the `--checksum` flag

### Common Warnings/Errors
- `rsync warning: some files vanished before they could be transferred (code 24)`
- `file has vanished`
  - These warnings indicate a file has been deleted between the time rsync started and stopped executing
  - This does not mean the backup has failed, it is an expected warning as rsync does not take system level snapshots and data will not always be consistent
    - This message should be used a warning that said file may need to be backed up using a different method in order to ensure its consistency
    - See [Data Consistency & Integrity section](https://github.com/mattlyons0/Rsync-Snapshot#data-consistency--integrity) for more information
- `opendir failed: Permission Denied`
- `send_files failed to open: Permission Denied`
- Any other permission related error
  - These errors indicate there is an issue with the permissions rsync is being run with.
- `An error occurred connecting to server while preparing for backup: sudo: no tty present and no askpass program specified`
  - The backup snapshot management needs access to `sudo rm` and `sudo mv` without a password. If this error occurs the sudoers file (on the server) needs to be modified to allow `rm` and `mv` without a password.
- `An error occurred connecting to server whiel preparing for backup: mkdir: cannot create directory: Permission denied`
  - The backup user does not have permission to create directories in `--dst`

### Recovery
- Partial Recovery
  - Since the backups are not compressed partial recovery is as easy as using SFTP (Filezilla works great if you want a GUI) and copying files over from the desired dated snapshot
    - *Note: SFTP does not preserve all file attributes, if this is desired it is recommended to write a rsync script to transfer files using rsync parameters found in this script*
      - At some point in the future I could make a flag for restores, if this would be useful to you feel free to open an issue
- Complete Recovery
  - Recovering an entire installation is very similar to partial recovery
  - You will need to boot on a Live CD with access to networking, install rsync and then use it to rsync the files to the desired partition(s) (of course you will have to make the partition(s) first if they don't already exist)
  - Note that you may have to update things like /etc/fstab if disk names have changed or regenerate the bootloader
  - For more details see [Recovering entire systems from backups](http://www.sanitarium.net/golug/rsync_backups_2010.html)
  - If the server with backups has ssh access to the client, see `--restore`

### Additional Resources
  - [Do It Yourself Backup System Using Rsync](http://www.sanitarium.net/golug/rsync_backups_2010.html) - Tons of useful information and what this script is based on
    - [Snapshot Diff Script](http://www.sanitarium.net/unix_stuff/Kevin%27s%20Rsync%20Backups/diff_backup.pl.txt) Script to find differences between two rsync snapshots
    - [Partition Table Backup](http://www.sanitarium.net/unix_stuff/Kevin%27s%20Rsync%20Backups/getinfo.pl.txt) Script to backup the current partition table schema
  - [Rsync Snapshot Backup - Arch Wiki](https://wiki.archlinux.org/index.php/rsync#Snapshot_backup) Arch Linux Wiki Page on Rsync and using snapshots
  - [Rsync over SSH without root](https://unix.stackexchange.com/questions/92123/rsync-all-files-of-remote-machine-over-ssh-without-root-user/92125#92125) - A good setup for the permissions model required
