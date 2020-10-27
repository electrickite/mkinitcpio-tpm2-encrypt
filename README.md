mkinitcpio TPM2 hook
====================

This mkinitcpio hook allows for an encrypted root device to use a key sealed by
a TPM 2.0.

During boot, the hook will initialize the TPM and attempt to unseal the key. If
the key is successfully unsealed, it will be passed to the `encrypt` hook to
perform the actual decryption of the root file system.

Depending on the PCR banks to which the sealed key is bound, system changes such
as kernel updates or firmware adjustments may prevent the key from being
unsealed. If this happens, the disk must be manually unlocked with a passphrase
and a new sealed key file needs to be generated. For this reason, it is CRUCIAL
to add a separate "recovery" passphrase to the LUKS keys.


Use
---

The `tpm2` hook should be placed immediately before the `encrypt` hook in
`/etc/mkinitcpio.conf`.

    HOOKS="... block tpm2 encrypt filesystems ...
    
In case of systemd init the `sd-tpm2` hook should be used instead and placed 
immediately before the `sd-encrypt` hook in `/etc/mkinitcpio.conf`.


    HOOKS="... block sd-tpm2 sd-encrypt filesystems ...

You may also need to add the `vfat` file system driver to the `MODULES` array:

    MODULES=(vfat)

Finally, rebuild the initramfs:

    # mkinitcpio -p linux


TPM configuration
-----------------

The `tpm2` hook attempts to "unseal" a LUKS keyfile previously sealed by the
TPM. The sealed files must reside on an unencrypted filesystem available to the
kernel at boot or may be stored in TPM non-volatile memory (NVRAM). For example,
assuming your unencrypted keyfile is at `/root/mykey` and a primary TPM key has
been persisted to `0x81000001`:

    # tpm2_createpolicy --policy-pcr -l sha1:0,2,4,7 -L pcr.pol
    # tpm2_create -C 0x81000001 -g sha256 -G keyedhash -a 0x492 -i /root/mykey \
      -L pcr.pol -r /boot/mykey.priv -u /boot/mykey.pub


Kernel command line parameters
------------------------------

The hook is controlled by a number of kernel command line parameters. Minimally,
after generating a TPM-sealed key, both `tpmkey` and `tpmpcr` should be
specified.

### tpmkey

The `tpmkey` parameter has several formats:

    tpmkey=[device]:[path]:[handle]
    tpmkey=[device]:[publicpath]:[privatepath]:[handle]
    tpmkey=nvram:[index]
    tpmkey=nvram:[index]:[offset]:[size]

Where `[device]` represents the raw block device on which the key exists,
`[path]` is the absolute base path of the sealed files within the device, and
`[handle]` is the TPM handle of the key's parent object. If only `[path]` is
specified, '.pub' and '.priv' will be appended to the path to locate the public
and private files, respectively. The absolute `[publicpath]` and `[privatepath]`
can be specified separately if needed. For example, if `/dev/sda1` is an EFI
partition mounted at `/boot`:

    tpmkey=/dev/sda1:/mykey:0x81000001

If `[device]` is `rootfs`, the key files will be read from the initramfs root
file system.

Setting `[device]` to 'nvram' indicates that the key is stored in TPM NVRAM. In
this case `[index]` is the NVRAM area index, `[offset]` is the offset of the key
in bytes and `[size]` is the size of the key in bytes.

### tpmpcr

The `tpmpcr` parameter should hold the TPM2 PCR bank specification that will
unlock the sealed key.

    tpmpcr=sha1:0,2,7

Multiple specs can be separated by a '|' and key decryption will be attempted
with each set of banks.

    tpmpcr=sha1:0,2,4,7|sha1:0,2,7

### tpmextend

The `tpmextend` parameter may be used to indicate a PCR to extend _after_ the
key has been unsealed.

    tpmextend=[alg]:[pcrnum]

Where `[alg]` is the bank algorithm and `[pcrnum]` is the PCR number to extend.
For example, to extend PCR 8 in the sha1 bank:

    tpmextend=sha1:8

### tpmprompt

If the `tpmprompt` command line parameter is set, the user will be prompted for
the parent encryption key password during boot. This password will be used while
loading the sealed key. This option has no effect when the key is stored in
NVRAM.

    tpmprompt=1

### Other parameters

If required, the TPM device can be set using `tpmdev`. The default is the in-
kernel resource manager, `/dev/tpmrm0`.

In recent kernel versions, some systems may not generate enough entropy early in
the boot process to utilize the TPM. There are several possible solutions to
this problem. On x86_64 systems, the following kernel parameter may help:

    random.trust_cpu=on
