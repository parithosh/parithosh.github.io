# Backups with BorgBackup

Backups are an extremely important part of any system. You never need them until you really do. The golden rule 
for backups is called the `3-2-1 rule`, essentially 3 copies of the data on 2 different media and 1 offsite location. Ideally 
the backups are automated and run at some fixed interval.   

Data de-duplication refers to a method to eliminate storing duplicate copies of repeating data. This is extremely advantageous
if we want to save space on our backup machine for our regular backups. Since the offsite location might not be fully under
our control we would also want to encrypt our backups as well as compress it to save space.  

So essentially we are looking for a backup tool that:  
    - is space efficient
    - supports encryption
    - supports compression
    - run from the cli/scriptable
    - performs de-duplication
    - available on multiple platforms

Enter BorgBackup(short Borg), a client-server based backup tool that fits all of our criteria! Let's set it up and see how it works.   

## Installation  
For the purpose of this post, I will assume the host and target machine is a linux based system (Ubuntu/Debian ideally).   

Borg is part of the official ubuntu/debian repos, so you can install it with `sudo apt install borgbackup`  

## Configuration
Borgbackup can be run to backup locally or to a mounted volume or to backup remotely. This config will focus on remote backups.   

- Initialize the backup repository on your `backup server` with `borg init --encryption=repokey </path/to/repo>`. Borg will setup an
empty backup repository that is ready for future backups. Please note the path, as that is important information. Enter
  an encryption password when prompted, this is your encryption key and is needed for each future backup or restore.  
- Now shift to your host machine and get your SSH public key, typically `cat ~/.ssh/id_rsa.pub` and add this key to the 
  backup machine's file named `~/.ssh/authorized_keys`. This would allow your host machine to SSH into your target machine
  without an SSH password.    
- Test the SSH connection between the two with `ssh username@backup-ip-address`. Additionally, you can create a dedicated
backup user on the backup machine for a tighter control.  
- We can create a test backup to check if everything works as expected with this command: `borg create -v --stats username@backup-ip-address:</path/to/repo>::BACKUP_NAME </path/to/something/to/backup>`.  
- Now run `borg list username@backup-ip-address:</path/to/repo>`, it should list the backup you just created.  
- We can start scripting this process now, create a file called `borgbackup.sh`:  
Note: I like to have my backups with unique names, so I append the backup name with the current time.   
  
```shell
#!/bin/bash
# Backup a folder to a remote address using borg.
set -eu

echo "Successfully set the time"
TIME=$(date +%s)

# export the borg passphrase as an environment variable to avoid having to enter it each time
export BORG_PASSPHRASE='blablabla' 
echo "Backing up local to remote server"

# Backing up xyz content
borg create --progress --stats username@backup-ip-address:</path/to/repo>::BACKUP_NAME-$TIME </path/to/something/to/backup>

# Prune old backups
borg prune username@backup-ip-address:</path/to/repo> --keep-weekly=1 --keep-monthly=2
```  
- Make the script executable with `chmod +x borgbackup.sh` and run the script to test it. Run `borg list username@backup-ip-address:</path/to/repo>`
to confirm.   
- Let's make the script run periodically now, we will do this using `crontab`. Run `crontab -e` and configure the 
frequency and script, e.g: `0 3 * * * /path/to/borgbackup.sh` to run the script at 3AM every morning. Check a crontab guide
  to figure out how to modify the time.  
- Run `crontab -l` to confirm the crontab has been setup.  
  
## Conclusion

Have fun with your nightly backups! I will try to write up a guide for setting up a wireguard system, so that you can roll
your own private VPN and move your backup server far away, yet private.  
