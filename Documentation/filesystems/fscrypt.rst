=====================================
Filesystem-level encryption (fscrypt)
=====================================

Introduction
============

fscrypt is a library which filesystems can hook into to support
transparent encryption of files and directories.

Note: "fscrypt" in this document refers to the kernel-level portion,
implemented in ``fs/crypto/``, as opposed to the userspace tool
`fscrypt <https://github.com/google/fscrypt>`_.  This document only
covers the kernel-level portion.  For command-line examples of how to
use encryption, see the documentation for the userspace tool `fscrypt
<https://github.com/google/fscrypt>`_.  Also, it is recommended to use
the fscrypt userspace tool, or other existing userspace tools such as
`fscryptctl <https://github.com/google/fscryptctl>`_ or `Android's key
management system
<https://source.android.com/security/encryption/file-based>`_, over
using the kernel's API directly.  Using existing tools reduces the
chance of introducing your own security bugs.  (Nevertheless, for
completeness this documentation covers the kernel's API anyway.)

Unlike dm-crypt, fscrypt operates at the filesystem level rather than
at the block device level.  This allows it to encrypt different files
with different keys and to have unencrypted files on the same
filesystem.  This is useful for multi-user systems where each user's
data-at-rest needs to be cryptographically isolated from the others.
However, except for filenames, fscrypt does not encrypt filesystem
metadata.

Unlike eCryptfs, which is a stacked filesystem, fscrypt is integrated
directly into supported filesystems --- currently ext4, F2FS, and
UBIFS.  This allows encrypted files to be read and written without
caching both the decrypted and encrypted pages in the pagecache,
thereby nearly halving the memory used and bringing it in line with
unencrypted files.  Similarly, half as many dentries and inodes are
needed.  eCryptfs also limits encrypted filenames to 143 bytes,
causing application compatibility issues; fscrypt allows the full 255
bytes (NAME_MAX).  Finally, unlike eCryptfs, the fscrypt API can be
used by unprivileged users, with no need to mount anything.

fscrypt does not support encrypting files in-place.  Instead, it
supports marking an empty directory as encrypted.  Then, after
userspace provides the key, all regular files, directories, and
symbolic links created in that directory tree are transparently
encrypted.

Threat model
============

Offline attacks
---------------

Provided that userspace chooses a strong encryption key, fscrypt
protects the confidentiality of file contents and filenames in the
event of a single point-in-time permanent offline compromise of the
block device content.  fscrypt does not protect the confidentiality of
non-filename metadata, e.g. file sizes, file permissions, file
timestamps, and extended attributes.  Also, the existence and location
of holes (unallocated blocks which logically contain all zeroes) in
files is not protected.

fscrypt is not guaranteed to protect confidentiality or authenticity
if an attacker is able to manipulate the filesystem offline prior to
an authorized user later accessing the filesystem.

Online attacks
--------------

fscrypt (and storage encryption in general) can only provide limited
protection, if any at all, against online attacks.  In detail:

fscrypt is only resistant to side-channel attacks, such as timing or
electromagnetic attacks, to the extent that the underlying Linux
Cryptographic API algorithms are.  If a vulnerable algorithm is used,
such as a table-based implementation of AES, it may be possible for an
attacker to mount a side channel attack against the online system.
Side channel attacks may also be mounted against applications
consuming decrypted data.

After an encryption key has been provided, fscrypt is not designed to
hide the plaintext file contents or filenames from other users on the
same system, regardless of the visibility of the keyring key.
Instead, existing access control mechanisms such as file mode bits,
POSIX ACLs, LSMs, or mount namespaces should be used for this purpose.
Also note that as long as the encryption keys are *anywhere* in
memory, an online attacker can necessarily compromise them by mounting
a physical attack or by exploiting any kernel security vulnerability
which provides an arbitrary memory read primitive.

While it is ostensibly possible to "evict" keys from the system,
recently accessed encrypted files will remain accessible at least
until the filesystem is unmounted or the VFS caches are dropped, e.g.
using ``echo 2 > /proc/sys/vm/drop_caches``.  Even after that, if the
RAM is compromised before being powered off, it will likely still be
possible to recover portions of the plaintext file contents, if not
some of the encryption keys as well.  (Since Linux v4.12, all
in-kernel keys related to fscrypt are sanitized before being freed.
However, userspace would need to do its part as well.)

Currently, fscrypt does not prevent a user from maliciously providing
an incorrect key for another user's existing encrypted files.  A
protection against this is planned.

Key hierarchy
=============

Master Keys
-----------

Each encrypted directory tree is protected by a *master key*.  Master
keys can be up to 64 bytes long, and must be at least as long as the
greater of the key length needed by the contents and filenames
encryption modes being used.  For example, if AES-256-XTS is used for
contents encryption, the master key must be 64 bytes (512 bits).  Note
that the XTS mode is defined to require a key twice as long as that
required by the underlying block cipher.

To "unlock" an encrypted directory tree, userspace must provide the
appropriate master key.  There can be any number of master keys, each
of which protects any number of directory trees on any number of
filesystems.

Userspace should generate master keys either using a cryptographically
secure random number generator, or by using a KDF (Key Derivation
Function).  Note that whenever a KDF is used to "stretch" a
lower-entropy secret such as a passphrase, it is critical that a KDF
designed for this purpose be used, such as scrypt, PBKDF2, or Argon2.

Per-file keys
-------------

Since each master key can protect many files, it is necessary to
"tweak" the encryption of each file so that the same plaintext in two
files doesn't map to the same ciphertext, or vice versa.  In most
cases, fscrypt does this by deriving per-file keys.  When a new
encrypted inode (regular file, directory, or symlink) is created,
fscrypt randomly generates a 16-byte nonce and stores it in the
inode's encryption xattr.  Then, it uses a KDF (Key Derivation
Function) to derive the file's key from the master key and nonce.

