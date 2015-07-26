# tpmt
Team Password Management Tool

This is going to be a set of zsh scripts that allow a team to securely share passwords, to act as a reference implementation of the password management system described here.

I hope eventually other people will contribute fancier implementations in other languages.

The crypto will be based on `seccure` (which uses elliptic curve cryptography).

No GUI is planned.

Goals:

 * it shouldn't get in the way;
 * everyday tasks should be very simple;
 * everything should be stored in text files for easy portability;
 * it shouldn't depend on anything excessive other than zsh and the small seccure package;
 * Devuan/Debian packaging scripts should be included.

## How it'll work

 * A password "database" is backed by a subversion repository (or some other RCS, but initially I'm only going to support subversion).
  * I'm going to assume the local checkout the script works in is always mostly up to date; various interesting things might happen if it is not (for example, we might encrypt a password with a revoked public key).
  * The point of using a revision control system is to support semi-distributed (but not necessarily heavily concurrent or disconnected) operation; people managing the passwords should be able to do so on their own single-user computers, but the password repository should be kept up to date in a central location.
 * Every user has an Elliptic Curve (EC) public key (or, rather, potentially several; see below).
 * The scripts will use the filesystem as a database.
  * Directory layout:
   * `passwords`. Under this directory, we'll have one subdir per database entry (effectively, per password), and use different files in those subdirs to store different kinds of data. This should scale to a few thousand entries (after that, it might be better to hash the subdirs).
   * `pubkeys`, with one directory per user, named after the user, which contains the EC pubkeys of the user.
    * Supporting more than one pubkey is necessary to support gradual migration from one pubkey to the other; see below.
    * The pubkey files will be named after the epoch second they were created in (so we know which of two keys is newer).
     * I'm going to call this the *keyid*.
    * We could have a symlink called `current` that points to the latest key; however, instead of using the symlinks it seems better to always find the latest key algorithmically, eliminating a potential source of inconsistency. 
    * `pubkeys/username` should always only contain non-revoked keys; revoked keys get moved to `revoked-keys/username`.
  * `revoked-keys`: same layout as `pubkeys`, for public keys that have been revoked due to compromise or because their owners should no longer have access to any passwords (e.g. because they quit).
  * `lost-keys`: same layout as `pubkeys`, for public keys whose passphrase is no longer known.
   * Tracking these allows us to track what passwords we may no longer be able to access.
 * Keys live forever. It doesn't make sense to ever remove a public key, even after it's been revoked (since the old version could always be pulled out of the RCS).
 * Storing a password means creating a new subdir for it, encrypting the password using the current EC key of the adding user and storing the encrypted version in the subdir.
 * Granting access to a password involves decrypting the password with our secret key, then re-encrypting it with the current pubkey of the other user.
  * Since passwords are short and EC is efficient, there is little point in adding another layer of encryption, with a symmetric cipher, as is commonly done with RSA.
 * It'll be possible to have a "global master" account that has access to all passwords, but this isn't going to be enforced.

At least initially, the tools don't aim to be secure against other local users on the computer they're being executed on. Having to run them on a single-user system shouldn't be an impractical limitation.

## Use-cases:

 * Initialize a new password repository.
  * This is just `mkdir passwords pubkeys revoked-keys`, plus revision control.
 * Add user (add pubkey).
  * Something like `seccure-key -d -q >pubkeys/username/$EPOCHSECONDS` (but obviously query EPOCHSECONDS first, then save it in a variable).
   * An svn pre-commit hook script can make sure people can only add their own pubkeys (svn usernames have to match `tpmt` usernames for this to work).
  * A user can always add a new pubkey. Subsequently updated/added passwords will be encrypted with the latest pubkey, but earlier pubkeys will still be accessible.
   * (Note that `seccure` typically generates different pubkeys even for the same passphrase, likely due to salting.)
   * When a user adds a new pubkey, we should immediately re-encrypt all passwords encrypted with any of the older keys with the new key.
    * It is possible for "dangling" keys to be left if the user doesn't know the password for one of their old pubkeys.
     * This is not a problem as long as there are other users who can access those passwords; we'll prompt these users to re-encrypt these passwords with the new pubkey of the forgetful user.
     * Dangling passwords that we can't regain access to will have to be recovered out of band.
     * The "lint" operation should list passwords that are only accessible to a pubkey that is not the latest pubkey of a user.
     * The "lint" operation should prompt to re-encrypt passwords that are currently encrypted with a pubkey that is not the latest pubkey of a user.
     * The "lint" operation should prompt to revoke old pubkeys that are no longer used to encrypt any passwords.
      * Potential problem: someone working with an outdated working copy may still add a password and encrypt it with this revoked key. Do we care?
   * Adding a new pubkey will thus be somewhat analogous to "changing the master password" in a conventional single user password manager.
    * However, if you have revision tracking, the RCS will still contain the versions of the encrypted passwords that were encrypted with the previous pubkey; thus, to truly change your "master password", you'd also need to muck with the RCS (not recommended) or change all the individual passwords. This is definitely a drawback of revision tracking in this instance, which I think is offset by better support for multi-user operation.
   * If your pubkey was compromised, you have to assume that all passwords that were encrytped with it were also compromised, and instead of just adding a new pubkey you should also revoke the old one.
    * Challenge: the "revoke" operation must involve adding a new pubkey, but it must also enforce (to the extent possible) that all affected passwords be changed.
     * Moving the revoked key to `revoked-keys` takes care of this, because the "lint" operation can check for passwords encrypted with revoked keys and prompt to change them.
   * It is not necessary to support adding a new pubkey for a user other than yourself (the svn pre-commit hook script can enforce this).
   * Challenge: what if person A has forgotten their passphrase, and while there are no passwords that only they had access to, there is no person B who has access to all the passwords person A can access? I.e. if you need more than one additional user to access all the passwords of person A?
    * We can gradually phase out a pubkey whose passphrase was lost. First, person B re-encrypts all passwords of person A they have access to with the new pubkey. Then person C does the same with the remaining passwords (and if neessary, so does person D etc.).
     * If there are passwords we won't be able to access again (because we only have versions of them encrypted with public keys we don't have the corresponding passphrase to), we should keep them around and "lint" should list them.
      * We need a "forget" or similar operation that moves a pubkey into `lost-keys`.
      * The "lint" operation needs to check for passwords encrypted with lost keys and prompt to re-encrypt them.
 * Revoke a pubkey due to compromise (and optionally replace it with a new one).
  * This should force a password change on all passwords accessible for that pubkey (which is potentially a nightmare but the only secure thing to do).
   * It may be possible that an affected password can't be changed immediately for whatever reason (e.g. because the system it belongs to is inaccessible). See below for how we can handle that.
   * Revoking has to be performed by a person who has access to all the passwords the revoked pubkey has access to; otherwise re-encryption can't take place.
    * Of course, there may not *be* such a single person (other than the person whose pubkey is being revoked). Thus, revocations must also be possible in multiple stages.
    * The specific revocation procedure works as follows:
     1. We move the key to be revoked (`pubkeys/username/keyid`) to `revoked-keys/username/keyid`.
      * We create a `revoked-keys/username/keyid.revocation-stamp` file with the time the key was revoked (the current time).
       * This will allow us to find passwords that haven't been changed since the time the revoked key had access to them.
      * We *could* create a `revoked-keys/username/keyid.pwlist` file with a list of the IDs of all passwords the revoked key has access to at the time of revocation.
       * However, the repository may exist in another state elsewhere, with the revoked key having access to a password we don't even know about; thus creating such a file would only introduce potential inconsistency.
     2. We trigger a "lint" operation.
      * "lint" has to enumerate revoked keys, then check if they have access to any passwords that the current user also has access to. If it finds any, it must prompt to change and re-encrypt them.
       * If you want to revoke all access from a user, you must revoke all their keys. If a user has no non-revoked, non-lost keys, the lint procedure won't be able to grant them acccess to changed passwords.
 * Add new password.
  * The new password gets a new unique ID assigned to it.
   * The user can specify the ID (to make it mnemonic), but we can generate one from the keywords provided.
  * We create a new subdir named for this ID.
  * Data to record:
   * keywords ("attic linksys wrt54g admin") -- these go into `subdir/keywords`, one per line
    * and maybe we could have a `subdir/kw` directory too, with one entry per keyword? This requires some sanitization of keywords though. Keeping these in sync would be nigh impossible. So, no, don't do this.
   * comments -- these go into `subdir/comments/currentuser` (because another user may add another comment)
   * name of user who added the password -- written into `subdir/creator` and `subdir/last-changed-by` (not sure if we actually need this data)
    * both of these could be symlinks instead of real files, which would save some space but would make grepping more difficult; so don't do this.
   * the password itself (encrypted with the current user's current public key)
    * this can be done with a command line like `echo plainpass | seccure-encrypt -f pubkeys/currentuser/keyid >subdir/users/currentuser/keyid)`
   * date and time 
    * written to `subdir/stamp-creation` and `subdir/stamp-change`
   * the script should print out the unique ID of the new password before exiting
 * Retrieve password.
  * Either by ID or by keyword (if there are multiple matches, either query interactively or print all matching passwords).
  * Ideally we'd also be able to copy the decrypted password to the X clipboard without printing it, maybe using xclip: https://www.debian-administration.org/article/565/Using_the_X_clipboard_from_the_command_line
  * example command line: `seccure-decrypt -q -f <subdir/users/currentuser/keyid`
 * Update/change password. Same as adding; the operation should be different because we want to avoid the same password being added twice with different keywords.
  * The update-password operation doesn't change password access privileges.
  * This boils down to enumerating who had access to the old password, then re-encrypting the new password with the pubkey of all relevant users.
  * `subdir/stamp-change` should be updated as well.
  * Changing keywords or comments doesn't require script support; it can be done perfectly well manually (editing the comment/keyword files).
 * Grant password access rights.
  * (Originally I thought it would make sense to have two privilege levels: read-only and write. However, since we're going to keep this in svn (or maybe even git), that doesn't make much sense; we can restrict write access in svn if necessary. Also keep in mind that in most cases people who know a password can also change it in the system that uses it; it would be stupid to prevent them from updating the password in the "database".)
  * Granting access means decrypting the password (if we can), then re-encrypting it with the pubkey of every user who should have access.
  * So, parameters: password selector (keywords or ID), username(s).
  * Audit trail is preserved in svn; the commit should happen automatically and use a message of a well-defined format.
   * Problem: what if the commit fails? We'll have to let the user deal with it.
 * Revoke password access right.
  * This means choosing a new password, then re-encrypting it with the pubkey of every user *except* the one(s) whose access is being revoked.
   * Choosing a new password is necessary because we don't know whether someone who technically could hasn't already viewed a password; also, since we're storing all data in a versioning system, the old password could still be retrieved by the person who now no longer has access (not to mention that they have access to their own older working copy as well).
    * However, while this is a secure default, it should be possible to force the password to remain unchanged, to accommodate possible real-world constraints regarding changing it.
  * Policy may require that at least some number of users have access to every password.
  * It would be stupid (misleading) to revoke your own access, because while you'd still possibly know the password, this wouldn't be apparent from looking at the "database". Therefore, revoking your own access should not be supported (if someone wants to, they can still do it, pro forma at least, manually and the only way we could prevent that would be by way of a complex pre-commit script).
 * Delete password.
 * Mass-grant access to all passwords matching some criteria (e.g. keywords).
 * (Meta)queries:
  * What passwords are stored at all?
  * "Who knows password X?"
   * `ls subdir/users`
  * "Which passwords does user Y know?" (Also: which passwords do I have access to?)
   * `for i in */users/Y; do echo ${i:h:h}; done`
  * Are there any passwords that match keyword foo that user X doesn't have access to? (If so, which ones?)
  * "When was password Z last changed?"
   * `date --date @$(cat subdir/stamp-change)`
  * "Which passwords are older than K days?" 
   * `for i in */stamp-change; do compute_age $i; [[ $age -gt K ]] && echo ${i:h}; done`
  * "Which passwords are known to more than J people?"
   * `for i in */users; do count=$(ls -1 $i | wc -l); [[ $count -gt J ]] && echo ${i:h}; done`
  * "Which passwords are only known to/accessible by user L?"
   * `for i in */users/L; do count=$(ls -1 $i | wc -l); [[ $count = 1 ]] && echo ${i:h}; done`
  * "Which passwords are accessible to less than M people?" 
   * `for i in */users; do count=$(ls -1 $i | wc -l); [[ $count -lt M ]] && echo ${i:h}; done`

Ideally, a pre-commit script should make sure that:

 * people only add pubkeys for themselves;
 * if a commit removes a pubkey, all passwords encrypted with that pubkey are also removed;
 * orphaned passwords (password dirs with no encrypted passwords in them) can't be committed;
 * people can't accidentally change other people's pubkeys (maybe require some magic string in the commit message to override);
 * if a pubkey is updated, all passwords encrypted with that pubkey should be re-encrypted (from svn's perspective, updated);
 * if a pubkey is compromised, all passwords it has access to have to be effectively ''changed'' as well;
 * a policy of the kind "every password has to be known by at least n people" can be enforced.

However, initially it should be fine if these sanity checks are performed by the scripts themselves, before auto-committing. Such a pre-commit check could also make sure the same password is not set in more than one place (by decrypting all passwords the current user has access to and counting the occurrence of each password).

A post-commit script may send email to people affected by changes:

 * to the people whose public keys were added, updated or revoked;
 * to the people who gained access to new passwords;
 * to the people who lost access to passwords;
 * to the people who have access to a password that was changed, or whose metadata was changed.
