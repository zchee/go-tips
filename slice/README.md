# slice

## Check whether the contain any string in slice

This is mostly common sense, but occasionally forget.

For example, the situation is there is the `[]string` that C language compile command.  
If you want to the check whether the contains `-std=` compiler flag into this slice:

### `strings.Contains`

```go
args := []string{"clang", "-o", "foo", "-O2", "-march=native" "foo.c"}

if strings.Contains("-std=", strings.Join(args, " ")) {
	args = append(args, "-std=c11")
}
```

### `strings.Index`

```go
args := []string{"clang", "-o", "foo", "-O2", "-march=native" "foo.c"}

if strings.Index(strings.Join(args, " "), "-std=") >= 0 {
	args = append(args, "-std=c11")	
}
```

The `strings.Contains` is actually wrapper of `strings.Index(s, substr) >= 0`.
