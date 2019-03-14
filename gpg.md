# Using a GPG Master Key with GPG Smart Cards

## Create a Master Key to Certify Sub-Keys

1. Start creating a new key:

       gpg2 --expert --full-generate-key

1. Choose "RSA (set your own capabilities)"
1. Disable all but "Certify."
1. Choose 3072 bits.
1. Back up the revocation certificate from `~/.gnupg/openpgp-revocs.d/`.
1. Export the keypair to a backup drive:

       gpg2 --export-secret-keys $ID > my-private-key.asc

1. Remove the secret key from the local machine:

       gpg2 --delete-secret-key $ID

## Certify an Existing Smart Card

## Provision a New Smart Card

