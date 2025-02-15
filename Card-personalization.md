# Card personalization


## Introduction

> Nothing is impossible for the man who doesn't have to do it himself. -- A.H. Weiler

This guide is about initializing and personalizing (no distinction made) cards with the OpenSC library and tools (mostly pkcs15-init).

Some knowlegde about smart cards is assumed. Below is a short overview of some
key words and concepts. 

### Filesystem - MF - DF - EF - FID

  A smart cards has a non-volatile memory (EEPROM) in which usually a PC-like file system is implemented. The directories are called Dedicated Files (DF) and the files are called Elementary Files (EF). They are identified by a a File ID (FID) of 2 bytes. For example, the root of the file system (called Master File or MF) has FID = ``3F 00`` (hex).

### Commands - APDUs

  It is possible to send commands (APDUs) to the card to select, read, write, create, list, delete, ... EFs and DFs (not all cards support or allow all commands).

### Access control - PIN, PUK
 
 The file system usually implements some sort of access control on EFs and DFs. This is usually done by PINs or keys: you have to provide a PIN or show knowledge of a key before you can perform some command on some EF/DF. A PIN is often accompanied by a PUK (Pin Unblock Key), which can be used to reset (or unblock) that PIN.

### Security Officer - SO

  Security officer is a person (or a role) who manages the smart card issuing process. A SO PIN can be asked if modifications can be made to the card (keys generated or certificates written), but the existence of a SO is not mandatory. 

### Cryptographic keys

  On cryptographic cards, it is possible to sign, decrypt and generate key pairs (what can be done exactly depends on the card). On some cards, key and/or PINs are files in the filesystem, on other cards, they don't exist in the filesystem but are referenced through an ID.

### Reader - PC/SC or OpenCT, CT-API

  Smart card readers come with a library that can be used on a PC to send APDUs to the card. The common API is PC/SC, but also OpenCT and CT-API are used with some readers.

### PKCS#15

  There are standards (e.g. ISO7816, parts 4-...) that specify how to select, read, write EFs and DFs, and how to sign, decrypt, verify PIN etc. However, there is also a need to know which files contain what, or where the keys and PINs can be found.

  For crypto cards, PKCS#15 adresses this need by defining some files that contain info on where to find keys, certificates, PINs, and other data. For example, there is a PrKDF (Private Key Directory File) that contains the EFs or ID of the private keys, what those keys can be used for, by which PINs they are protected and so on.

  So a "PKCS#15 card" is nothing but any other card on which the right set of files has been added. In short: PKCS#15 allows you to describe where to find PINs, keys, certificates and data on a card, plus all the info that is needed to use them.

## A little PKCS#15 example

Here's the textual contents of 3 PKCS#15 files: the AODF (Authentication Object Directory File), PrKDF (Private Key Directory File) and CDF (Certificate Directory File) that contain info on resp. the PINs, private keys and  certificates. 

Each of these example files contains 1 entry.

```
AODF:
    Com. Flags  : private, modifiable
    Auth ID     : 01
    Flags       : [0x32], local, initialized, needs-padding
    Length      : min_len:4, max_len:8, stored_len:8
    Pad char    : 0x00
    Reference   : 1
    Encoding    : ASCII-numeric
    Path        : 3F005015

PrKDF:
    Com. Flags  : private, modifiable
    Com. Auth ID: 01
    Usage       : [0x32E], decrypt, sign, signRecover, unwrap, derive, nonRep
    Access Flags: [0x1D], sensitive, alwaysSensitive, neverExtract, local
    ModLength   : 1024
    Key ref     : 0
    Native      : yes
    Path        : 3F00501530450012
    ID          : 45

X.509 Certificate [/C=BE/ST=...]
    Com. Flags  : modifiable
    Authority   : no
    Path        : 3f0050154545
    ID          : 45
```

Some things to note:
 * The Auth ID (01) of the private key is the same as the one of the PIN which means that you first have to do a login with this PIN before you can use this key.
 * The key is in an EF with ID = ``0012`` in the DF with ID = ``3045``, which on its turn is a DF with ID = ``5015``, which on its turn is a DF of the MF (``3F00``).
 * The private key and certificate share the same ID = ``45``, which means that they belong together.
 * The cert is in the EF with as path = ``3F00\5015\4545`` and is no CA cert.

