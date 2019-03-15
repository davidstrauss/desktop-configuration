# GPG Smart Cards with Only Sub-Keys

## Create a Certification/Master Key

1. Start creating a new key:

       gpg2 --expert --full-generate-key

1. Choose "RSA (set your own capabilities)"
1. Disable all but "Certify."
1. Choose 3072 bits.
1. Back up the revocation certificate from `~/.gnupg/openpgp-revocs.d/`.
1. Consider this key ID to be `$CERTKEY`.
1. Export the keypair to a backup drive:

       gpg2 --export-secret-keys $CERTKEY > my-private-key.asc

1. Remove the secret key from the local machine:

       gpg2 --delete-secret-key $CERTKEY

## Provision a New Smart Card

1. YubiKey Only: Reset the GPG module:

       ykman openpgp reset

1. YubiKey Only: Set the mode:

       sudo ykpersonalize -m6

1. Set the PIN and admin PIN:

       gpg2 --change-pin  # Change both the PIN (default is 123456)
                          # and the Admin PIN (default is 12345678).
                          # I use pwgen for the admin PIN.

1. YubiKey 4+ Only: Configure the key to use RSA with 3072 bits:

       gpg2 --card-edit
       gpg/card> admin
       gpg/card> key-attr

1. Add the new keys as subkeys on the card:

       gpg2 --edit-key $CERTKEY
       gpg> addcardkey  # Repeat for signing, encryption, and authentication.

## Certify an Existing Smart Card

@TODO
