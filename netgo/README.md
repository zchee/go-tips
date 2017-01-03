From the `func isStale()` in [$GOROT/src/cmd/go/pkg.go](https://github.com/golang/go/blob/master/src/cmd/go/pkg.go).

# isStale and computeBuildID

## Theory of Operation

There is an installed copy of the package (or binary).  
Can we reuse the installed copy, or do we need to build a new one?

We can use the installed copy if it matches what we'd get  
by building a new one. The hard part is predicting that without  
actually running a build.

To start, we must know the set of inputs to the build process that can  
affect the generated output. At a minimum, that includes the source  
files for the package and also any compiled packages imported by those  
source files. The *Package has these, and we use them. One might also  
argue for including in the input set: the build tags, whether the race  
detector is in use, the target operating system and architecture, the  
compiler and linker binaries being used, the additional flags being  
passed to those, the cgo binary being used, the additional flags cgo  
passes to the host C compiler, the host C compiler being used, the set  
of host C include files and installed C libraries, and so on.  
We include some but not all of this information.

Once we have decided on a set of inputs, we must next decide how to  
tell whether the content of that set has changed since the last build  
of p. If there have been no changes, then we assume a new build would  
produce the same result and reuse the installed package or binary.  
But if there have been changes, then we assume a new build might not  
produce the same result, so we rebuild.

There are two common ways to decide whether the content of the set has  
changed: modification times and content hashes. We use a mixture of both.

The use of modification times (mtimes) was pioneered by make:  
assuming that a file's mtime is an accurate record of when that file was last written,  
and assuming that the modification time of an installed package or  
binary is the time that it was built, if the mtimes of the inputs  
predate the mtime of the installed object, then the build of that  
object saw those versions of the files, and therefore a rebuild using  
those same versions would produce the same object. In contrast, if any  
mtime of an input is newer than the mtime of the installed object, a  
change has occurred since the build, and the build should be redone.

Modification times are attractive because the logic is easy to  
understand and the file system maintains the mtimes automatically  
(less work for us). Unfortunately, there are a variety of ways in  
which the mtime approach fails to detect a change and reuses a stale  
object file incorrectly. (Making the opposite mistake, rebuilding  
unnecessarily, is only a performance problem and not a correctness  
problem, so we ignore that one.)

As a warmup, one problem is that to be perfectly precise, we need to  
compare the input mtimes against the time at the beginning of the  
build, but the object file time is the time at the end of the build.  
If an input file changes after being read but before the object is  
written, the next build will see an object newer than the input and  
will incorrectly decide that the object is up to date. We make no  
attempt to detect or solve this problem.

Another problem is that due to file system imprecision, an input and  
output that are actually ordered in time have the same mtime.  
This typically happens on file systems with 1-second (or, worse,  
2-second) mtime granularity and with automated scripts that write an  
input and then immediately run a build, or vice versa. If an input and  
an output have the same mtime, the conservative behavior is to treat  
the output as out-of-date and rebuild. This can cause one or more  
spurious rebuilds, but only for 1 second, until the object finally has  
an mtime later than the input.

Another problem is that binary distributions often set the mtime on  
all files to the same time. If the distribution includes both inputs  
and cached build outputs, the conservative solution to the previous  
problem will cause unnecessary rebuilds. Worse, in such a binary  
distribution, those rebuilds might not even have permission to update  
the cached build output. To avoid these write errors, if an input and  
output have the same mtime, we assume the output is up-to-date.  
This is the opposite of what the previous problem would have us do,  
but binary distributions are more common than instances of the  
previous problem.

A variant of the last problem is that some binary distributions do not  
set the mtime on all files to the same time. Instead they let the file  
system record mtimes as the distribution is unpacked. If the outputs  
are unpacked before the inputs, they'll be older and a build will try  
to rebuild them. That rebuild might hit the same write errors as in  
the last scenario. We don't make any attempt to solve this, and we  
haven't had many reports of it. Perhaps the only time this happens is  
when people manually unpack the distribution, and most of the time  
that's done as the same user who will be using it, so an initial  
rebuild on first use succeeds quietly.

More generally, people and programs change mtimes on files. The last  
few problems were specific examples of this, but it's a general problem.  
For example, instead of a binary distribution, copying a home  
directory from one directory or machine to another might copy files  
but not preserve mtimes. If the inputs are new than the outputs on the  
first machine but copied first, they end up older than the outputs on  
the second machine.

Because many other build systems have the same sensitivity to mtimes,  
most programs manipulating source code take pains not to break the  
mtime assumptions. For example, Git does not set the mtime of files  
during a checkout operation, even when checking out an old version of  
the code. This decision was made specifically to work well with  
mtime-based build systems.

The killer problem, though, for mtime-based build systems is that the  
build only has access to the mtimes of the inputs that still exist.  
If it is possible to remove an input without changing any other inputs,  
a later build will think the object is up-to-date when it is not.  
This happens for Go because a package is made up of all source  
files in a directory. If a source file is removed, there is no newer  
mtime available recording that fact. The mtime on the directory could  
be used, but it also changes when unrelated files are added to or  
removed from the directory, so including the directory mtime would  
cause unnecessary rebuilds, possibly many. It would also exacerbate  
the problems mentioned earlier, since even programs that are careful  
to maintain mtimes on files rarely maintain mtimes on directories.

A variant of the last problem is when the inputs change for other  
reasons. For example, Go 1.4 and Go 1.5 both install $GOPATH/src/mypkg  
into the same target, $GOPATH/pkg/$GOOS_$GOARCH/mypkg.a.  
If Go 1.4 has built mypkg into mypkg.a, a build using Go 1.5 must  
rebuild mypkg.a, but from mtimes alone mypkg.a looks up-to-date.  
If Go 1.5 has just been installed, perhaps the compiler will have a  
newer mtime; since the compiler is considered an input, that would  
trigger a rebuild. But only once, and only the last Go 1.4 build of  
mypkg.a happened before Go 1.5 was installed. If a user has the two  
versions installed in different locations and flips back and forth,  
mtimes alone cannot tell what to do. Changing the toolchain is  
changing the set of inputs, without affecting any mtimes.

To detect the set of inputs changing, we turn away from mtimes and to  
an explicit data comparison. Specifically, we build a list of the  
inputs to the build, compute its SHA1 hash, and record that as the  
``build ID'' in the generated object. At the next build, we can  
recompute the build ID and compare it to the one in the generated  
object. If they differ, the list of inputs has changed, so the object  
is out of date and must be rebuilt.

Because this build ID is computed before the build begins, the  
comparison does not have the race that mtime comparison does.

Making the build sensitive to changes in other state is  
straightforward: include the state in the build ID hash, and if it  
changes, so does the build ID, triggering a rebuild.

To detect changes in toolchain, we include the toolchain version in  
the build ID hash for package runtime, and then we include the build  
IDs of all imported packages in the build ID for p.

It is natural to think about including build tags in the build ID, but  
the naive approach of just dumping the tags into the hash would cause  
spurious rebuilds. For example, 'go install' and 'go install -tags neverusedtag'  
produce the same binaries (assuming neverusedtag is never used).  
A more precise approach would be to include only tags that have an  
effect on the build. But the effect of a tag on the build is to  
include or exclude a file from the compilation, and that file list is  
already in the build ID hash. So the build ID is already tag-sensitive  
in a perfectly precise way. So we do NOT explicitly add build tags to  
the build ID hash.

We do not include as part of the build ID the operating system,  
architecture, or whether the race detector is enabled, even though all  
three have an effect on the output, because that information is used  
to decide the install location. Binaries for linux and binaries for  
darwin are written to different directory trees; including that  
information in the build ID is unnecessary (although it would be  
harmless).

TODO(rsc): Investigate the cost of putting source file content into  
the build ID hash as a replacement for the use of mtimes. Using the  
file content would avoid all the mtime problems, but it does require  
reading all the source files, something we avoid today (we read the  
beginning to find the build tags and the imports, but we stop as soon  
as we see the import block is over). If the package is stale, the compiler  
is going to read the files anyway. But if the package is up-to-date, the  
read is overhead.

TODO(rsc): Investigate the complexity of making the build more  
precise about when individual results are needed. To be fully precise,  
there are two results of a compilation: the entire .a file used by the link  
and the subpiece used by later compilations (__.PKGDEF only).  
If a rebuild is needed but produces the previous __.PKGDEF, then  
no more recompilation due to the rebuilt package is needed, only  
relinking. To date, there is nothing in the Go command to express this.

## Special Cases

When the go command makes the wrong build decision and does not  
rebuild something it should, users fall back to adding the -a flag.  
Any common use of the -a flag should be considered prima facie evidence  
that isStale is returning an incorrect false result in some important case.  
Bugs reported in the behavior of -a itself should prompt the question  
``Why is -a being used at all? What bug does that indicate?''

There is a long history of changes to isStale to try to make -a into a  
suitable workaround for bugs in the mtime-based decisions.  
It is worth recording that history to inform (and, as much as possible, deter) future changes.

(1) Before the build IDs were introduced, building with alternate tags  
would happily reuse installed objects built without those tags.  
For example, "go build -tags netgo myprog.go" would use the installed  
copy of package net, even if that copy had been built without netgo.  
(The netgo tag controls whether package net uses cgo or pure Go for  
functionality such as name resolution.)  
Using the installed non-netgo package defeats the purpose.

Users worked around this with "go build -tags netgo -a myprog.go".

Build IDs have made that workaround unnecessary:
```sh
go build -tags netgo myprog.go
```
cannot use a non-netgo copy of package net.

(2) Before the build IDs were introduced, building with different toolchains,  
especially changing between toolchains, tried to reuse objects stored in  
$GOPATH/pkg, resulting in link-time errors about object file mismatches.

Users worked around this with "go install -a ./...".

Build IDs have made that workaround unnecessary:  
"go install ./..." will rebuild any objects it finds that were built against  
a different toolchain.

(3) The common use of "go install -a ./..." led to reports of problems  
when the -a forced the rebuild of the standard library, which for some  
users was not writable. Because we didn't understand that the real  
problem was the bug -a was working around, we changed -a not to  
apply to the standard library.

(4) The common use of "go build -tags netgo -a myprog.go" broke  
when we changed -a not to apply to the standard library, because  
if go build doesn't rebuild package net, it uses the non-netgo version.

Users worked around this with "go build -tags netgo -installsuffix barf myprog.go".  
The -installsuffix here is making the go command look for packages  
in pkg/$GOOS_$GOARCH_barf instead of pkg/$GOOS_$GOARCH.  
Since the former presumably doesn't exist, go build decides to rebuild  
everything, including the standard library. Since go build doesn't  
install anything it builds, nothing is ever written to pkg/$GOOS_$GOARCH_barf,  
so repeated invocations continue to work.

If the use of -a wasn't a red flag, the use of -installsuffix to point to  
a non-existent directory in a command that installs nothing should  
have been.

(5) Now that (1) and (2) no longer need -a, we have removed the kludge  
introduced in (3): once again, -a means ``rebuild everything,'' not  
``rebuild everything except the standard library.'' Only Go 1.4 had  
the restricted meaning.

In addition to these cases trying to trigger rebuilds, there are  
special cases trying NOT to trigger rebuilds. The main one is that for  
a variety of reasons (see above), the install process for a Go release  
cannot be relied upon to set the mtimes such that the go command will  
think the standard library is up to date. So the mtime evidence is  
ignored for the standard library if we find ourselves in a release  
version of Go. Build ID-based staleness checks still apply to the  
standard library, even in release versions. This makes  
'go build -tags netgo' work, among other things.