The Adiantum encryption mode (see `Encryption modes and usage`_) is
special, since it accepts longer IVs and is suitable for both contents
and filenames encryption.  For it, a "direct key" option is offered
where the file's nonce is included in the IVs and the master key is
used for encryption directly.  This improves performance; however,
users must not use the same master key for any other encryption mode.

Below, the KDF and design considerations are described in more detail.

The current KDF works by encrypting the master key with AES-128-ECB,
using the file's nonce as the AES key.  The output is used as the
derived key.  If the output is longer than needed, then it is
truncated to the needed length.

Note: this KDF meets the primary security requirement, which is to
produce unique derived keys that preserve the entropy of the master
key, assuming that the master key is already a good pseudorandom key.
However, it is nonstandard and has some problems such as being
reversible, so it is generally considered to be a mistake!  It may be
replaced with HKDF or another more standard KDF in the future.

Key derivation was chosen over key wrapping because wrapped keys would
require larger xattrs which would be less likely to fit in-line in the
filesystem's inode table, and there didn't appear to be any
significant advantages to key wrapping.  In particular, currently
there is no requirement to support unlocking a file with multiple
alternative master keys or to support rotating master keys.  Instead,
the master keys may be wrapped in userspace, e.g. as is done by the
`fscrypt <https://github.com/google/fscrypt>`_ tool.

DIRECT_KEY policies
-------------------

The Adiantum encryption mode (see `Encryption modes and usage`_) is
suitable for both contents and filenames encryption, and it accepts
long IVs --- long enough to hold both an 8-byte logical block number
and a 16-byte per-file nonce.  Also, the overhead of each Adiantum key
is greater than that of an AES-256-XTS key.

Therefore, to improve performance and save memory, for Adiantum a
"direct key" configuration is supported.  When the user has enabled
this by setting FSCRYPT_POLICY_FLAG_DIRECT_KEY in the fscrypt policy,
per-file keys are not used.  Instead, whenever any data (contents or
filenames) is encrypted, the file's 16-byte nonce is included in the
IV.  Moreover:

- For v1 encryption policies, the encryption is done directly with the
  master key.  Because of this, users **must not** use the same master
  key for any other purpose, even for other v1 policies.

- For v2 encryption policies, the encryption is done with a per-mode
  key derived using the KDF.  Users may use the same master key for
  other v2 encryption policies.

IV_INO_LBLK_64 policies
-----------------------

When FSCRYPT_POLICY_FLAG_IV_INO_LBLK_64 is set in the fscrypt policy,
the encryption keys are derived from the master key, encryption mode
number, and filesystem UUID.  This normally results in all files
protected by the same master key sharing a single contents encryption
key and a single filenames encryption key.  To still encrypt different
files' data differently, inode numbers are included in the IVs.
Consequently, shrinking the filesystem may not be allowed.

This format is optimized for use with inline encryption hardware
compliant with the UFS or eMMC standards, which support only 64 IV
bits per I/O request and may have only a small number of keyslots.

Key identifiers
---------------

For master keys used for v2 encryption policies, a unique 16-byte "key
identifier" is also derived using the KDF.  This value is stored in
the clear, since it is needed to reliably identify the key itself.

Encryption modes and usage
==========================

