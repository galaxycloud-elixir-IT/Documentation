The encryption layer
====================

Device mapper is the Linux kernel driver for volume management and provides transparent encryption of devices through the Linux kernel crypto API, using its device mapper crypt (dm-crypt) module. Dm-crypt is commonly used through Cryptsetup [cryptsetup], a command line interface to dm-crypt, allowing user to setup a new encrypted block device in /dev, specifying the encryption mode, the cipher and the key. Then the device can be formatted with a file system (e.g. ext4), mounted like any other partition and used as persistent storage.

Cryptsetup supports different encryption modes, like plain dm-crypt [cryptsetup] and LUKS volumes [LUKS_web, LUKS_spec] already included in the Linux kernel, but also Loop-AES [loopaes] and TrueCrypt/VeraCrypt [vera] requiring extra modules installation.

We restricted our choice to dm-crypt usage, which exploits Linux kernel built-in APIs, avoiding the installation of any additional external package other than cryptsetup. In particular, the LUKS encryption grants better usability and flexibility to end users without neglecting data security. Unlike others encryption modes, LUKS stores all dm-crypt setup information in the partition header at the beginning of the block device itself, allowing for multiple passphrases that can be changed and/or revoked anytime. It provides robustness against low-entropy passphrases attack using salting and iterated PBKDF2 passphrase hashing.

Cryptsetup allows for different ciphers usage. A cipher consists of three parts: a block cipher, i.e. it is the encryption algorithm, which operate on fixed-length blocks of data; a block cipher mode of operation, which describes how to repeatedly apply a cipher single block operation to data larger than cipher block size and an Initialization Vector (IV) generator, used to randomize the output of the encryption algorithm, ensuring that the same data are encrypted differently with the same key.

LUKS default cipher is aes-xts-plain64, i.e. AES as block cipher, XTS as mode of operation and plain64 as IV generator. The Advanced Encryption Standard (AES) [AES] is a symmetric-key algorithm, I.e. the same key is used either to encrypt and decrypt data, applying several substitution and permutation rounds to plaintext block to produce encrypted blocks. The Xor encrypt xor Tweakable block Cipher (XTS) mode of operation [XTS1, XTS2] is intended specifically to encrypt data on a block-structured storage device, e.g. disk sectors. The mode works with AES as underlying block cipher which is applied two times to each data chunk: the plain text block is combined with the tweak value, i.e. the plain64 IV, encrypted with AES. Then the block is AES encrypted with the key. Finally, the result is combined again with the tweak value before storing the cipher block.

These options represent the current standard on storage encryption and their modification is strongly discouraged, unless user requires particular configurations. For this reason, even if the Laniakea encryption layer can in theory accept user-defined configuration, e.g. different ciphers, we did not expose these options in the user-interface.





While the adoption of a distributed environment for data analysis makes data difficult to be tracked and identified by a malevolus attacker, full data anonymity and isolation is still not granted.

For this reason, we have introduced the possibility for the user to isolate data through standard filesystem encryption, using the Linux device mapper encryption module, dm-crypt as encryption backend, cryptsetup, which is a command line interface to dm-crypt and LUKS (Linux Unified Key Setup) as encryption mode.

Disk encryption ensures that files are stored on disk in an encrypted form: the files only become available to the operating system and applications in readable when the volume is unlocked by a trusted user. The adopted block device encryption method, operates below the filesystem layer and ensures that everything is written to the block device (i.e. the external volume) is encrypted. 

Dm-crypt is the standard device-mapper encryption functionality provided by the Linux kernel. Dm-crypt management is entrusted to the cryptsetup userspace utility, using LUKS as default block-device encryption method. LUKS, is an additional layer which stores all the needed setup information for dm-ctypt on the disk itself, abstracts partition and key management in an attempt to improve ease of use and cryptographic security

