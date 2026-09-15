# libgit2-sys

libgit2 is a portable implementation of the Git core methods, provided
as a C library with no dependency on the `git` command line tool. It
reads and writes the same repositories `git` does: the same object
store, the same packfiles, the same references. Its interface is
documented in [the libgit2 reference](https://libgit2.org/docs/reference/main/).
This package declares fifty-two of that library's entry points to
novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libgit2. The package contains no logic of
its own, and it does nothing without the C library installed. The
fifty-two entry points read a repository: its references, its commits,
its trees and its file contents. The section "What is not included"
says what a program still cannot do with them alone, and the first item
is writing a commit.

## What it is

A Git repository is a **content-addressed object store**. Every object
is named by the hash of its own bytes, so a name and the thing it names
cannot disagree.

There are four kinds of object. A **blob** is the contents of one file,
with no name and no permissions. A **tree** is a directory listing:
names, permissions and the identifier of the blob or tree each name
points at. A **commit** names one tree, zero or more parent commits, an
author, a committer and a message. A **tag** is an annotated name for
another object.

An **object identifier** is the hash. In this package it is twenty raw
bytes, not text, and it is not a handle: the caller reserves twenty
bytes with `ptr.alloc(20)`, and an entry point that answers an
identifier answers the address of twenty bytes the object owns.
`git_oid_tostr` turns one into the forty hexadecimal digits a person
reads.

A **reference** is a name that points at an object, such as
`refs/heads/main`. A **direct** reference holds an object identifier. A
**symbolic** reference points at another reference, and `HEAD` is
usually one of those.

A **revision walk** visits the commits reachable from a starting point.
It is how a history is read: push a commit onto the walk and take
identifiers out of it until it says it is finished.

A repository is **bare** when it has no working directory. The object
store then sits directly in the repository's own directory rather than
in a `.git` subdirectory.

## Install

```
novo pkg add libgit2-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its headers come from the system package `libgit2-dev`:

```
sudo apt install libgit2-dev
```

On macOS the Homebrew formula is `libgit2`. On other systems the
library builds from the libgit2 source with CMake.

## Example

The subject line of the commit at the tip of the current branch:

```novo ignore
use libgit2

fn main() [io, ffi]
    // Nothing else in the library may be called before this.
    let _ = libgit2.git_libgit2_init()
    let repo_slot = ptr.alloc_word()
    let head_slot = ptr.alloc_word()
    let commit_slot = ptr.alloc_word()

    if libgit2.git_repository_open(repo_slot, ".") != 0
        let err = libgit2.git_error_last()
        println(ptr.read_str(ptr.read_word(err)))
        return
    let repo = ptr.read_word(repo_slot)

    // HEAD resolves to a reference, and the reference to an identifier.
    let _ = libgit2.git_repository_head(head_slot, repo)
    let head = ptr.read_word(head_slot)
    println("on " + ptr.read_str(libgit2.git_reference_shorthand(head)))

    let _ = libgit2.git_commit_lookup(commit_slot, repo, libgit2.git_reference_target(head))
    let commit = ptr.read_word(commit_slot)
    println(ptr.read_str(libgit2.git_commit_summary(commit)))

    libgit2.git_commit_free(commit)
    libgit2.git_reference_free(head)
    libgit2.git_repository_free(repo)
    ptr.free(commit_slot)
    ptr.free(head_slot)
    ptr.free(repo_slot)
    let _ = libgit2.git_libgit2_shutdown()
```

The example is fenced as an illustration rather than a compiled block
because it needs a repository with at least one commit, which this
package cannot create.

## What the package contains

| Module | Contents |
| --- | --- |
| `libgit2` | Every entry point, in eight groups: the library, errors, the repository, references, object identifiers, objects, commits with trees and blobs, and the revision walk. |

The eight groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Library | 3 | Starts and stops the library and reports its version. |
| Errors | 2 | Reads and clears the last failure on this thread. |
| Repository | 8 | Opens or creates a repository and describes it. |
| References | 4 | Names a reference and reads the identifier it points at. |
| Object identifiers | 4 | Parses, prints, compares and tests the twenty bytes. |
| Objects | 5 | Looks an object up whatever its kind, and reports its kind. |
| Commits, trees and blobs | 20 | Reads a commit's message, time, tree and parents, a tree's entries, and a blob's bytes. |
| Revision walk | 6 | Walks the history reachable from a starting point. |

## How to choose an entry point

`git_object_lookup` reads an object whose kind the program does not
know yet, and `git_object_type` then says what it is.
`git_commit_lookup`, `git_tree_lookup` and `git_blob_lookup` read one
whose kind is known, and refuse an object of another kind.

`git_reference_target` is the identifier a direct reference holds.
`HEAD` is symbolic, so `git_repository_head` is what resolves it: it
answers the reference `HEAD` points at, whose target is the commit.

`git_revwalk_push_head` starts a walk at the current branch, and
`git_revwalk_push` starts it at a commit the program names. Pushing
more than one starting point walks the union of the histories.

## The rules a user needs

1. **`git_libgit2_init` comes first and `git_libgit2_shutdown` comes
   last.** Both are counted, so nested initialisation is allowed, and
   nothing else in the library may be called outside the pair.
2. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
3. **An out-parameter is the address of a caller-owned slot.** Every
   lookup writes its handle into one, because the return value is the
   result code. `ptr.alloc_word` reserves one, `ptr.read_word` reads it
   back, and `ptr.free` releases it.
4. **An object identifier is twenty bytes, not a handle.** Reserve
   them with `ptr.alloc(20)`. An identifier an object answers belongs
   to that object and stops being valid when the object is freed.
5. **A result code is negative, and it arrives in 32 bits.** Write
   `as i32` on the answer before comparing it with a negative number.
   These are the codes this package's entry points give.

   | Code | Name | What it means |
   | --- | --- | --- |
   | 0 | — | success |
   | -1 | `GIT_ERROR` | a generic failure; `git_error_last` says more |
   | -3 | `GIT_ENOTFOUND` | the object or the repository is not there |
   | -9 | `GIT_EUNBORNBRANCH` | HEAD names a branch with no commits |
   | -31 | `GIT_ITEROVER` | the walk has no more commits |

6. **`git_error_last` answers a structure, not a string.** It is a
   `char *` message at offset 0 and a 32-bit class at offset 8. Read
   the message with `ptr.read_str(ptr.read_word(addr))`.
7. **Every handle has its own release.** A repository by
   `git_repository_free`, a reference by `git_reference_free`, a commit
   by `git_commit_free`, a tree by `git_tree_free`, a blob by
   `git_blob_free`, a walk by `git_revwalk_free`. `git_object_free`
   releases any of the four object kinds. A tree entry from
   `git_tree_entry_byindex` or `git_tree_entry_byname` belongs to the
   tree and is **not** released.
8. **A string an accessor answers belongs to the object.** It stops
   being valid when the object is freed. Copy it with `ptr.read_str`
   first.
9. **`git_oid_tostr` needs 41 bytes for a whole identifier.** Forty
   digits and a terminator. A shorter buffer answers a shortened
   identifier, which is not an error.
10. **`git_commit_summary` is the first paragraph, not the first
    line.** Line breaks inside the first paragraph become spaces.
11. **`git_repository_head` fails in a repository with no commits**,
    with `GIT_EUNBORNBRANCH`. That is the state of a repository just
    created, and it is not a broken repository.
12. **The object type numbers are 1 for a commit, 2 for a tree, 3 for a
    blob and 4 for a tag.** `GIT_OBJECT_ANY` is -2 and matches any of
    them.

## What is not included

- **Writing a commit.** `git_commit_create` needs a `git_signature` for
  the author and another for the committer, and a signature is a
  structure passed by value. The novo-lang foreign function interface
  passes integers, floats and strings, so the whole signature family is
  absent: `git_signature_now`, `git_signature_new`,
  `git_commit_author` and `git_commit_committer` among them. This
  package reads history; it does not write it.
- **Everything configured by an options structure.** Cloning, checking
  out, the status listing, the diff, the merge and the rebase all take
  a structure the caller fills in. `git_clone_options`,
  `git_checkout_options`, `git_status_options` and
  `git_diff_options` have no shape a binding can carry.
- **Everything that answers a `git_buf`.** `git_repository_item_path`,
  `git_message_prettify` and the patch printers write into a growable
  buffer structure the caller owns.
- **Every entry point that takes a C function pointer.** The tree walk,
  the status callback, the diff callbacks, the transfer progress and
  the credential callback are absent. A novo-lang function is not a C
  function pointer, and the credential callback is what makes an
  authenticated fetch impossible here.
- **Remotes and the network.** `git_remote_create`, `git_remote_fetch`
  and `git_remote_push` need the options and the callbacks above.
- **The index.** `git_index_add_bypath` and its neighbours are the
  staging area, which only matters to a program that writes.
- **The configuration and the reflog.** Both are left out of the first
  release.
- **SHA-256 repositories.** libgit2 1.7 builds them only with an
  experimental flag, and the object identifier is then thirty-two bytes
  rather than twenty. This package assumes twenty.

## Related packages

`git-nv` is the Git implementation written in novo-lang, with no C
library, and `git-core-nv` is the object store and packfile format
beneath it. libgit2 is the reference this port measures itself against.
Both are planned and not published yet.

Choose `git-nv` when the program must build for a microcontroller or
for WebAssembly, when a C toolchain is not wanted, or when the program
has to write. Choose this package when the program needs libgit2's own
behaviour on a large repository, or must agree with it exactly.

## Tests

`tests/libgit2_tests.nv` holds eight tests written against the
signatures. They call the C library, so `novo test` needs libgit2
installed and linkable:

```
novo test tests/libgit2_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The suite creates one bare repository under the system temporary
directory for each test that needs one, and leaves it there; the
standard library deletes only empty directories, and a repository is
not one.

A repository this suite creates has no commits, because writing one
needs a signature. So the commit, tree and blob accessors are written
out in full against a lookup that cannot succeed, which exercises their
signatures against the compiler where the library cannot be asked. What
the suite does assert against the library: that the version is reported,
that an identifier parses, prints and compares, that the three object
type names are `commit`, `tree` and `blob`, that opening a path with no
repository answers `GIT_ENOTFOUND` and leaves a message on the error
queue, that a bare repository has no working directory and reports
itself empty, that `HEAD` in it answers `GIT_EUNBORNBRANCH`, and that a
walk over it answers `GIT_ITEROVER` at once.

## Implementation status

| Group | State |
| --- | --- |
| Library | Complete. |
| Errors | Complete for reading and clearing. |
| Repository | Complete for opening, creating and describing. |
| References | Complete for reading. Creating and renaming are absent. |
| Object identifiers | Complete for twenty-byte identifiers. |
| Objects | Complete. |
| Commits | Read-only. The author and committer are absent, and so is creating one. |
| Trees | Read-only, by index and by name. |
| Blobs | Read-only. |
| Revision walk | Complete. |
| Index, remotes, diff, status, merge | Absent. Each needs a structure or a callback. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

libgit2 itself is distributed under the GNU General Public License
version 2 with a linking exception, and installing it is the reader's
own step.