fscrypt allows one encryption mode to be specified for file contents
and one encryption mode to be specified for filenames.  Different
directory trees are permitted to use different encryption modes.
Currently, the following pairs of encryption modes are supported:

- AES-256-XTS for contents and AES-256-CTS-CBC for filenames
- AES-128-CBC for contents and AES-128-CTS-CBC for filenames
- Adiantum for both contents and filenames

If unsure, you should use the (AES-256-XTS, AES-256-CTS-CBC) pair.

AES-128-CBC was added only for low-powered embedded devices with
crypto accelerators such as CAAM or CESA that do not support XTS.  To
use AES-128-CBC, CONFIG_CRYPTO_ESSIV and CONFIG_CRYPTO_SHA256 (or
another SHA-256 implementation) must be enabled so that ESSIV can be
used.

Adiantum is a (primarily) stream cipher-based mode that is fast even
on CPUs without dedicated crypto instructions.  It's also a true
wide-block mode, unlike XTS.  It can also eliminate the need to derive
per-file keys.  However, it depends on the security of two primitives,
XChaCha12 and AES-256, rather than just one.  See the paper
"Adiantum: length-preserving encryption for entry-level processors"
(https://eprint.iacr.org/2018/720.pdf) for more details.  To use
Adiantum, CONFIG_CRYPTO_ADIANTUM must be enabled.  Also, fast
implementations of ChaCha and NHPoly1305 should be enabled, e.g.
CONFIG_CRYPTO_CHACHA20_NEON and CONFIG_CRYPTO_NHPOLY1305_NEON for ARM.

New encryption modes can be added relatively easily, without changes
to individual filesystems.  However, authenticated encryption (AE)
modes are not currently supported because of the difficulty of dealing
with ciphertext expansion.

Contents encryption
-------------------

For file contents, each filesystem block is encrypted independently.
Currently, only the case where the filesystem block size is equal to
the system's page size (usually 4096 bytes) is supported.

Each block's IV is set to the logical block number within the file as
a little endian number, except that:

- With CBC mode encryption, ESSIV is also used.  Specifically, each IV
  is encrypted with AES-256 where the AES-256 key is the SHA-256 hash
  of the file's data encryption key.

- In the "direct key" configuration (FS_POLICY_FLAG_DIRECT_KEY set in
  the fscrypt_policy), the file's nonce is also appended to the IV.
  Currently this is only allowed with the Adiantum encryption mode.

Filenames encryption
--------------------

For filenames, each full filename is encrypted at once.  Because of
the requirements to retain support for efficient directory lookups and
filenames of up to 255 bytes, the same IV is used for every filename
in a directory.

However, each encrypted directory still uses a unique key, or
alternatively has the file's nonce (for `DIRECT_KEY policies`_) or
inode number (for `IV_INO_LBLK_64 policies`_) included in the IVs.
Thus, IV reuse is limited to within a single directory.

With CTS-CBC, the IV reuse means that when the plaintext filenames
share a common prefix at least as long as the cipher block size (16
bytes for AES), the corresponding encrypted filenames will also share
a common prefix.  This is undesirable.  Adiantum does not have this
weakness, as it is a wide-block encryption mode.

All supported filenames encryption modes accept any plaintext length
>= 16 bytes; cipher block alignment is not required.  However,
filenames shorter than 16 bytes are NUL-padded to 16 bytes before
being encrypted.  In addition, to reduce leakage of filename lengths
via their ciphertexts, all filenames are NUL-padded to the next 4, 8,
16, or 32-byte boundary (configurable).  32 is recommended since this
provides the best confidentiality, at the cost of making directory
entries consume slightly more space.  Note that since NUL (``\0``) is
not otherwise a valid character in filenames, the padding will never
produce duplicate plaintexts.

Symbolic link targets are considered a type of filename and are
encrypted in the same way as filenames in directory entries, except
that IV reuse is not a problem as each symlink has its own inode.

User API
========

Setting an encryption policy
----------------------------

The FS_IOC_SET_ENCRYPTION_POLICY ioctl sets an encryption policy on an
empty directory or verifies that a directory or regular file already
has the specified encryption policy.  It takes in a pointer to a
:c:type:`struct fscrypt_policy`, defined as follows::

    #define FS_KEY_DESCRIPTOR_SIZE  8

    struct fscrypt_policy {
            __u8 version;
            __u8 contents_encryption_mode;
            __u8 filenames_encryption_mode;
            __u8 flags;
            __u8 master_key_descriptor[FS_KEY_DESCRIPTOR_SIZE];
    };

This structure must be initialized as follows:

- ``version`` must be 0.

- ``contents_encryption_mode`` and ``filenames_encryption_mode`` must
  be set to constants from ``<linux/fs.h>`` which identify the
  encryption modes to use.  If unsure, use
  FS_ENCRYPTION_MODE_AES_256_XTS (1) for ``contents_encryption_mode``
  and FS_ENCRYPTION_MODE_AES_256_CTS (4) for
  ``filenames_encryption_mode``.

- ``flags`` must contain a value from ``<linux/fs.h>`` which
  identifies the amount of NUL-padding to use when encrypting
  filenames.  If unsure, use FS_POLICY_FLAGS_PAD_32 (0x3).
  In addition, if the chosen encryption modes are both
  FS_ENCRYPTION_MODE_ADIANTUM, this can contain
  FS_POLICY_FLAG_DIRECT_KEY to specify that the master key should be
  used directly, without key derivation.

- ``master_key_descriptor`` specifies how to find the master key in
  the keyring; see `Adding keys`_.  It is up to userspace to choose a
  unique ``master_key_descriptor`` for each master key.  The e4crypt
  and fscrypt tools use the first 8 bytes of
  ``SHA-512(SHA-512(master_key))``, but this particular scheme is not
  required.  Also, the master key need not be in the keyring yet when
  FS_IOC_SET_ENCRYPTION_POLICY is executed.  However, it must be added
  before any files can be created in the encrypted directory.

If the file is not yet encrypted, then FS_IOC_SET_ENCRYPTION_POLICY
verifies that the file is an empty directory.  If so, the specified
encryption policy is assigned to the directory, turning it into an
encrypted directory.  After that, and after providing the
corresponding master key as described in `Adding keys`_, all regular
files, directories (recursively), and symlinks created in the
directory will be encrypted, inheriting the same encryption policy.
The filenames in the directory's entries will be encrypted as well.

Alternatively, if the file is already encrypted, then
FS_IOC_SET_ENCRYPTION_POLICY validates that the specified encryption
policy exactly matches the actual one.  If they match, then the ioctl
returns 0.  Otherwise, it fails with EEXIST.  This works on both
regular files and directories, including nonempty directories.

Note that the ext4 filesystem does not allow the root directory to be
encrypted, even if it is empty.  Users who want to encrypt an entire
filesystem with one key should consider using dm-crypt instead.

FS_IOC_SET_ENCRYPTION_POLICY can fail with the following errors:

- ``EACCES``: the file is not owned by the process's uid, nor does the
  process have the CAP_FOWNER capability in a namespace with the file
  owner's uid mapped
- ``EEXIST``: the file is already encrypted with an encryption policy
  different from the one specified
- ``EINVAL``: an invalid encryption policy was specified (invalid
  version, mode(s), or flags; or reserved bits were set)
- ``ENOKEY``: a v2 encryption policy was specified, but the key with
  the specified ``master_key_identifier`` has not been added, nor does
  the process have the CAP_FOWNER capability in the initial user
  namespace
- ``ENOTDIR``: the file is unencrypted and is a regular file, not a
  directory
- ``ENOTEMPTY``: the file is unencrypted and is a nonempty directory
- ``ENOTTY``: this type of filesystem does not implement encryption
- ``EOPNOTSUPP``: the kernel was not configured with encryption
  support for filesystems, or the filesystem superblock has not
  had encryption enabled on it.  (For example, to use encryption on an
  ext4 filesystem, CONFIG_FS_ENCRYPTION must be enabled in the
  kernel config, and the superblock must have had the "encrypt"
  feature flag enabled using ``tune2fs -O encrypt`` or ``mkfs.ext4 -O
  encrypt``.)
- ``EPERM``: this directory may not be encrypted, e.g. because it is
  the root directory of an ext4 filesystem
- ``EROFS``: the filesystem is readonly

Getting an encryption policy
----------------------------

The FS_IOC_GET_ENCRYPTION_POLICY ioctl retrieves the :c:type:`struct
fscrypt_policy`, if any, for a directory or regular file.  See above
for the struct definition.  No additional permissions are required
beyond the ability to open the file.

FS_IOC_GET_ENCRYPTION_POLICY can fail with the following errors:

- ``EINVAL``: the file is encrypted, but it uses an unrecognized
  encryption context format
- ``ENODATA``: the file is not encrypted
- ``ENOTTY``: this type of filesystem does not implement encryption
- ``EOPNOTSUPP``: the kernel was not configured with encryption
  support for this filesystem

Note: if you only need to know whether a file is encrypted or not, on
most filesystems it is also possible to use the FS_IOC_GETFLAGS ioctl
and check for FS_ENCRYPT_FL, or to use the statx() system call and
check for STATX_ATTR_ENCRYPTED in stx_attributes.

Getting the per-filesystem salt
-------------------------------

Some filesystems, such as ext4 and F2FS, also support the deprecated
ioctl FS_IOC_GET_ENCRYPTION_PWSALT.  This ioctl retrieves a randomly
generated 16-byte value stored in the filesystem superblock.  This
value is intended to used as a salt when deriving an encryption key
from a passphrase or other low-entropy user credential.

FS_IOC_GET_ENCRYPTION_PWSALT is deprecated.  Instead, prefer to
generate and manage any needed salt(s) in userspace.

Adding keys
-----------

FS_IOC_ADD_ENCRYPTION_KEY
~~~~~~~~~~~~~~~~~~~~~~~~~

The FS_IOC_ADD_ENCRYPTION_KEY ioctl adds a master encryption key to
the filesystem, making all files on the filesystem which were
encrypted using that key appear "unlocked", i.e. in plaintext form.
It can be executed on any file or directory on the target filesystem,
but using the filesystem's root directory is recommended.  It takes in
a pointer to a :c:type:`struct fscrypt_add_key_arg`, defined as
follows::

    struct fscrypt_add_key_arg {
            struct fscrypt_key_specifier key_spec;
            __u32 raw_size;
            __u32 __reserved[9];
            __u8 raw[];
    };

    #define FSCRYPT_KEY_SPEC_TYPE_DESCRIPTOR        1
    #define FSCRYPT_KEY_SPEC_TYPE_IDENTIFIER        2

    struct fscrypt_key_specifier {
            __u32 type;     /* one of FSCRYPT_KEY_SPEC_TYPE_* */
            __u32 __reserved;
            union {
                    __u8 __reserved[32]; /* reserve some extra space */
                    __u8 descriptor[FSCRYPT_KEY_DESCRIPTOR_SIZE];
                    __u8 identifier[FSCRYPT_KEY_IDENTIFIER_SIZE];
            } u;
    };

:c:type:`struct fscrypt_add_key_arg` must be zeroed, then initialized
as follows:

- If the key is being added for use by v1 encryption policies, then
  ``key_spec.type`` must contain FSCRYPT_KEY_SPEC_TYPE_DESCRIPTOR, and
  ``key_spec.u.descriptor`` must contain the descriptor of the key
  being added, corresponding to the value in the
  ``master_key_descriptor`` field of :c:type:`struct
  fscrypt_policy_v1`.  To add this type of key, the calling process
  must have the CAP_SYS_ADMIN capability in the initial user
  namespace.

  Alternatively, if the key is being added for use by v2 encryption
  policies, then ``key_spec.type`` must contain
  FSCRYPT_KEY_SPEC_TYPE_IDENTIFIER, and ``key_spec.u.identifier`` is
  an *output* field which the kernel fills in with a cryptographic
  hash of the key.  To add this type of key, the calling process does
  not need any privileges.  However, the number of keys that can be
  added is limited by the user's quota for the keyrings service (see
  ``Documentation/security/keys/core.rst``).

