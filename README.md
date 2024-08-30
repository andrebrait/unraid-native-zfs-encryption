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

# Example

In this example, we will:

1. Encrypt a dummy array to store the key for the ZFS dataset
2. Create an encrypted dataset: `appdata-crypt`
3. Create a script to automatically mount it after the array starts
4. Moving Immich, Nextcloud and Paperless-ngx to the new encrypted dataset

# Encrypting your Array/Pool

>_If you already have an encrypted **array** or **pool**, skip this section._

_This section assumes you either have **no array** or that you have a **dummy array**._

The setup I have is composed of:
* A dummy array
* Three ZFS Pools
    * `data-pool` for media storage
    * `cache-pool` as a cache for `data-pool` 
    * `apps-pool` for `appdata`, `domain`, `system`, etc.



`PLACEHOLDER_ENCRYPTION_STEPS`

# Creating the encrypted ZFS dataset

