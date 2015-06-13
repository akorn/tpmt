# tpmt
Team Password Management Tool

This is going to be a set of zsh scripts that allow a team to securely share passwords.

It'll be based on `seccure`.

No GUI is planned.

Goals:

 * it shouldn't get in the way;
 * everyday tasks should be very simple;
 * everything should be stored in text files for easy portability;
 * it shouldn't depend on anything excessive other than zsh and the small seccure package;
 * Devuan/Debian packaging scripts should be included.

## How it'll work

 * A password "database" is backed by a subversion repository (or some other RCS, but initially I'm only going to support subversion).
 * Every user has an Elliptic Curve (EC) public key.
 * The scripts will use the filesystem as a database. We'll have one subdir per database entry (effectively, per password), and use different files in those subdirs to store different kinds of data. This should scale to a few thousand entries (after that, it might be better to hash the subdirs).
 * In addition to these subdirs, we have a subdir called `pubkeys`, with one file per user, named after the user, which contains the EC pubkey of the user.
 * Storing a password means creating a new subdir for it, encrypting the password using the EC key of the adding user and storing the encrypted version in the subdir.
 * Granting access to a password involves decrypting the password with our secret key, then re-encrypting it with the pubkey of the other user.
  * Since passwords are short and EC is efficient, there is little point in adding another layer of encryption, with a symmetric cipher, as is commonly done with RSA.

Initially at least the tools don't aim to be secure against other local users on the computer they're being executed on. Having to run them on a single-user system shouldn't be an impractical limitation.

## Use-cases:

 * Initialize a new password repository.
  * This is just `mkdir passwords pubkeys`
 * Add user (add pubkey).
  * Something like `seccure-key -d -q >pubkeys/username`
 * Update pubkey.
 * Revoke a pubkey due to compromise (and replace it with a new one).
  * This should force a password change on all passwords accessible for that pubkey (which is potentially a nightmare but the only secure thing to do).
   * It may be possible that an affected password can't be changed immediately for whatever reason (e.g. because the system it belongs to is inaccessible). Therefore, we must support a system of passwords flags, such as "must-change-asap". The simplest solution seems to be to create a `flags/` subdir under each password that has flags, and every file inside it is a flag. The contents of the file can be some explanation for why the flag was set.
 * Add new password.
  * The new password gets a new unique ID assigned to it.
  * We create a new subdir named after this ID.
  * Data to record:
   * keywords ("attic linksys wrt54g admin") -- these go into `subdir/keywords`, one per line
    * and maybe we could have a `subdir/kw` directory too, with one entry per keyword? This requires some sanitization of keywords though.
   * comments -- these go into `subdir/comments/currentuser` (because another user may add another comment)
   * name of user who added the password -- written into `subdir/creator` and `subdir/last-changed-by` (not sure if we actually need this data)
    * both of these could be symlinks instead of real files, which would save some space but would make grepping more difficult
   * the password itself (encrypted with the current user's public key)
    * this can be done with a command line like `echo plainpass | seccure-encrypt -f pubkeys/currentuser >subdir/users/currentuser)`
   * date and time 
    * written to `subdir/stamp-creation` and `subdir/stamp-change`
   * the script should print out the unique ID of the new password before exiting
   * ideally the unique ID should be constructed from the keywords and not be totally random
 * Retrieve password.
  * Either by ID or by keyword (if there are multiple matches, either query interactively or print all matching passwords).
  * Ideally we'd also be able to copy the decrypted password to the X clipboard without printing it.
  * example command line: `seccure-decrypt -q -f <subdir/users/currentuser`
 * Update/change password. Same as adding; the operation should be different because we want to avoid the same password being added twice with different keywords.
  * The update-password operation doesn't change password access privileges.
  * This boils down to enumerating who had access to the old password, then re-encrypting the new password with the pubkey of all relevant users.
  * `subdir/stamp-change` should be updated as well.
  * Changing keywords or comments doesn't require script support; it can be done perfectly well manually (editing the comment/keyword files).
 * Grant password access rights.
  * (Originally I thought it would make sense to have two privilege levels: read-only and write. However, since we're going to keep this in svn (or, hey, maybe even git?), that doesn't make much sense; we can restrict write access in svn if necessary. Also keep in mind that in most cases people who know a password can also change it in the system that uses it; it would be stupid to prevent them from updating the password in the "database".)
  * Granting access means decrypting the password (if we can), then re-encrypting it with the pubkey of every user who should have access.
  * So, parameters: password selector (keywords or ID), username(s).
  * Audit trail is preserved in svn; the commit should happen automatically and use a message of a well-defined format.
   * Problem: what if the commit fails?
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