- ``raw_size`` must be the size of the ``raw`` key provided, in bytes.

- ``raw`` is a variable-length field which must contain the actual
  key, ``raw_size`` bytes long.

For v2 policy keys, the kernel keeps track of which user (identified
by effective user ID) added the key, and only allows the key to be
removed by that user --- or by "root", if they use
`FS_IOC_REMOVE_ENCRYPTION_KEY_ALL_USERS`_.

However, if another user has added the key, it may be desirable to
prevent that other user from unexpectedly removing it.  Therefore,
FS_IOC_ADD_ENCRYPTION_KEY may also be used to add a v2 policy key
*again*, even if it's already added by other user(s).  In this case,
FS_IOC_ADD_ENCRYPTION_KEY will just install a claim to the key for the
current user, rather than actually add the key again (but the raw key
must still be provided, as a proof of knowledge).

FS_IOC_ADD_ENCRYPTION_KEY returns 0 if either the key or a claim to
the key was either added or already exists.

FS_IOC_ADD_ENCRYPTION_KEY can fail with the following errors:

- ``EACCES``: FSCRYPT_KEY_SPEC_TYPE_DESCRIPTOR was specified, but the
  caller does not have the CAP_SYS_ADMIN capability in the initial
  user namespace
- ``EDQUOT``: the key quota for this user would be exceeded by adding
  the key
