# Changelog

All notable changes to libgit2-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-15

The first release: fifty-two entry points of the libgit2 C API, one
`@ffi` declaration each, and no logic.

### Added

- `libgit2` — the whole surface, in eight groups.
  - The library: `git_libgit2_init`, `git_libgit2_shutdown` and
    `git_libgit2_version`.
  - Errors: `git_error_last` and `git_error_clear`.
  - The repository: open, init, free, the two paths, the two
    questions and `git_repository_head`.
  - References: free, the two names and the target.
  - Object identifiers: `git_oid_fromstr`, `git_oid_tostr`,
    `git_oid_cmp` and `git_oid_is_zero`.
  - Objects: lookup, free, type, identifier and the type's name.
  - Commits, trees and blobs: the three lookups and the accessors for
    the message, the summary, the time, the tree, the parents, the
    entries and the contents.
  - The revision walk: create, free, the two starting points, the
    step and the ordering.
- `tests/libgit2_tests.nv` — eight tests over the fifty-two entry
  points. They call the C library, so they need libgit2 installed.

### Named as missing

**Passing a structure by value, and writing into one the caller lays
out.** The novo-lang foreign function interface passes integers, floats
and strings. That rules out three large parts of libgit2 at once.

- **Signatures.** `git_signature` is a structure, and every call that
  takes or answers one is absent: `git_commit_author`,
  `git_commit_committer` and `git_signature_now` among them. Without
  them a commit cannot be created, so `git_commit_create` is absent
  too, and this package reads history rather than writing it.
- **Options structures.** `git_clone_options`, `git_checkout_options`,
  `git_status_options`, `git_diff_options` and their neighbours are
  structures the caller fills in, so cloning, checking out, the status
  listing and the diff are absent.
- **`git_buf`.** The calls that answer a growable buffer —
  `git_repository_item_path`, `git_message_prettify`, the patch
  printers — take a `git_buf` the caller owns.

**Every entry point that takes a C function pointer.** The tree walk,
the status callback, the diff callbacks, the remote transfer progress
and the credential callback are absent, because a novo-lang function is
not a C function pointer.
