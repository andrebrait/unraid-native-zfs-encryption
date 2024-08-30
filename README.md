# Why use native ZFS encryption?

unRAID already supports Full Disk Encryption (FDE) through LUKS in combination with ZFS pools. 
The support is solid and LUKS is a widely used tool with great performance. So, why use native ZFS
encryption then? Here are a few pros and cons, starting with the cons:

### Cons

* LUKS is transparent to the filesystem and does not depend on any filesystem-specific tooling and it's 
easy to find help on the internet, should you need it.
* LUKS is fully supported by unRAID, meaning the GUI and the UI flows are thought out with it in mind.
    * We will actually tap into this support in this document in order to bring a "similar" level of support
    for ZFS encryption, _but it is still not officially supported_, meaning unRAID might change something 
    in the future. It's unlikely, but it _could_ happen.
* ZFS encryption does not encrypt empty disk blocks, nor file metadata. LUKS encrypts everything, including 
file metadata and the filesystem structured themselves
    * If absolute secrecy is your priority and you're willing to manage encryption on your backups as well,
    you might want to go with LUKS instead.
    * However, ZFS encryption is not weak. Keep reading!

### Pros

* Unlike LUKS, ZFS encryption can be turned on and tweaked at the _dataset_ level, meaning for a single _pool_
you can have a mix of encrypted and regular datasets.
    * That means you can have both _secrecy_ for your data **and** _speed_ for your applications
    * e.g. place an app's _data_ in an encrypted dataset and its _configuration files_, _database_ 
    or _log files_ in a regular dataset
* ZFS replication does **_not_** require decryption!
    * This is a major advantage. Not only can you do `zfs send` without loading the decryption key to your dataset,
    but you can also replicate your dataset in another host with complete secrecy. The remote host does not need to 
    have your decryption key, nor does it need to receive the data decrypted for it to then encrypt again
* All other filesystem features are still available and work as expected.
    * TRIM, compression and other features are all available with compression with no limitations or further loss of
    security implications.
        * Yes, LUKS does support TRIM as well, but it's disabled by default and it has security implications due to free 
        space not being encrypted, defeating one of the advantages of FDE over ZFS's native encryption.

# The state of support in unRAID

As of now (August 2024) unRAID lacks GUI support for ZFS encryption. The _ZFS Master_ plugin does so, but it requires
the user to interact with it in order to unlock the dataset (via _password_ or _key file_). That means the encrypted 
dataset would only be available for unlocking after the array is started and it would take a while to do so, making 
it unusable for Docker mounts or VMs as those services start right after the array, if enabled.

# The workaround

ZFS, like LUKS, can store the decryption key in a _key file_. Normally, that's a complicated subject because it means 
you need some form of storage device (commonly a USB thumb drive or a network share) that is accessible at the time 
of mounting the dataset, but ideally not permanently attached to your unRAID machine (as it defeats the purpose of
encryption) nor too easy to get your hands on, should someone steal the machine. For this reason, many people would 
prefer to rely on a password to decrypt the drives at startup, which does not decrypt any ZFS datasets using native 
encryption by default.

Auto-mounting such ZFS datasets is possible, but it would require a key file, which leads us back to the "where do I store it?" problem.

Well, what if we store it in a LUKS-encrypted drive that gets auto-mounted when the Array starts and then we auto-mount
the dataset after that, but before anything starts? That way we get the GUI support, ensure the dataset is usable for Docker/VMs
and we never have to worry about the key file because it is itself protected by our drive encrypted using our password!

And that is exactly what we'll be doing here!

`Password -> LUKS -> ZFS encryption key -> ZFS encrypted dataset auto-mounted`

# Requirements

This tutorial assumes you have:

1. **_At least one_** of the following:
    1. An encrypted unRAID Array
    2. An encrypted pool with another filesystem (e.g. btrfs or XFS)
    3. No unRAID array (ver. 7 or higher) + a spare drive
    4. A dummy array (ver. 6.12 or lower)
2. A ZFS pool
3. Some experience with the command-line
4. unRAID 6.12 or later
5. ZFS Master plugin
6. User Scripts plugin

# Recommendations

These are not requirements, but are highly recommended:

1. Enable "Exclusive Shares"
    1. Stop the array
    2. Go to Settings -> Global Share Settings 
    3. Set "Permit exclusive shares" to "Yes"

# Tutorial

In this tutorial, we will:

1. Encrypt a dummy array to store the key for the ZFS dataset
2. Create an encrypted dataset: `appdata-crypt`
3. Create a script to automatically mounts it after the array starts
4. Move some parts of Immich, Nextcloud and Paperless-ngx to the new encrypted dataset

