ca-certificates-whitelist-filter
''''''''''''''''''''''''''''''''

Rather simple script to process p11-kit_ Certificate Authority bundles
for "ca-certificates" package in linux distros, and only leave explicitly
whitelisted certs there, removing the rest.

This allows to control which TLS Certificate Authorities are trusted
by all applications and tools system-wide on a whitelist-basis.

Modern Web PKI requires only a few CAs for almost all websites (as of 2023),
and others on that list are unlikely to be used in non-bogus ways, so nothing
beyond few top CAs is realistically worth "trusting" for every TLS connection.
p11-kit only allows blacklisting CAs, which doesn't work for "these few and no
others" approach, and is not safe against junk-CAs added upstream in the future.

Despite same name between linux distributions, ca-certificates package tends
to be somewhat custom per-distro (e.g. in Arch_, Fedora_, OpenSUSE_),
so maybe scripts in some of those might not have this issue, and allow
whitelisting already in their wrapper tools - make sure to check there first.

Note that there are other ways to address the issue of having too many
untrustworthy or unnecessary CAs, with always some more in the works,
so I'd recommend also looking into alternative options for a specific use-case,
as this is a rather blunt approach.

Related `"Trimming-down list of trusted TLS ca-certificates" blog post`_
goes into a bit more detail of what this script is meant to do.

Repository URLs:

- https://github.com/mk-fg/ca-certificates-whitelist-filter
- https://codeberg.org/mk-fg/ca-certificates-whitelist-filter
- https://fraggod.net/code/git/ca-certificates-whitelist-filter

.. _p11-kit: https://p11-glue.github.io/p11-glue/
.. _Arch: https://gitlab.archlinux.org/archlinux/packaging/packages/ca-certificates
.. _Fedora: https://src.fedoraproject.org/rpms/ca-certificates/tree/rawhide
.. _OpenSUSE: https://github.com/openSUSE/ca-certificates
.. _"Trimming-down list of trusted TLS ca-certificates" blog post:
  https://blog.fraggod.net/2023/12/28/trimming-down-list-of-trusted-tls-ca-certificates-system-wide-using-a-whitelist-approach.html


Usage
-----

It is a python 3.8+ script, with an optional dependency on `cryptography.io module`_,
only used for printing certificate fingerprints and attributes nicely in ``-l/--list``
or ``-L/--list-all`` modes (will be simply omitted without the module).

To work in normal mode, script requires a whitelist file, and changes files in
"ca-certificates/trust-source" directory (``/usr/share/ca-certificates/trust-source/``
by default).

Whitelist filters CAs by "label:" attribute from p11-kit bundles, allows using
shell-glob wildcards, #-comments, and can look something like this::

  Baltimore CyberTrust Root # CloudFlare
  ISRG Root X* # Let's Encrypt
  GlobalSign * # Google
  DigiCert *
  Sectigo *
  Go Daddy *
  Microsoft *
  USERTrust *

These labels can be listed using ``-L/--list-all`` tool option, for example:
``./ca-certs-whitelist-filter -L``

If `"cryptography" python module`_ is installed, it will also list X.509 cert
fingerprints and all attributes (organizationName, commonName, countryName, etc).

To actually do the filtering, script should be run with ``-w/--whitelist`` option::

  ./ca-certs-whitelist-filter -w my-roots.conf

(add optional trust-dir argument to use local test-dir instead of system-wide one)

Script makes a backup of the original CA bundles, and can be re-run without
clobbering the backups, to further filter remaining CAs in the produced bundle.

Browsers like firefox use this bundle dir directly, so should pick the change
up immediately in both old and new tabs, while other apps might also need running
``update-ca-trust`` distro script to export those into e.g. ``/etc/ssl/cert.pem``
for OpenSSL and other libs/tools.

It is intended to be run automatically on linux from a package manager hook like
this (Arch Linux pacman hook example)::

  [Trigger]
  Operation = Install
  Operation = Upgrade
  Operation = Remove
  Type = Path
  Target = usr/share/ca-certificates/trust-source/*

  [Action]
  Description = Filtering ca-certificates...
  When = PostTransaction
  Exec = /usr/local/bin/ca-certs-whitelist-filter -w /etc/ca-certificates/whitelist.conf

So that this filtering policy will get re-applied automatically on all package
updates that install/update/remove ca-certificates bundles.

To work properly, such hook must either be invoked before other hook(s) that run
``update-ca-trust`` or similar export-script, or hook should run that itself as well.

.. _cryptography.io module: https://cryptography.io
.. _"cryptography" python module: https://cryptography.io