- ``EINVAL``: invalid key size or key specifier type, or reserved bits
  were set
- ``ENOTTY``: this type of filesystem does not implement encryption
- ``EOPNOTSUPP``: the kernel was not configured with encryption
  support for this filesystem, or the filesystem superblock has not
  had encryption enabled on it

Legacy method
~~~~~~~~~~~~~

For v1 encryption policies, a master encryption key can also be
provided by adding it to a process-subscribed keyring, e.g. to a
session keyring, or to a user keyring if the user keyring is linked
into the session keyring.

This method is deprecated (and not supported for v2 encryption
policies) for several reasons.  First, it cannot be used in
combination with FS_IOC_REMOVE_ENCRYPTION_KEY (see `Removing keys`_),
so for removing a key a workaround such as keyctl_unlink() in
combination with ``sync; echo 2 > /proc/sys/vm/drop_caches`` would
have to be used.  Second, it doesn't match the fact that the
locked/unlocked status of encrypted files (i.e. whether they appear to
be in plaintext form or in ciphertext form) is global.  This mismatch
has caused much confusion as well as real problems when processes
running under different UIDs, such as a ``sudo`` command, need to
access encrypted files.

Nevertheless, to add a key to one of the process-subscribed keyrings,
the add_key() system call can be used (see:
``Documentation/security/keys/core.rst``).  The key type must be
"logon"; keys of this type are kept in kernel memory and cannot be
read back by userspace.  The key description must be "fscrypt:"
followed by the 16-character lower case hex representation of the
``master_key_descriptor`` that was set in the encryption policy.  The
key payload must conform to the following structure::

    #define FS_MAX_KEY_SIZE 64

    struct fscrypt_key {
            u32 mode;
            u8 raw[FS_MAX_KEY_SIZE];
            u32 size;
    };

