# Subversion with write routes

This fork of Apache Subversion lets reads and writes reach the same
repository by different URLs, so writes can need a stronger login than
reads: a password typed every time, another SSH key, or a FIDO2 security
key (`ed25519-sk`) that has to be touched. Everything else is stock
Subversion.

The `write-routes` branch is based on the 1.15.0 release. Releases add a
number to the upstream version (1.15.0.1, 1.15.0.2, ...).

## Setting it up

In `~/.subversion/config`, name each route with a read URL prefix and a
write URL prefix:

```ini
[write-routes]
example = https://svn.example.com/ svn+ssh://svn-write/
```

and in `~/.ssh/config`, say which key the write route uses:

```
Host svn-write
    HostName svn.example.com
    User svn
    IdentityFile ~/.ssh/id_ed25519_sk
    IdentitiesOnly yes
    ControlMaster auto
    ControlPath ~/.ssh/cm-%C
    ControlPersist 60
```

Check out with the read URL. Updates, logs, diffs and `status -u` use the
read route. Commits, locks, unlocks and revision property changes open a
second connection, under the write prefix, the first time a command
writes. That's one SSH login, so one touch, per command, and
`ControlPersist` lets a quick run of commands share it.

Writes are the four repository calls that change anything: the commit
editor (`commit`, `import`, `mkdir`/`rm`/`cp`/`mv` on URLs, `svnmucc`,
`svnsync`, `svnrdump load`), `lock`, `unlock` and revision property
changes. Since this lives in the RA layer, every client built on
libsvn_ra gets it, including the language bindings.

## Safety

- **Same repository.** Both prefixes must reach the same repository. Its
  UUID and root URL are checked before the first write, and a mismatch
  stops the write.
- **Reads never break.** A bad route only stops writes, with an error
  that names it.
- **Nothing saved.** Credentials used on a write route are never saved.
- **Working copies stay on read URLs.** A URL under a write prefix is
  answered like a redirect to the read prefix:
  - a checkout of a write URL lands on the read URL;
  - `svn update` relocates a working copy that is on a write URL;
  - `svn relocate` to a write URL is refused.

So an svn without write routes, whether stock Subversion or a distribution
update that replaces this build, only ever has the read login. Make that
login read-only on the server, and such an svn can read but not write. It
fails with "Authorization failed" and never tries the write login.

## The server side

The server serves the same repository both ways. For example:

- reads over HTTPS, with logins that `mod_authz_svn` only lets read;
- writes over `svnserve -t` through SSH, with one `authorized_keys` line
  per key, such as
  `command="svnserve -t -r /srv/svn --tunnel-user=alice",restrict sk-ssh-ed25519@openssh.com AAAA...`.

`svnserve.conf` can point `authz-db` and `groups-db` at the same files
Apache uses. The repositories must be writable by both the web server and
the SSH account, for example through a shared group.

For a typed password instead, serve writes at a second location with its
own `AuthName`, so its realm differs from the read login's.

## Building and testing

The build is the usual `./autogen.sh && ./configure && make`. The write-route
tests are in `subversion/tests/libsvn_ra/ra-test` (the tests named
"write routes"). One of them reads through `file://` and writes through
svnserve over a tunnel.