## Setup

>_If you already have an encrypted **array** or **pool**, skip this section._

_This section assumes you either have **no array** or that you have a **dummy array**._

The setup I have is composed of:
* A dummy array
* Three ZFS Pools
    * `data-pool` for media storage
    * `cache-pool` as a cache for `data-pool` 
    * `apps-pool` for `appdata`, `domain`, `system`, etc.

![setup](https://github.com/user-attachments/assets/a32af72c-9c3b-47f7-bdb2-2d0c35a844f2)

![setup-2](https://github.com/user-attachments/assets/29c3c68d-0bf4-4aea-be21-17f8c79bb6b5)

## Encrypting an Array or Pool

In order to encrypt the array, first stop it

![array-stop](https://github.com/user-attachments/assets/a4c09e0a-610b-42b6-868f-24024e7b59ec)


Then select your array device

![array-device-list](https://github.com/user-attachments/assets/4774e7b8-a1d5-440c-b6ad-fd54e7482d77)


Choose any of the encrypted filesystem options. In this example, we'll be using XFS for its stability.

![filesystems](https://github.com/user-attachments/assets/5066c2b7-9071-444c-afac-044b0e8720df)


Observe that the array device now has a lock icon

![array-device-list-pre-format](https://github.com/user-attachments/assets/91ff2559-9561-4291-8bf7-10bac3dcdb69)


Scroll down to where the Array Start button is. Type your password of choice and start the array.

![password](https://github.com/user-attachments/assets/a9a7ab19-c838-4b5e-8b55-f72db7ea1aae)


The array is started, but the device is not yet encrypted! It must be formatted first.

![unformatted](https://github.com/user-attachments/assets/2750b4fd-c1d7-4060-9e0b-2d76eda7913a)


Scroll down, tick the `Yes, I want to do that` box and format it.

![formatting](https://github.com/user-attachments/assets/e805d429-5908-41c7-80b2-a5b6978412e4)


Click OK to format the device. _THIS WILL ERASE ALL DATA ON IT!_

![formatting-warn](https://github.com/user-attachments/assets/43da7def-66f7-4b6a-9139-d796ff812a96)


The device is now correctly formatted and unlocked

![formatted](https://github.com/user-attachments/assets/a364a3c6-31aa-43ff-a0aa-f232774e9f65)

## Creating the encrypted ZFS dataset


First, let's find out the path for the encrypted drive we just created by running `ls /mnt`.\
In this case it's `/mnt/disk1`.

![ls-mnt](https://github.com/user-attachments/assets/953486fe-601e-4c95-ba05-8f2766341bb3)


Now let's create the directory where we will store the key file for the encrypted ZFS dataset inside the encrypted disk.

```shell
# Let's set the key directory in this variable so we can use it further down the line
KEY_DIR="/mnt/disk1/.zfs-crypt-keys"
mkdir -p "${KEY_DIR}"
```


Then create the key file and the dataset itself.\
Note that the dataset "knows" where its keyfile is and what its mountpoint is.\
You don't have to remeber that in the future.

Please note you must choose *only one of the formats* for creating and keyfile/dataset: `raw` or `hex`.\
I find `hex` to be the best one because it's human-readable and you can then back it up to e.g. a password manager.

```shell
# Set these so we can use them down below
KEY_FILE="${KEY_DIR}/appdata-crypt.key"
DATASET_LOCATION="apps-pool/appdata-crypt"

# SELECT ONLY ONE OF THE FORMATS BELOW

# Raw (notice different keyformat)
# dd if=/dev/random of="${KEY_FILE}" bs=32 count=1 iflag=fullblock
# zfs create -o encryption=on -o keyformat=raw -o keylocation="file://${KEY_FILE}" "${DATASET_LOCATION}"

# Hex (notice different keyformat)
od -Anone -x -N 32 -w64 /dev/random | tr -d [:blank:] > "${KEY_FILE}"
zfs create -o encryption=on -o keyformat=hex -o keylocation="file://${KEY_FILE}" "${DATASET_LOCATION}"

# Assuming you've used the 'hex' format
# Backup your key file
cat "${KEY_FILE}"

# EXAMPLE OUTPUT. WRITE YOURS DOWN SOMEWHERE!
# 316e548796d2307290353d94676269a629ebc9b14dcd5baf8910467b35c3f199
```


The dataset should now be visible in `Main -> ZFS Master`!

![dataset-in-zfs-master](https://github.com/user-attachments/assets/531430d6-902a-4ac5-8305-07b4ffc16f26)


Head over to `Shares` and set the newly crated `appdata-crypt` to be stored _only_ in the pool containing it

![make-exclusive-share](https://github.com/user-attachments/assets/337bc5fd-f2cf-4ca4-b4aa-3297008bf70e)


Now head back to `Main -> ZFS Master` and lock the dataset.

![lock-dataset](https://github.com/user-attachments/assets/32758f9b-be02-4a28-9b0a-8ee78f0fa136)


The icon changes to a locked drive.

![locked-dataset](https://github.com/user-attachments/assets/43c3549d-5feb-4497-8c02-e70b70eb2b21)

Back to the command line, you'll noticed that `/mnt/apps-pool/appdata-crypt` exists even though the dataset is locked.\
To prevent any unintended files from being created there while the array does not start, let's make the directory immutable.

```shell
# Make the mount point immutable. Not even root can modify it!
chattr +i "/mnt/${DATASET_LOCATION}"
```

Back to `Main -> ZFS Master`, try unlocking the dataset.

![locked-dataset](https://github.com/user-attachments/assets/de724923-6e19-47ee-a102-26283b5aeffd)


If it prompts you for a password, do not enter anything and just click OK.

![password-prompt](https://github.com/user-attachments/assets/05de2494-c065-45a1-8310-273f989f2ef6)


You should see this screen reporting it successfully unlocked the dataset!

![dataset-unlocked](https://github.com/user-attachments/assets/88d0547d-a7ed-47e5-9fed-ada4f4d24654)


Now that we know everthing is working, let's head back to the terminal and reduce the chances this key gets leaked.

```shell
# Set the key location to only be readably by the 'root' user.
chown -R root:root "${KEY_DIR}"
chmod -R 500 "${KEY_DIR}"

# Set key directory and files to be immutable so it's absolutely impossible to be chaned
chattr -R +i "${KEY_DIR}"
```

## Setting up the startup script

In order to have the encrypted dataset be unlocked at the same time the Array is started, 
head over to the `User Scripts` plugin and create a new script.

Ensure its schedule is set to `At Startup of Array`, like so:

![user-scripts](https://github.com/user-attachments/assets/23e0f5d8-3922-49a2-9cbc-77637e91147c)


Edit the script's contents and paste the following:

```shell
#!/bin/bash

# Set this to 'yes' in order to receive notifications of success or failure to mount the encrypted dataset
NOTIFY="no"

# Discover all encrypted datasets we have, skipping over the prompt key locations
declare -a encrypted_datasets=($(zfs list -r -t filesystem -H -o name,keylocation | grep -F '/' | grep -vE '\b(none|prompt)\b' | cut -d$'\t' -f1))

curr_date() {
    date +'%Y-%m-%d %H:%M:%S'
}

unraid_notify() {
    if [[ "${NOTIFY}" != "yes" ]]; then
        return
    fi
    
    local message="$1"
    local flag="$2"
    
    if [[ "$flag" == "success" ]]; then
        severity="normal"
    else
        severity="warning"
    fi
    
    /usr/local/emhttp/webGui/scripts/notify -s "Backup Notification" -d "$message" -i "$severity"
}

echo "[$(curr_date)] Detected encrypted datasets: ${encrypted_datasets[@]}"

for d in "${encrypted_datasets[@]}"; do
    
    echo "[$(curr_date)] Loading key for: ${d}"
    out=$(zfs load-key -r "${d}" 2>&1)
    
    if (($? == 0)); then
        echo "[$(curr_date)] Loaded key for ${d}"
        unraid_notify "Loaded key for ZFS dataset ${d}" "success"
    else
        echo "[$(curr_date)] Failed to load key for ${d}: ${out}"
        unraid_notify "Failed to load key for ZFS dataset ${d}: ${out}" "failure"
        exit 1
    fi
    
    mountpoint="$(zfs list -H -o mountpoint ${d})"
    
    # Ensure the regular location is immutable so nothing can write to it unless we're mounted!
    out="$(mkdir -p "${mountpoint}" 2>&1 && chattr +i "${mountpoint}" 2>&1)"
    if (($? > 0)); then
        echo "[$(curr_date)] Failed to create directory for ${d}: ${out}"
        unraid_notify "Failed to create directory for ${d}: ${out}" "failure"
        exit 1
    fi
    
    
    echo "[$(curr_date)] Mounting ${d} at ${mountpoint}"
    out=$(zfs mount "${d}" 2>&1)
    if (($? == 0)); then
        echo "[$(curr_date)] Mounted ${d}"
        unraid_notify "Mounted ZFS dataset ${d}" "success"
    else
        echo "[$(curr_date)] Failed to mount ${d}: ${out}"
        unraid_notify "Failed to mount ZFS dataset ${d}: ${out}" "failure"
        exit 1
    fi
done
```

## Testing the script

Go to `Main -> ZFS Master` and lock the dataset again

![zfs-master-lock](https://github.com/user-attachments/assets/14c57a16-9240-4f1f-b9cf-0dfaac101833)


Check that it displays the locked icon

![locked-dataset](https://github.com/user-attachments/assets/384ebbca-db22-444b-9f69-9cd4eea45293)


Head back to `User Scripts` and run the script

![usr-scripts](https://github.com/user-attachments/assets/faa084e7-6051-4e3e-9b58-e96d79235970)


You should see the output of the script, which should report success.

![script-result](https://github.com/user-attachments/assets/c4f2d56d-4ba3-4cef-9605-dbfca42c6549)


The script has an option to enable notifications. If set to `yes`, you'll also receive the results like so:

![notifications](https://github.com/user-attachments/assets/3c81d9aa-13bb-40d3-8ae3-14b120f736e3)


As a final test for the script, try stopping and starting the array again. 
It should mount the encrypted dataset right away, whe the Array starts.

## Done!

That's it! From now on, you have an encrypted ZFS dataset in your unRAID server!\
Whenever the array stops, the dataset is automatically unmounted.\
When it starts, you'll be asked to fill in the password for the encryted array, just like always.\
The script will then ensure the dataset is unlocked and mounted before Docker and VMs start up.


# Moving confidential parts of an application

```shell
# 7. Identify what to transfer

# nextcloud: mount some non-encrypted directory (e.g. /mnt/user/appdata/nextcloud/logs) as log directory and add this to php config
#  'log_type' => 'file',
#  'logfile' => '/var/log/nextcloud/nextcloud.log',
#  'log_type_audit' => 'file',
#  'logfile_audit' => '/var/log/nextcloud/audit.log',

# /mnt/user/appdata/immich/photos to /mnt/user/appdata-crypt/immich/photos
# /mnt/user/appdata/nextcloud/data to /mnt/user/appdata-crypt/nextcloud/data
# /mnt/user/appdata/paperless/server/media to /mnt/user/appdata-crypt/paperless/server/media
# /mnt/user/appdata/paperless/server/export to /mnt/user/appdata-crypt/paperless/server/export
# /mnt/user/appdata/paperless/server/consume to /mnt/user/appdata-crypt/paperless/server/consume

OLD_DATASET="apps-pool/appdata"
NEW_DATASET="${DATASET_LOCATION}"

declare -a PATHS_TO_TRANSFER=(
    "immich/photos"
    "nextcloud/data"
    "paperless/server/media"
    "paperless/server/export"
    "paperless/server/consume"
)

for p in "${PATHS_TO_TRANSFER[@]}"; do
    rsync -aRv "/mnt/${OLD_DATASET}/./${p}" "/mnt/${NEW_DATASET}/"
done

# or, to move it (BUT BE CAREFUL):

for p in "${PATHS_TO_TRANSFER[@]}"; do
    rsync -aRv --remove-source-files "/mnt/${OLD_DATASET}/./${p}" "/mnt/${NEW_DATASET}/" && rm -rf "/mnt/${OLD_DATASET}/${p}"
done

# change directories in docker templates and compose, start the apps again, test tha they're ok

# remove each one of the old folders
for p in "${PATHS_TO_TRANSFER[@]}"; do
    rm -rf "/mnt/${OLD_DATASET}/${p}"
done
```

![Screenshot 2024-08-30 042204](https://github.com/user-attachments/assets/789527a3-c4ae-4949-ae5d-f6308aecff29)

![Screenshot 2024-08-30 045840](https://github.com/user-attachments/assets/c8f82a43-b8ff-4a0a-9c7f-cf67a4734809)

![Screenshot 2024-08-30 045907](https://github.com/user-attachments/assets/b2b8be9d-01a9-4698-8a10-7e39d69b3cc7)

![Screenshot 2024-08-30 045937](https://github.com/user-attachments/assets/75aeee45-e339-49ef-a623-4cabd2668f7f)

![Screenshot 2024-08-30 050015](https://github.com/user-attachments/assets/e4e2b334-fd19-49bb-91bc-9782a9276c12)

![Screenshot 2024-08-30 052010](https://github.com/user-attachments/assets/7eb41705-1ab8-4b9c-a893-44a7b59b2925)