``mode`` is ignored; just set it to 0.  The actual key is provided in
``raw`` with ``size`` indicating its size in bytes.  That is, the
bytes ``raw[0..size-1]`` (inclusive) are the actual key.

The key description prefix "fscrypt:" may alternatively be replaced
with a filesystem-specific prefix such as "ext4:".  However, the
filesystem-specific prefixes are deprecated and should not be used in
new programs.

There are several different types of keyrings in which encryption keys
may be placed, such as a session keyring, a user session keyring, or a
user keyring.  Each key must be placed in a keyring that is "attached"
to all processes that might need to access files encrypted with it, in
the sense that request_key() will find the key.  Generally, if only
processes belonging to a specific user need to access a given
encrypted directory and no session keyring has been installed, then
that directory's key should be placed in that user's user session
keyring or user keyring.  Otherwise, a session keyring should be
installed if needed, and the key should be linked into that session
keyring, or in a keyring linked into that session keyring.

Note: introducing the complex visibility semantics of keyrings here
was arguably a mistake --- especially given that by design, after any
process successfully opens an encrypted file (thereby setting up the
per-file key), possessing the keyring key is not actually required for
any process to read/write the file until its in-memory inode is
evicted.  In the future there probably should be a way to provide keys
directly to the filesystem instead, which would make the intended
semantics clearer.

Access semantics
================

With the key
------------

With the encryption key, encrypted regular files, directories, and
symlinks behave very similarly to their unencrypted counterparts ---
after all, the encryption is intended to be transparent.  However,
astute users may notice some differences in behavior:

- Unencrypted files, or files encrypted with a different encryption
  policy (i.e. different key, modes, or flags), cannot be renamed or
  linked into an encrypted directory; see `Encryption policy
  enforcement`_.  Attempts to do so will fail with EXDEV.  However,
  encrypted files can be renamed within an encrypted directory, or
  into an unencrypted directory.

  Note: "moving" an unencrypted file into an encrypted directory, e.g.
  with the `mv` program, is implemented in userspace by a copy
  followed by a delete.  Be aware that the original unencrypted data
  may remain recoverable from free space on the disk; prefer to keep
  all files encrypted from the very beginning.  The `shred` program
  may be used to overwrite the source files but isn't guaranteed to be
  effective on all filesystems and storage devices.

- Direct I/O is not supported on encrypted files.  Attempts to use
  direct I/O on such files will fall back to buffered I/O.

- The fallocate operations FALLOC_FL_COLLAPSE_RANGE,
  FALLOC_FL_INSERT_RANGE, and FALLOC_FL_ZERO_RANGE are not supported
  on encrypted files and will fail with EOPNOTSUPP.

- Online defragmentation of encrypted files is not supported.  The
  EXT4_IOC_MOVE_EXT and F2FS_IOC_MOVE_RANGE ioctls will fail with
  EOPNOTSUPP.

- The ext4 filesystem does not support data journaling with encrypted
  regular files.  It will fall back to ordered data mode instead.