To automatically setup LUKS volumes on your Galaxy instances a bash script, named `fast-luks <https://github.com/mtangaro/GalaxyCloud/blob/master/LUKS/fast_luks.sh>`_ has been created. The script has been integrated in the Galaxy instantiation procedure: if the File System Encryption option is selected through the dialogue window the users will be required to insert a password to encrypt/decrypt data on the virtual instance during its deployment, avoiding any interaction with the cloud administrator(s). For more details see: `Fast-luks script`_.

.. Note::

   During the ancryption procedure an e-mail is sent to the Galaxy Instance Administrator, with the Virtual Machine IP address and a detailed description of the procedure to inject the encryption password in the system. The Galaxy installation procedure is paused until the password is correctly set (after 5 hours the system will return an error).

.. Warning::

   The system does not store your keys on its servers and cannot access your protected data unless you provide the key. This also means that if you forget or lose your key, there is no way to recover the key or to recover any data encrypted with the lost key.

Default configuration
---------------------
Fast-luks, by default adopt ``xts-aes-plain64`` cipher with ``256`` bit keys ans ``sha256`` hashing algorithm.

Once the LUKS partition is created, it is unlocked.

The unlocking process will map the partition to a new device name using the device mapper. This alerts the kernel that device is actually an encrypted device and should be addressed through LUKS using the ``/dev/mapper/<cryptdev_name>`` so as not to overwrite the encrypted data. ``cryptdev_name`` is random generated to avoid accidental overwriting.

The volume is mounted, by default, on ``/export``, with standard ``ext4`` filesystem and Galaxy is configured to store here datasets.

Defaults values
***************

::

  # Defaults
  cipher_algorithm='aes-xts-plain64'
  keysize='256'
  hash_algorithm='sha256'
  device='/dev/vdb'
  cryptdev='crypt'
  mountpoint='/export'
  filesystem='ext4'

Password choice
---------------
During the encryption procedure ( :doc:`FS_encryption_procedure`), users are required to configure their encryption password. You must provide at least an alphanumeric 8-character long password.
A random generated password is generated by the script, ONLY FOR EXAMPLE!

During the encryption procedure, the password will be required three times.

The script will query for passwords twice, to verify it. Then, you have to unlock your device, by inserting your password.

The procedure is described here: ( :doc:`FS_encryption_procedure`).

Luksctl
-------
During the encryption procedure ( :doc:`FS_encryption_procedure`), ``fast-luks`` creates a configuration ini file, allowing Galaxy administrator to easily mange LUKS devices, through the ``luksctl`` script (:doc:`script_luksctl`).

By default the file is stored in ``/etc/galaxy/luks-cryptdev.ini``.

The script requires superuser rights.

========  ======================  =========================
Action    Command		  Description
========  ======================  =========================
Open      sudo luksctl open	  Open the encrypted device, requiring your passphrase.
Close     sudo luksctl close      Close the encrypted device
Status    sudo luksctl status     Check device status using dm-setup
========  ======================  =========================

Galaxyctl can be used to parse luksctl commands:

=====================  ==============================  =========================
Action                 Command			       Description
=====================  ==============================  =========================
Open LUKS volume       sudo galaxyctl open luks        Open the encrypted device, requiring your passphrase.
Close LUKS volume      sudo galaxyctl close luks       Close the encrypted device
Check LUKS volume      sudo galaxyctl status luks      Check device status using dm-setup
=====================  ==============================  =========================

.. _luks_anchor:

Fast-luks script
----------------
The ``fast-luks`` script is located in ``/usr/local/bin/fast-luks``.

It parse common cryptsetup parameters to encrypt the volume. For this reason it checks for cryptsetup and dm-setup packages and it install cryptsetup, if not installed.

Typing ``sudo fast-luks`` the script will load defaults parameters and will LUKS format ``/dev/vdb`` device, otherwise different parameters can be specified.

NB: Run as root.