Use the _pkcs15-tool --dump_ tool to see yourself what pkcs15 data is on your card, or _opensc-explorer_ to browse through the files.

Have the PKCS#15 files a fixed place so everyone can find them? No, there's only one: the EF(DIR) in the MF and with ID 2F00. That's the starting place.


## The OpenSC pkcs15-init tool and profiles

Reading and writing files, PIN verification, signing and decryption happen in much the same way on all cards. Therefore, the "normal life" commands have been implemented in OpenSC for all supported cards.

However, creating and deleting files, PINs and keys is very card specific and has not yet been implemented for all cards. The following cards support personalization:
* [Athena ASEPCOS ASEKey](https://github.com/OpenSC/OpenSC/wiki/Athena-ASEPCOS-ASEKey)
* entersafe.c
* [IAS-ECC](https://github.com/OpenSC/OpenSC/wiki/IAS-ECC)
* Oberthur 64k
* [Aktiv Co. Rutoken S](https://github.com/OpenSC/OpenSC/wiki/Aktiv-Co.-Rutoken-S)
* [Oberthur AuthentIC](https://github.com/OpenSC/OpenSC/wiki/Oberthur-AuthentIC-applet-v2.2)
* [Feitian ePass2003](https://github.com/OpenSC/OpenSC/wiki/Feitian-ePass2003)
* [SmartCardHSM](https://github.com/OpenSC/OpenSC/wiki/SmartCardHSM)
* [Siemens CardOS M4](https://github.com/OpenSC/OpenSC/wiki/Siemens-CardOS-M4)
* [GIDS Applet](https://www.mysmartlogon.com/generic-identity-device-specification-gids-smart-card/)
* isoApplet
* MUSCLE card applet
* [OpenPGP card](https://github.com/OpenSC/OpenSC/wiki/OpenPGP-card)
* [Setcos](https://github.com/OpenSC/OpenSC/wiki/Setcos-driver)
* [Schlumberger Axalto Cryptoflex](https://github.com/OpenSC/OpenSC/wiki/Schlumberger-Axalto-Cryptoflex)
* [Schlumberger Axalto Cyberflex](https://github.com/OpenSC/OpenSC/wiki/Schlumberger-Axalto-Cyberflex)
* [Aventra MyEID PKI card](https://github.com/OpenSC/OpenSC/wiki/Aventra-MyEID-PKI-card)
* [StarCos 2.2](https://github.com/OpenSC/OpenSC/wiki/STARCOS-cards)

#### Profile

Because the initialisation/personalisation is so card-specific, it would be very hard to make a tool or API that accepts all parameters for all current and future cards. Therefore, a profile file has been made in OpenSC that
contains all the card-specific parameters. This card-specific profile is read by card-specific code in the pkcs15-init library each time this library is used on that card.

See the *.profile files in ``src/pkcs15init/``. There is one general file (``pkcs15.profile``) and one card-specific profile for each card.

#### Profile options

There are currently 3 options you can specify to modify a profile:
 * default: creation/deletion/generation is controlled by the SO PIN (*Note that this different from the regular user of the card)
 * onepin: creation/deletion/generation is controlled by the user PIN and thus by the user. As a result, only 1 user PIN is possible
 * small: like default, but suitable for card with little memory

## pkcs15-init tool

This is the main personalization tool that allows you to do the __all the initialization things__, e.g. add/delete keys, certificates, PINs and data, generate keys, while specifying key usage, which PIN protects which key etc..

As said before, not all cards are supported for initialization. In that case, the pkcs15-init tool won't work (top 5 questions on the mailing list :-).

To find out which card you have, try ``opensc-tool -n``.

Below is explained how to do the operations that are supported by pkcs15-tool.  Not all options might explained here (run ``pkcs15-tool -h`` to see them).

Disclaimer: The things in this section are fairly general but not guaranteed to work for all cards. See also the section on "card-specific issues".

The ``--reader`` or ``-r`` can be given with any command. By default the first reader with a card is used. Do ``opensc-tool -l`` to see the list of available readers.

The typical order of the commands is:
 * erase ``-E`` the card if needed
 * create ``-C`` the PKCS15 files
 * add a user PIN (unless you did a ``-C`` with the ``onepin`` profile option)
 * add a key + cert ``-S`` or generate a key ``-G`` + a add a cert ``-W``

To see the results of what you did, you can do one of the following:

```
pkcs15-tool --list-pins --list-public-keys -k -c -C
pkcs15-tool --dump
```
To see/dump the content of any file, use the ``opensc-explorer`` tool.

### Create the PKCS15 files
```
pkcs15-init -C {-T} {-p <profile>} --so-pin <PIN> --so-puk <PUK> | --no-so-pin | --pin <PIN> --puk <PUK>
```

This will create the PKCS15 DF (5015) and all the PKCS15 files (some of which will be empty until a key, PIN, ... will be added). It must be done before you can do any of the operations below.

 * This operation may require a 'transport' key. ``pkcs15-init`` will ask you for this key and propose the default one for that card. With ``-T``, the default key will be used without asking. NOTE: if you get a "Failed to erase card: PIN code or key incorrect", the transport key is wrong. Find this key and then try again, <b>DO NOT</b> try with the default key again or you may permanently lock your card!
 * If you want an SO PIN and PUK, you can specify them with the ``--so-pin`` and ``--so-puk`` options, or specify --no-so-pin if you don't want them. If you use the onepin profile, there is no SO PIN so you should specify --pin and --puk instead. (So you get: ``pkcs15-init -CT -p pkcs15+onepin --pin <PIN> --puk <PUK>``)
 * To specify the profile + option. The profile file can only be "pkcs15" for the moment, so you can have:
  * ``pkcs15+default`` : the default (not needed to specify it)
  * ``pkcs15+onepin``  : for the onepin profile option
  * ``pkcs15+small``   : for the small profile option

### Erase the card's content
```
pkcs15-init -E {-T}
```

This will delete all keys, PINs, certificates, data that were listed in PKCS15 files, along with the PKCS15 files themselves.
 * This operation may require a 'transport' key. ``pkcs15-init`` will ask you for this key and propose the default one for that card. With ``-T``, the default key will be used without asking. NOTE: if you get a "Failed to erase card: PIN code or key incorrect", the transport key is wrong. Find this key and then try again, <b>DO NOT</b> try the default key again or you may permanently lock your card!

### Add a PIN (not possible with the onepin profile option)
```
pkcs15-init -P {-a <AuthID>} {--pin <PIN>} {--puk <PUK>} {-l <label>}
```

 * You can specify the AuthID with ``-a``, if you don't do so, a value that didn't exist yet on the card will be used.
 * Specify the PIN and PUK with ``--pin`` and ``--puk``, if you don't do so, the tool will prompt you for one.
 * Specify the label (name) of the PIN with ``-l``, or accept the default label.

### Generate a key pair
```
pkcs15-init -G <keyspec> -a <AuthID> --insecure {-i <ID>}
   {-u <keyusage>}
   {-l <privkeylabel>} {--public-key-label <pubkeylabel>}
```

This will generate a public and private key pair.
 * The keyspec consist of the key type (only RSA is supported) and optinally a slash followed by the keysize in bits (defaults to 1024). E.g to generate a 1024-bit RSA key, use ``pkcs15-init -G rsa/1024 -a 01 -l testkey``
 * Specify the AuthID of the PIN that protects this key (protect from being used in a signature or decryption operation) with ``-a``; or specify ``--insecure`` if you want the private key to be usable without first providing a PIN. 
 * Specify the ID of the key with ``-i``, otherwise the tool will choose one.
 * Specify the usage of the private key with ``-u``; if you add a corresponding certificate later, it should have the same key usage.
   (Do ``pkcs15-init -u help`` for help).
 * Specify the label (name) of the private key with ``-l``, or accept the default label.
 * Specify the label (name) of the public key with ``--public-key-label``, or accept the default label.
 * Depending on your card and profile option, you will be prompted to provide
   your SO PIN and/or PIN; if you don't want to be prompted, add them to the
   command line with ``--so-pin <SOPIN>`` and/or ``--pin <PIN>``.

NOTE: see the SSL engines (below) on how to make a certificate request with the key you generated.

### Add a private key
```
pkcs15-init -S <keyfile> {-f <keyformat>} -a <AuthID> --insecure
   {-i <ID>} {-u <keyusage>} {--passphrase <password>}
   {-l <label>}
```

 * The keyfile should be in DER (binary) or PEM format.
 * The keyformat should be PEM (default) or DER.
 * Specify the AuthID of the PIN that protects the private key (from being used in a signature or decryption operation) with ``-a``; or specify ``--insecure`` if you want the private key to be used without first providing a PIN
 * Specify the ID of the key with ``-i``
 * Specify the usage of the private key with ``-u``; if you add a corresponding certificate later, it should have the same key usage.
   (Do ``pkcs15-init -u help`` for help).
 * Specify the label (name) of the  with ``-l``, or accept the default label if you don't do so.
 * Depending on your card and profile option, you will be prompted to provide your SO PIN and/or PIN; if you don't want to be prompted, add them to the command line with ``--so-pin <SOPIN>`` and/or ``--pin <PIN>``.

### Add a private key + certificate(s) (in a pkcs12 file)
```
pkcs15-init -S <pkcs12file> -f PKCS12 -a <AuthID> {--insecure} {-i <ID>}
   {-u <keyusage>} {--passphrase <password>}
   {-l <privkeylabel>} {--cert-label <usercertlabel>}
```

This adds the private key and certificate chain to the card. If a certificate already exists in the card, it won't be added again.
 * Specify the AuthID of the PIN that protects this key (protect from being used in a signature or decryption operation) with ``-a``; or specify ``--insecure`` if you want the private key to be used without first providing a PIN.
 * Specify the ID of the key and the corresponding certificate with ``-i``, otherwise the tool with choose one; only the 'user cert' will get the same ID as the key, the other certificates will get 'authority' status and another ID.
 * You can specify the key-usage, but it is advised not to do this so the key usage is fetched from the certificate.
 * Specify the password of the pkcs12 key file if you don't want to be prompted for one.
 * Specify the label (name) of the private key with ``-l``, or accept the default label if you don't do so.
 * Specify the label (name) of the user certificate with ``--cert-label``, or accept the default label if you don't do so.
 * Depending on your card and profile option, you will be prompted to provide your SO PIN and/or PIN; if you don't want to be prompted, add them to the command line with ``--so-pin <SOPIN>`` and/or ``--pin <PIN>``.

### Add a certificate
```
pkcs15-init -X <certfile> {-f <certformat>} {-i <ID>} {--authority}
```

 * The certfile should be in {{DER}} (binary) or {{PEM}} format. PEM is assumed as the format unless specified via certformat.
 * Specify the ID of the certificate with ``-i``, otherwise the tool with choose one; if the certificate corresponds to a private and/or public key, you should specify the same ID as that key.
 * Specify ``--authority`` if it is a CA certificate.
 * Depending on your card and profile option, you will be prompted to provide your SO PIN and/or PIN; if you don't want to be prompted, add them to the command line with ``--so-pin <SOPIN>`` and/or ``--pin <PIN>``.

### Add a public key
```
pkcs15-init --store-public-key <keyfile> {-f <keyformat>} {-i <ID>}
                  {-l <label>}
```

 * The keyfile should be in DER (binary) or PEM format
 * The keyformat should be PEM (default) or DER
 * Specify the ID of the key with ``-i``, otherwise the tool with choose one; if the key corresponds to a private key and/or certificate, you should specify the same ID as that private key and/or certificate.
 * Specify the label (name) of the  with ``-l``, or accept the default label if you don't do so.
 * Depending on your card and profile option, you will be prompted to provide your SO PIN and/or PIN; if you don't want to be prompted, add them to the command line with ``--so-pin <SOPIN>`` and/or ``--pin <PIN>``.

### Add data
```
pkcs15-init -W <datafile> {-i <ID>} {-l <label>}
```

 * The datafile is stored "as is" onto the card.
 * Specify the ID of the data with ``-i``, or accept the default ID.
 * Specify the label (name) of the data with ``-l``, or accept the default label.
 * Depending on your card and profile option, you will be prompted to provide your SO PIN and/or PIN; if you don't want to be prompted, add them to the command line with ``--so-pin <SOPIN>`` and/or ``--pin <PIN>``.

### Update a certificate
```
pkcs15-init -U <certfile> -f <format> -i <ID> {-a <pinid>}
```

 * Specify path to the cert file with ``-U``, default format = PEM
 * Specify the cert format (DER or PEM) with ``-f``
 * Specify the ID of the cert with ``-i``.
 * Specify the ID of the PIN needed to update the cert file (and if needed to delete it and create and write a new one) with ``-a``.
 * Depending on your card and profile option, you will be prompted to provide your SO PIN and/or PIN; if you don't want to be prompted, add them to the command line with ``--so-pin <SOPIN>`` and/or ``--pin <PIN>``.
 * <b>NOTE</b>: if the new cert is bigger then the old one, the tool will try to delete the old cert file and create a new one. This won't work for most card (probably SetCOS 4.4.1 is the only one where it works..)

### Change attributes (currently only the label)
```
pkcs15-init -A <type> -i <ID> -l <label> {-a <pinid>}
```

This allows you to modify the label of a certain PKCS15 object.
 * The type of the object should be one of the following: privkey, pubkey, cert, data.
 * Specify the ID of the object with ``-i``.
 * Specify the new label with ``-l``.
 * Specify the ID of the PIN needed to update the corresponding PKCS15 file with ``-a``.
 * Depending on your card and profile option, you will be prompted to provide your SO PIN and/or PIN; if you don't want to be prompted, add them to the command line with ``--so-pin <SOPIN>`` and/or ``--pin <PIN>``.

## Other tools

### SSL-engines

These libraries can be loaded in OpenSSL so you can do a certificate request with the openssl tool; the signature of the certificate request will then be made using the smart card. The result can then be sent to a CA for certification or the resulting certificate can be put on the card with pkcs15-init or pkcs11-tool.

 * Run ``openssl``
 * On the ``openssl`` command prompt, type
```
engine dynamic -pre SO_PATH:engine_pkcs11 -pre ID:pkcs11 -pre LIST_ADD:1 -pre LOAD
```
   to use the PKCS #11 engine
 * Then type (on the openssl command prompt)
```
req -engine pkcs11 -new -key <ID> -keyform engine -out <cert_req>
```
   in which ID is the slot+ID in the following format:
   [slot_<slotID>][-][id_<ID>], e.g. id_45 or slot_0-id_45

### ``pkcs11-tool`` and Mozilla/Netscape

You can use the OpenSC pkcs11 library to generate a key pair in Mozilla or Netscape, and let the browser generate a certificate request that is sent to an on-line CA to issue and send your a certificate that is then added to the card.

Just go to an online CA (Globalsign, Thawte, ...) and follow their guidelines.  Because such a request either costs you or at least requires you to provide a valid mail address, it is advisable to first try you card with ``pkcs11-tool --moz-cert <cert_file_in_der_format> --login``.

NOTE: This can only be done with the onepin profile option (because the browser won't ask for an SO PIN, only for the user PIN).


## Card-specific issues

> Experience is that marvellous thing that enables you to recognize a mistake when you make it again. -- Franklin P. Jones

Cryptoflex:
 * DFs and EFs in a DF have to be deleted in reverse order of creation.
   OpenSC relies on this fact for security, but also has some downsides. For    example, if you did a ``pkcs15-init -C`` and then added some EFs or DFs in the MF, you won't be able to do a ``pkcs15-init -E`` afterwards to remove the  PKCS15 DF (5015). So you'll first have to manually remove all EFs/DFs you created in the MF before being able remove the pkcs15 DF.

Starcos SPK 2.3:
 * Due to the way Starcos SPK 2.3 manages access rights it is necessary to manually call ``pkcs15-init --finalize`` after card personalization if no SO-PIN has been specified. Once the card has been finalized it is not possible to add new private/secret keys or PINs. If a SO-PIN is used the card will automatically be finalized after the SO-PIN has been stored.
 * If an SO-PIN is used and if there is enough space in the key file left, then the owner of the SO-PIN can access/use every protected item by creating a PIN for the necessary state.
