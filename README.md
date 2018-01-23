Go Tips
=======

multiple `-extldflags` arguments
--------------------------------

### Q:

`-extldflags` is must be including `-ldflags` build flag.  
But if execute the `go build ...` command use some script(such as `Makefile`), sometimes failed go's `link` linker with below error.

```sh
flag provided but not defined: -fooflag'
```

build command is here.

```sh
go bulid -ldflags="-w -s -extldflags='-static -fooflag' example.com/john/doe"
```

However, will successful with `-extldflags=-static`.

### A:

In this case, go `link` linker will recognizes `link -fooflag -extldflags='-static`.  
To be cleared, `link` passes flags to `extld` command(such as `ld`) `-static` only.

As in the [golang/go/issues/6234](https://github.com/golang/go/issues/6234), need

```sh
#  -d	disable dynamic executable
#  -s	disable symbol table
#  -w	disable DWARF generation

go build -ldflags="-d -s -w '-extldflags=-static'" example.com/john/doe
```

Actually, We must be bracketed to `-extldflags=...` by a comma(`'`).  
After that, will passes all flags to C linker. Try and check with add `-v` to `-extldflags`.

Available`#cgo` magic comment
-----------------------------

Go version: `devel +334cbe3 Sun Oct 9 22:50:12 2016 +0000`  
defined in `$GOROOT/src/go/build/build.go.saveCgo()` switch cases, available

```sh
CFLAGS
CPPFLAGS
CXXFLAGS
FFLAGS
LDFLAGS
pkg-config
```
