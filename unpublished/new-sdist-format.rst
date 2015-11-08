PEP: ??
Title: Static metadata for source distributions
Version: $Revision$
Last-Modified: $Date$
Author: Nathaniel J. Smith <njs@pobox.com>
Status: Deferred
Type: Standards-Track
Content-Type: text/x-rst
Created: 7-Nov-2015
Post-History:
Discussions-To: <distutils-sig@python.org>

Abstract
========

We define a wheel-inspired static metadata format for sdists, suitable
for tools like PyPI and pip's resolver.


Synopsis
========

There is currently no formal definition of what qualifies as a "source
distribution" or standard for storing static metadata within them,
which creates problems for tools like PyPI which wish to statically
extract metadata about source distributions for display to users, and
for tools like pip which wish to make inferences about which
combinations of wheels and source distributions can be installed
together.


Statement of deferral
=====================

The original motivation for writing the text in this PEP was that I
wanted to propose a way for sdists to include static metadata about
install-requirements, in the hopes that it would let us avoid having
to include a "egg_info" command equivalent in the new build system
interface. `This email has the gory details of why this matters
<https://mail.python.org/pipermail/distutils-sig/2015-October/027471.html>`_. (And
see also the overall `thread that that subthread was split out of
<https://mail.python.org/pipermail/distutils-sig/2015-October/027360.html>`_.)
But right now it's not obvious that a new sdist format will make a
difference to that discusson either way, and it might be better to
defer the sdist formalization in any case, so I'm putting it aside for
the moment and submitting it as a Deferred-status PEP so that it will
be available for someone to pick up later.

I think the text below is reasonably solid, but the main questions
that would need to be addressed if picking this up again are:

- There is a `proposal
  <https://mail.python.org/pipermail/distutils-sig/2015-October/027364.html>`_
  for allowing a single sdist to produce multiple distinct wheels
  (e.g. numpy-1.10.0.zip might produce numpy-1.10.0.whl and also
  numpy[mkl]-1.10.0.whl -- see the linked email for the gory
  details). The text below assumes that there is exactly one metadata
  directory per sdist (``numpy-1.10.0.sdist-info/``), and uses it for
  both sdist-wide metadata (``SDIST``, ``RECORD``, ...) and per-wheel
  metadata (``METADATA`` and eventually any other files that might be
  standardized in the wheel format, like ``entry-points.txt``). If we
  want to allow for multiple wheels to be generated from the same
  sdist, then we'll want to split out the sdist-wide metadata into its
  own directory.

- The text below provides a mechanism for some parts of the metadata
  to be marked "dynamic" (i.e. unavailable until a wheel is actually
  built). It's not 100% clear whether this is actually necessary (see
  again the email linked in the previous point); hopefully this will
  be clearer by the time someone picks this up again.

In addition, in case the specs around metadata and source trees have
settled down more by the time this standard is ready to be addressed,
then it might make sense to take the opportunity to require that
new-style sdists use new-style metadata and/or source trees. Or maybe
it doesn't matter.


Source distributions
====================

We retroactively declare the legacy source distribution format to be
"version 0". Roughly speaking, it consists of any file named
{PACKAGE}-{VERSION}.{EXT}, where {EXT} is a "well-known" archive
format like zip or tar.gz, and which unpacks into something that looks
more-or-less like a snapshot of a source tree. Its de facto
specification is encoded in the source code and documentation of
``distutils``, ``pip``, PyPI, and other tools.

A "version 1" (or greater) source distribution is a file meeting the
following criteria:

