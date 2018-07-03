# git-sign: sign files in Git repositories securely

While Git supports signing via `git tag -s` (or `git tag -u`), those
are signing SHA-1 hashes only as of today, hence a bit insecure given
that SHA-1 collisions aren't very expensive (about $75k according to
recent [news](https://duckduckgo.com/html/?q=sha-1%20collisions); a
SHA-1 collision still doesn't allow to MITM directly (pre-image attack
would be needed), but given the right circumstances could still make
attacks easier).

`git-sign` is a wrapper that adds SHA-256 hashes of all files to the
tag message. `git-sign` then runs `git tag -s` to create a normal
signed git tag that just happens to also contain SHA-256 hash sums in
plain text. For an example tag message, see
[the git-sign v1 tag](https://github.com/pflanze/git-sign/releases/tag/v1)
(or run `git clone https://github.com/pflanze/git-sign; cd git-sign;
git cat-file -p v1`). Since the tag message is part of what the PGP
signature signs, it cannot be altered, assuming a recent enough
PGP/GnuPG key. The SHA-256 hashes in the tag message (as well as the
number of files) can then be verified to be the same as those checked
out by Git. Hence, content verification does not depend on Git hashes.

`verify-sig` does all of the verification steps of a git tag (it also
verifies that the PGP signature itself does not use known to be
insecure hashing algorithms).

The two programs are simple bash scripts so that you can verify their
source code easily enough. Also, the signed tag messages mention the
code that was run to generate them, hence the integrity can be
verified manually, without even installing verify-sig, by running `git
tag -v $tagname` then running the two code fragments and comparing the
output (but note that unlike `verify-sig`, `git tag -v` does not (to
my knowledge) verify what hash algorithm the PGP signature uses!)

Note that while this ensures that the checked-out files (in the
working directory) are of identical number and content as those the
author (signatory) had, the Git *history* might still have been
subverted by an attacker. In other words, `verify-sig` will only give
you assurance that the current checkout is fine, which is useful
enough to make secure software releases by using Git tags (with
`git-sign`), i.e. they can be built by users without risk, but it will
not give assurance for working with the Git repo arbitrarily. Even the
commits leading up to one that `verify-sig` confirmed as OK are not
assured to be pristine. For this, Git itself has to move to a more
secure hashing algorithm.

There are a couple additional utilities bundled with the repository:

* `git-tag-hashed $tag` shows `$tag:$sha256sum_of_tag_message`,
    i.e. gives a string that uniquely identifies the files to be
    checked out in a more secure way than a Git sha1 would. (More
    precisely the hash is formed from the tag message excluding any
    GPG signature, if any.)

* `checkout-signed $tag:$sha256sum_of_tag_message` takes a string as
    printed by `git-tag-hashed` and checks out and verifies the
    identified files.

* (`take-while-re` is used by `git-tag-hashed`.)


## Usage

* **Prepare**: GnuPG version 1.x still uses SHA1 as signing algorithm
    by default. This of course would make usage of git-sign moot (it
    replaces reliance on Git's SHA1 hashes with reliance on GnuPG's SHA1
    hashs). `verify-sig` checks against this case and warns and aborts
    if it sees such signatures. Thus, it's necessary to either use GnuPG
    2, or configure GnuPG 1 to use more secure hashes, which can be done
    via adding the folloging to `~/.gnupg/gpg.conf` (see
    [gpg 1 configuration](gpg1configuration.md) for more info):
  
        personal-digest-preferences SHA256
        cert-digest-algo SHA256
        default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed

* **Sign**: `git-sign v10`, `git-sign v10 "release 10"`,
    `GITSIGNKEY=04EDB072 git-sign v10`: create a Git tag named `v10` on
    the current HEAD, adding hashes of all files in the current working
    directory. See `git-sign -h` for more details. Note that it uses
    hashes of the files in the working directory, not those stored in
    Git's HEAD, thus you need to run this on a clean working directory
    (new files not added to the index don't hurt, but deletions or
    modifications do). Just run immediately after a `git commit -a`, or
    verify with `git status` that the working directory is clean.

* **Verify**: `verify-sig v10`: verify the signature and file hashes on
    the tag `v10`. Similarly to `git-sign`, this assumes that the
    working directory contains the same files that the tag being checked
    refers to. Otherwise `verify-sig` will report the files that are
    different between the tag and HEAD or the working directory. To get
    clean verification, first check out the version pointed to by the
    tag that you want to verify, e.g. with `git checkout -b v10 v10` (or
    just `git checkout v10` if you're not confused by working with a
    detached HEAD).
    
    Warning: by default, `verify-sig` does not restrict the keys that
    are accepted for signatures as long as they are in the local keyring
    and have a trust path. If you're using `verify-sig` in scripts, you
    may want to use the `--accept-keys` option or
    `VERIFY_SIG_ACCEPT_KEYS` env variable. See `verify-sig --help`.


## Dependencies

Both scripts are relying on `git` and `gpg`, as well as `sha256sum`
and various other utilities from the GNU core utilities
(e.g. `coreutils` package on Debian), `egrep` from the GNU grep
package, and `perl`. Except for `git` and `gpg` these should be
available in the minimal install of standard Linux distributions.


## Bugs

Only the contents of the files is verified, not their executable
status. Also, symlinks are followed before hashing, which in the case
of symlinks pointing outside the working directory will probably not
be useful. Changing these would make the scripts considerably more
complex, though.


## See also

* [git-evtag](https://github.com/cgwalters/git-evtag): much more
  complex; easy verifiability of the source code of the checker, or
  even just manual checking, was the aim for git-sign.

* [git-secure-tag](https://github.com/indutny/git-secure-tag): in
  Node.js, also much more complex.


