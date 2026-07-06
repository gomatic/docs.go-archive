---
title: go-archive
---
**Stream-oriented `tar.gz` archiving for Go: `Create` writes a gzip-compressed tar of filesystem paths to an `io.Writer`, `Extract` reads one from an `io.Reader` into a directory (guarding against zip-slip path traversal), and `List` reports an archive's entry names without writing anything.**

- **Source:** [gomatic/go-archive](https://github.com/gomatic/go-archive)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-archive](https://pkg.go.dev/github.com/gomatic/go-archive)

## Install

```sh
go get github.com/gomatic/go-archive
```

```go
import archive "github.com/gomatic/go-archive"
```

## Usage

The package exposes three functions over streams. `Create` takes a `SourcePaths` (a `[]string` of filesystem roots to walk) and writes the archive to any `io.Writer`. `Extract` reads from any `io.Reader` into a `DestDir` and returns the extracted entry names. `List` reads from any `io.Reader` and returns entry names only.

### Create

```go
out, err := os.Create("backup.tar.gz")
if err != nil {
	return err
}
defer out.Close()

if err := archive.Create(out, archive.SourcePaths{"src", "docs"}); err != nil {
	return err
}
```

`Create` walks each path with `filepath.Walk`, writing every entry beneath it (directories, regular files, and symlinks) into a single gzip-compressed tar stream, then flushes and closes the tar and gzip writers.

### Extract

```go
in, err := os.Open("backup.tar.gz")
if err != nil {
	return err
}
defer in.Close()

names, err := archive.Extract(in, archive.DestDir("./restore"))
if err != nil {
	return err
}
// names holds each extracted entry's archived path.
```

`Extract` recreates directories, regular files (with their recorded mode), and symlinks under `dest`. Every entry target and every symlink's resolved target is checked to stay within `dest` before any write; an entry that escapes is rejected with `ErrPathTraversal`.

### List

```go
in, err := os.Open("backup.tar.gz")
if err != nil {
	return err
}
defer in.Close()

names, err := archive.List(in)
if err != nil {
	return err
}
for _, name := range names {
	fmt.Println(name)
}
```

`List` iterates the archive and returns entry names without writing anything to disk.

## Errors

Every failure wraps a sentinel of type `errs.Const` from [gomatic/go-error](https://github.com/gomatic/go-error), matchable with `errors.Is`:

- `ErrCreateArchive` — wrapped when an archive cannot be created.
- `ErrExtract` — wrapped when an archive cannot be read, listed, or extracted (an I/O or decompression fault).
- `ErrPathTraversal` — a malicious entry whose path or symlink target escapes the destination directory. It is distinct from `ErrExtract` so callers can tell a hostile archive from an I/O fault.

```go
if _, err := archive.Extract(in, archive.DestDir("./restore")); err != nil {
	if errors.Is(err, archive.ErrPathTraversal) {
		// reject the archive as hostile
	}
	return err
}
```

## Design

The package is dependency-free beyond the standard library and [gomatic/go-error](https://github.com/gomatic/go-error). The `tar`/`gzip` writer chain and reader are owned by unexported `creator` and `extractor` types so the stateful stdlib pointers live in struct fields rather than being threaded through every helper; `tar.Header` values pass by value and entry bodies as `io.Reader`. Path and symlink parameters use named domain types (`SourcePaths`, `DestDir`, and internal `filePath`/`linkName`) rather than bare strings.
