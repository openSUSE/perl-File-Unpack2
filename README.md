# File::Unpack2

[![linux](https://github.com/openSUSE/perl-File-Unpack2/actions/workflows/linux.yml/badge.svg)](https://github.com/openSUSE/perl-File-Unpack2/actions/workflows/linux.yml)

An aggressive, mime-type based archive unpacker for [Cavil](https://github.com/openSUSE/cavil).

Cavil reviews the licensing of whole Linux distributions, and to do that it first has to get at the text
inside every source package. `File::Unpack2` is the component that turns an arbitrary archive into a tree of
plain files. It is *aggressive*: it identifies files by their **mime type** rather than their name, and it
descends recursively into everything it produces, so a tar wrapped in a zip wrapped in an rpm is peeled all
the way down — never fooled by a misleading or missing suffix.

- **Mime-type driven, not suffix driven.** Detection layers [`File::LibMagic`](https://metacpan.org/pod/File::LibMagic)
  (the `libmagic` engine behind `file`) with [`File::MimeInfo::Magic`](https://metacpan.org/pod/File::MimeInfo::Magic)
  and a little extra logic for formats that carry no usable magic (raw LZMA).
- **Recursive by design.** Every unpacked file that looks like an archive is unpacked again, up to a safety
  depth limit — the goal is to expose *all* readable payload, not just the top layer.
- **Pluggable helpers.** Most formats are handled by built-in helpers wrapping the usual tools (`tar`, `unzip`,
  `rpm2cpio`, `7z`, `unrar`, …); new formats are added as tiny external scripts named after their mime type.
  Bundled helpers ship *inside* the module and are found automatically.
- **Hardened against hostile input.** Scanning a distribution means ingesting binaries, malformed samples and
  archive "bombs". File::Unpack2 enforces optional caps on file count, total bytes and per-helper runtime,
  watches helpers for stalls, jails them inside the destination, and passes memory limits to the `xz`/`lzma`
  decoders — so a single bad input is logged and stepped over, never allowed to hang or exhaust the machine.
- **Precise logging.** Every mime type detected and every unpack action is recorded to a JSON (or plain) log.

`File::Unpack2` is released to CPAN but is primarily developed by the SUSE team as a dependency of Cavil, and
is packaged automatically for the SUSE Linux distributions.

See [`docs/Architecture.md`](docs/Architecture.md) for the full design and rationale.

## Install

```
perl Makefile.PL
make
make test
sudo make install
```

The Perl prerequisites are pulled in by `Makefile.PL`. Unpacking itself shells out to the usual command line
tools (`tar`, `xz`, `unzip`, `rpm2cpio`, `cpio`, `7z`, `unrar`, `cabextract`, `pdftotext`, …); install the ones
you need for the formats you care about. `libmagic` and the freedesktop `shared-mime-info` database power
mime-type detection.

## Synopsis

As a library:

```perl
use File::Unpack2;

# Recursively unpack an archive into a destination directory, capturing a JSON
# log of everything produced.
my $log;
my $u = File::Unpack2->new(logfile => \$log, destdir => '/tmp/out');
$u->unpack('inputfile.tar.bz2');
print "$1\n" while $log =~ m{^\s*"(.*?)":}g;   # every unpacked file

# Just identify a file's mime type (not fooled by the suffix).
my $m = $u->mime('/etc/init.d/rc');
print "$m->[0]; charset=$m->[1]\n";            # text/x-shellscript; charset=us-ascii
```

From the command line:

```
# Unpack one level into the current directory
file_unpack2 example.tar.gz

# Recursively unpack everything under src/ into /tmp/out
file_unpack2 --deep -D /tmp/out src/

# Just report the mime type, like `file -i`
file_unpack2 -m mystery.bin

# List the wired-up mime helpers
file_unpack2 -l
```

Run `file_unpack2 --help` for the full option list.

## Layout

```
lib/File/Unpack2.pm                     The module: mime detection, dispatch, recursion, hardening
lib/File/Unpack2/resources/helper/      Bundled external mime helpers (installed with the module)
script/file_unpack2                     Command line front-end
t/                                      Test suite, including adversarial / bomb fixtures
docs/Architecture.md                    Design and rationale, in prose
```

## Adding a mime helper

A helper is an executable named after the mime type it handles, with `/` written as `=` (an `x-` or `ANY+`
prefix after the `=` is implied). It is run inside the freshly-created output directory and receives the source
path, a suggested destination name, the destination directory, the mime type, the description and a config
directory. Dropping such a script into a directory named with `FILE_UNPACK2_HELPER_DIR` (or passed as
`helper_dir` to `new`) is all it takes:

```
$ echo 'ar x "$1"' > "$FILE_UNPACK2_HELPER_DIR/application=x-debian-package"
$ chmod a+x "$FILE_UNPACK2_HELPER_DIR/application=x-debian-package"
```

See the `unpack` and `mime_helper_dir` documentation in `perldoc File::Unpack2`, and
[`docs/Architecture.md`](docs/Architecture.md), for the full helper protocol.

## License

Copyright (C) 2010-2013 Juergen Weigert, (C) 2023-2026 Sebastian Riedel.

Free software, released under the same terms as Perl itself (GNU General Public License or Artistic License).
See [`LICENSES/`](LICENSES) and <https://dev.perl.org/licenses/>.