- It MUST have a name of the form: {PACKAGE}-{VERSION}.zip, where
  {PACKAGE} is the package name and {VERSION} is a PEP 440-compliant
  version number.

  (Rationale for only accepting .zip: in practice there's very little
  benefit to supporting multiple formats; it just creates confusion
  about what to use. Zip is chosen because it's universally supported,
  consistent with wheels, and it allows individual files to be
  accessed without decompressing the whole archive, which useful for
  tools like PyPI that want to extract metadata files but don't care
  about the rest of the archive.)

  The same filename handling rules apply as for wheels: runs of
  non-alphanumeric characters in the ``{PACKAGE}`` name are replaced
  by underscores, the archive filename itself is Unicode, and the
  filenames inside the archive are encoded as UTF-8. See `the
  "Escaping and Unicode" section of PEP 427
  <https://www.python.org/dev/peps/pep-0427/#escaping-and-unicode>`_
  for details.

- When unpacked, it MUST contain a single directory directory tree
  named ``{PACKAGE}-{VERSION}``.

- This directory tree MUST be a valid source tree as defined in PEP
  ??.

- It MUST additionally contain a directory named
  ``{PACKAGE}-{VERSION}.sdist-info`` (notice the ``s``), with the
  following contents:

  - ``SDIST``: Mandatory. Same record-oriented format as a wheel's
    ``WHEEL`` file, but with different fields::

      SDist-Version: 1.0
      Generator: setuptools sdist 20.1

    ``SDist-Version`` is the version number of this
    specification. Software that processes sdists should warn if
    ``SDist-Version`` is greater than the version it supports, and
    must fail if ``SDist-Version`` has a greater major version than
    the version it supports.

    ``Generator`` is the name and optionally the version of the
    software that produced the archive.

  - ``RECORD``: Mandatory. A list of all files contained in the sdist
    (except for the RECORD file itself and any signature files)
    together with their hashes, as specified in PEP 427.

  - ``RECORD.jws``, ``RECORD.p7s``: Optional. Signature files as
    specified in PEP 427.

  - ``METADATA``: Mandatory. Metadata version 1.1 or greater format
    metadata, with an additional rule that fields may contain the
    special sentinel value ``__SDIST_DYNAMIC__``, which indicates that
    the value of this field cannot be determined until build time. If
    a "multiple use field" is present with the value
    ``__SDIST_DYNAMIC__``, then this field MUST occur exactly once,
    e.g.::

       # Okay:
       Requires-Dist: lxml (> 3.3)
       Requires-Dist: requests

       # no Requires-Dist lines at all is okay
       # (meaning: this package's requirements are the empty set)

       # Okay, requirements will be determined at build time:
       Requires-Dist: __SDIST_DYNAMIC__

       # NOT okay:
       Requires-Dist: lxml (> 3.3)
       Requires-Dist: __SDIST_DYNAMIC__

    (The use of a special token allows us to distinguish between
    multiple use fields whose value is statically the empty list
    versus one whose value is dynamic; it also allows us to
    distinguish between optional fields which are statically not
    present versus ones whose value is dynamic.)

    When this sdist is built, the resulting wheel MUST have metadata
    which is identical to the metadata present in this file, except
    that any fields with value ``__SDIST_DYNAMIC__`` in the sdist may
    have arbitrary values in the wheel.

    A valid sdist MUST NOT use the ``__SDIST_DYNAMIC__`` mechanism for
    the package name or version (i.e., these must be given
    statically), and these MUST match the {PACKAGE} and {VERSION} of
    the sdist as described above.

    An sdist SHOULD NOT use the ``__SDIST_DYNAMIC__`` mechanism any
    more than necessary. If the value of some field is known to be the
    same in all generated wheels, then that value should be provided
    statically in the sdist metadata rather than using
    ``__SDIST_DYNAMIC__``.

    [TBD: do we want to forbid the use of dynamic metadata for any
    other fields? I assume PyPI will enforce some stricter rules at
    least, but I don't know if we want to make that part of the spec,
    or just part of PyPI's administrative rules.]

This is intentionally a close analogue of a wheel's ``.dist-info``
directory; intention is that as future metadata standards are defined,
the specifications for the ``.sdist-info`` and ``.dist-info``
directories will evolve in synchrony.


Copyright
=========

This document has been placed in the public domain.