- DAX (Direct Access) is not supported on encrypted files.

- The st_size of an encrypted symlink will not necessarily give the
  length of the symlink target as required by POSIX.  It will actually
  give the length of the ciphertext, which will be slightly longer
  than the plaintext due to NUL-padding and an extra 2-byte overhead.

- The maximum length of an encrypted symlink is 2 bytes shorter than
  the maximum length of an unencrypted symlink.  For example, on an
  EXT4 filesystem with a 4K block size, unencrypted symlinks can be up
  to 4095 bytes long, while encrypted symlinks can only be up to 4093
  bytes long (both lengths excluding the terminating null).

Note that mmap *is* supported.  This is possible because the pagecache
for an encrypted file contains the plaintext, not the ciphertext.

Without the key
---------------

Some filesystem operations may be performed on encrypted regular
files, directories, and symlinks even before their encryption key has
been provided:

- File metadata may be read, e.g. using stat().

- Directories may be listed, in which case the filenames will be
  listed in an encoded form derived from their ciphertext.  The
  current encoding algorithm is described in `Filename hashing and
  encoding`_.  The algorithm is subject to change, but it is
  guaranteed that the presented filenames will be no longer than
  NAME_MAX bytes, will not contain the ``/`` or ``\0`` characters, and
  will uniquely identify directory entries.

  The ``.`` and ``..`` directory entries are special.  They are always
  present and are not encrypted or encoded.

- Files may be deleted.  That is, nondirectory files may be deleted
  with unlink() as usual, and empty directories may be deleted with
  rmdir() as usual.  Therefore, ``rm`` and ``rm -r`` will work as
  expected.

- Symlink targets may be read and followed, but they will be presented
  in encrypted form, similar to filenames in directories.  Hence, they
  are unlikely to point to anywhere useful.

Without the key, regular files cannot be opened or truncated.
Attempts to do so will fail with ENOKEY.  This implies that any
regular file operations that require a file descriptor, such as
read(), write(), mmap(), fallocate(), and ioctl(), are also forbidden.

Also without the key, files of any type (including directories) cannot
be created or linked into an encrypted directory, nor can a name in an
encrypted directory be the source or target of a rename, nor can an
O_TMPFILE temporary file be created in an encrypted directory.  All
such operations will fail with ENOKEY.

It is not currently possible to backup and restore encrypted files
without the encryption key.  This would require special APIs which
have not yet been implemented.

Encryption policy enforcement
=============================

After an encryption policy has been set on a directory, all regular
files, directories, and symbolic links created in that directory
(recursively) will inherit that encryption policy.  Special files ---
that is, named pipes, device nodes, and UNIX domain sockets --- will
not be encrypted.

Except for those special files, it is forbidden to have unencrypted
files, or files encrypted with a different encryption policy, in an
encrypted directory tree.  Attempts to link or rename such a file into
an encrypted directory will fail with EXDEV.  This is also enforced
during ->lookup() to provide limited protection against offline
attacks that try to disable or downgrade encryption in known locations
where applications may later write sensitive data.  It is recommended
that systems implementing a form of "verified boot" take advantage of
this by validating all top-level encryption policies prior to access.

Implementation details
======================

Encryption context
------------------

An encryption policy is represented on-disk by a :c:type:`struct
fscrypt_context`.  It is up to individual filesystems to decide where
to store it, but normally it would be stored in a hidden extended
attribute.  It should *not* be exposed by the xattr-related system
calls such as getxattr() and setxattr() because of the special
semantics of the encryption xattr.  (In particular, there would be
much confusion if an encryption policy were to be added to or removed
from anything other than an empty directory.)  The struct is defined
as follows::

    #define FS_KEY_DESCRIPTOR_SIZE  8
    #define FS_KEY_DERIVATION_NONCE_SIZE 16

    struct fscrypt_context {
            u8 format;
            u8 contents_encryption_mode;
            u8 filenames_encryption_mode;
            u8 flags;
            u8 master_key_descriptor[FS_KEY_DESCRIPTOR_SIZE];
            u8 nonce[FS_KEY_DERIVATION_NONCE_SIZE];
    };

    #define FSCRYPT_KEY_IDENTIFIER_SIZE  16
    struct fscrypt_context_v2 {
            u8 version;
            u8 contents_encryption_mode;
            u8 filenames_encryption_mode;
            u8 flags;
            u8 __reserved[4];
            u8 master_key_identifier[FSCRYPT_KEY_IDENTIFIER_SIZE];
            u8 nonce[FS_KEY_DERIVATION_NONCE_SIZE];
    };