===============================  =====================================  ============================================
Argument	                 Defaults                               Description
===============================  =====================================  ============================================
``-c``, ``--cipher``             aes-xts-plain64                        Set cipher specification string.
``-k``, ``--keysize``            256					Set key size in bits.
``-a``, ``--hash_algorithm``     sha256                                 For luksFormat action specifies hash used\
                                 					in LUKS key setup scheme and volume\
 				 					key digest.
``-d``, ``--device``             /dev/vdb				Set device to be mounted
``-e``, ``--cryptdev``           crypt                                  Sets up a mapping <name> after successful\
									verification of the supplied key\
									(via prompting).
``-m``, ``--mountpoint``         /export 				Set mount point
``-f``, ``--filesystem``         ext4					Set filesystem
===============================  =====================================  ============================================

::

  $ sudo fast-luks --help
  =========================================================
                        ELIXIR-Italy
                 Filesystem encryption script

  A password with at least 8 alphanumeric string is needed
  There's no way to recover your password.
  Example (automatic random generated passphrase):
                        PcHhaWx4

  You will be required to insert your password 3 times:
    1. Enter passphrase
    2. Verify passphrase
    3. Unlock your volume

  The connection will be  automatically closed.

  =========================================================

  fast-luks: a bash script to automate LUKS file system encryption.
   usage: fast-luks [-h]

   optionals argumets:
   -h, --help 		show this help text
   -c, --cipher 		set cipher algorithm [default: aes-xts-plain64]
   -k, --keysize 		set key size [default: 256]
   -a, --hash_algorithm 	set hash algorithm used for key derivation
   -d, --device 		set device [default: /dev/vdb]
   -e, --cryptdev	 	set crypt device [default: cryptdev]
   -m, --mountpoint 		set mount point [default: /export]
   -f, --filesystem 		set filesystem [default: ext4]
   --default 			load default values

Cryptsetup howto
----------------

The cryptsetup action to set up a new dm-crypt device in LUKS encryption mode is luksFormat:

::

  cryptsetup -v --cipher aes-xts-plain64 --key-size 256 --hash sha 256 --iter-time 2000 --use-urandom --verify-passphrase luksFormat crypt --batch-mode

where ``crypt`` is the new device located to ``/dev/mapper/crypt``.

To open and mount to ``/export``  an encrypted device:

::

  cryptsetup luksOpen /dev/vdb crypt

  mount /dev/mapper/crypt /export

To show LUKS device info:

::

  dmsetup info /dev/mapper/crypt

To umount and close an encrypted device:

::

  umount /export

  cryptsetup close crypt

To force LUKS volume removal:

::

  dmsetup remove /dev/mapper/crypt

..Note::

NB: Run as root.

Change LUKS password
********************

LUKS provides 8 slots for passwords or key files. First, check, which of them are used:

::

  cryptsetup luksDump /dev/<device> | grep Slot

where the output, for example, looks like:

::

  Key Slot 0: ENABLED
  Key Slot 1: DISABLED
  Key Slot 2: DISABLED
  Key Slot 3: DISABLED
  Key Slot 4: DISABLED
  Key Slot 5: DISABLED
  Key Slot 6: DISABLED
  Key Slot 7: DISABLED

Then you can add, change or delete chosen keys:

::

  cryptsetup luksAddKey /dev/<device> (/path/to/<additionalkeyfile>) 

  cryptsetup luksChangeKey /dev/<device> -S 6

As for deleting keys, you have 2 options:

#. delete any key that matches your entered password:

   ::

     cryptsetup luksRemoveKey /dev/<device>

#. delete a key in specified slot:

   ::

     cryptsetup luksKillSlot /dev/<device> 6

References
----------

Disk encryption archlinux wiki page: https://wiki.archlinux.org/index.php/disk_encryption#Block_device_encryption_specific

Dm-crypt archlinux wiki page: https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode

Original LUKS script: https://github.com/JohnTroony/LUKS-OPs/blob/master/luks-ops.sh (Credits to John Troon for the original script))

LUKS: https://guardianproject.info/code/luks/

LUKS how-to: http://www.thegeekstuff.com/2016/03/cryptsetup-lukskey