The context structs contain the same information as the corresponding
policy structs (see `Setting an encryption policy`_), except that the
context structs also contain a nonce.  The nonce is randomly generated
by the kernel and is used as KDF input or as a tweak to cause
different files to be encrypted differently; see `Per-file keys`_ and
`DIRECT_KEY policies`_.

Data path changes
-----------------

For the read path (->readpage()) of regular files, filesystems can
read the ciphertext into the page cache and decrypt it in-place.  The
page lock must be held until decryption has finished, to prevent the
page from becoming visible to userspace prematurely.

For the write path (->writepage()) of regular files, filesystems
cannot encrypt data in-place in the page cache, since the cached
plaintext must be preserved.  Instead, filesystems must encrypt into a
temporary buffer or "bounce page", then write out the temporary
buffer.  Some filesystems, such as UBIFS, already use temporary
buffers regardless of encryption.  Other filesystems, such as ext4 and
F2FS, have to allocate bounce pages specially for encryption.

Filename hashing and encoding
-----------------------------

Modern filesystems accelerate directory lookups by using indexed
directories.  An indexed directory is organized as a tree keyed by
filename hashes.  When a ->lookup() is requested, the filesystem
normally hashes the filename being looked up so that it can quickly
find the corresponding directory entry, if any.

With encryption, lookups must be supported and efficient both with and
without the encryption key.  Clearly, it would not work to hash the
plaintext filenames, since the plaintext filenames are unavailable
without the key.  (Hashing the plaintext filenames would also make it
impossible for the filesystem's fsck tool to optimize encrypted
directories.)  Instead, filesystems hash the ciphertext filenames,
i.e. the bytes actually stored on-disk in the directory entries.  When
asked to do a ->lookup() with the key, the filesystem just encrypts
the user-supplied name to get the ciphertext.

Lookups without the key are more complicated.  The raw ciphertext may
contain the ``\0`` and ``/`` characters, which are illegal in
filenames.  Therefore, readdir() must base64-encode the ciphertext for
presentation.  For most filenames, this works fine; on ->lookup(), the
filesystem just base64-decodes the user-supplied name to get back to
the raw ciphertext.

However, for very long filenames, base64 encoding would cause the
filename length to exceed NAME_MAX.  To prevent this, readdir()
actually presents long filenames in an abbreviated form which encodes
a strong "hash" of the ciphertext filename, along with the optional
filesystem-specific hash(es) needed for directory lookups.  This
allows the filesystem to still, with a high degree of confidence, map
the filename given in ->lookup() back to a particular directory entry
that was previously listed by readdir().  See :c:type:`struct
fscrypt_digested_name` in the source for more details.

Note that the precise way that filenames are presented to userspace
without the key is subject to change in the future.  It is only meant
as a way to temporarily present valid filenames so that commands like
``rm -r`` work as expected on encrypted directories.

Tests
=====

To test fscrypt, use xfstests, which is Linux's de facto standard
filesystem test suite.  First, run all the tests in the "encrypt"
group on the relevant filesystem(s).  For example, to test ext4 and
f2fs encryption using `kvm-xfstests
<https://github.com/tytso/xfstests-bld/blob/master/Documentation/kvm-quickstart.md>`_::

    kvm-xfstests -c ext4,f2fs -g encrypt

UBIFS encryption can also be tested this way, but it should be done in
a separate command, and it takes some time for kvm-xfstests to set up
emulated UBI volumes::

    kvm-xfstests -c ubifs -g encrypt

No tests should fail.  However, tests that use non-default encryption
modes (e.g. generic/549 and generic/550) will be skipped if the needed
algorithms were not built into the kernel's crypto API.  Also, tests
that access the raw block device (e.g. generic/399, generic/548,
generic/549, generic/550) will be skipped on UBIFS.

Besides running the "encrypt" group tests, for ext4 and f2fs it's also
possible to run most xfstests with the "test_dummy_encryption" mount
option.  This option causes all new files to be automatically
encrypted with a dummy key, without having to make any API calls.
This tests the encrypted I/O paths more thoroughly.  To do this with
kvm-xfstests, use the "encrypt" filesystem configuration::

    kvm-xfstests -c ext4/encrypt,f2fs/encrypt -g auto

Because this runs many more tests than "-g encrypt" does, it takes
much longer to run; so also consider using `gce-xfstests
<https://github.com/tytso/xfstests-bld/blob/master/Documentation/gce-xfstests.md>`_
instead of kvm-xfstests::

    gce-xfstests -c ext4/encrypt,f2fs/encrypt -g auto
