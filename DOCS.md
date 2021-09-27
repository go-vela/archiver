# Documentation

This document intends to provide information on how to use the `archiver` library.

## Prerequisites

This section covers the dependencies required to get develop locally with `archiver`.

* [Golang](https://golang.org/dl/) - for source code and [dependency management](https://github.com/golang/go/wiki/Modules)
* [Make](https://www.gnu.org/software/make/) - start up local development

Introducing **Archiver 3.1** - a cross-platform, multi-format archive utility and Go library. A powerful and flexible library meets an elegant CLI in this generic replacement for several platform-specific or format-specific archive utilities.

## Features

Package `archiver` makes it trivially easy to make and extract common archive formats such as tarball (and its compressed variants) and zip. Simply name the input and output file(s). It is powered by the Go standard library and several third-party, pure-Go libraries.

Files are put into the root of the archive; directories are recursively added, preserving structure.

- Make whole archives from a list of files
- Open whole archives to a folder
- Extract specific files/folders from archives
- Stream files in and out of archives without needing actual files on disk
- Traverse archive contents without loading them
- Compress files
- Decompress files
- Streaming compression and decompression
- Several archive and compression formats supported

### Format-dependent features

- Gzip is multithreaded
- Optionally create a top-level folder to avoid littering a directory or archive root with files
- Toggle overwrite existing files
- Adjust compression level
- Zip: store (not compress) already-compressed files
- Make all necessary directories
- Open password-protected RAR archives
- Optionally continue with other files after an error

### Supported compression formats

- brotli (br)
- bzip2 (bz2)
- flate (zip)
- gzip (gz)
- lz4
- snappy (sz)
- xz
- zstandard (zstd)

### Supported archive formats

- .zip
- .tar (including any compressed variants like .tar.gz)
- .rar (read-only)

Tar files can optionally be compressed using any of the above compression formats.

## Library Use

The archiver package allows you to easily create and open archives, walk their contents, extract specific files, compress and decompress files, and even stream archives in and out using pure io.Reader and io.Writer interfaces, without ever needing to touch the disk.

To use as a dependency in your project:

```bash
go get github.com/go-vela/archiver/v3
```

```go
import "github.com/go-vela/archiver/v3"
```

[See the package's GoDoc](https://pkg.go.dev/github.com/mholt/archiver?tab=doc) for full API documentation.

For example, creating or unpacking an archive file:

```go
err := archiver.Archive([]string{"testdata", "other/file.txt"}, "test.zip")
// ...
err = archiver.Unarchive("test.tar.gz", "test")
```

The archive format is determined by file extension. (There are [several functions in this package](https://pkg.go.dev/github.com/mholt/archiver?tab=doc) which perform a task by inferring the format from file extension or file header, including `Archive()`, `Unarchive()`, `CompressFile()`, and `DecompressFile()`.)

To configure the archiver used or perform, create an instance of the format's type:

```go
z := archiver.Zip{
	CompressionLevel:       flate.DefaultCompression,
	MkdirAll:               true,
	SelectiveCompression:   true,
	ContinueOnError:        false,
	OverwriteExisting:      false,
	ImplicitTopLevelFolder: false,
}

err := z.Archive([]string{"testdata", "other/file.txt"}, "/Users/matt/Desktop/test.zip")
```

Inspecting an archive:

```go
err = z.Walk("/Users/matt/Desktop/test.zip", func(f archiver.File) error {
	zfh, ok := f.Header.(zip.FileHeader)
	if ok {
		fmt.Println("Filename:", zfh.Name)
	}
	return nil
})
```

Streaming files into an archive that is being written to the HTTP response:

```go
err = z.Create(responseWriter)
if err != nil {
	return err
}
defer z.Close()

for _, fname := range filenames {
	info, err := os.Stat(fname)
	if err != nil {
		return err
	}

	// get file's name for the inside of the archive
	internalName, err := archiver.NameInArchive(info, fname, fname)
	if err != nil {
		return err
	}

	// open the file
	file, err := os.Open(f)
	if err != nil {
		return err
	}

	// write it to the archive
	err = z.Write(archiver.File{
		FileInfo: archiver.FileInfo{
			FileInfo:   info,
			CustomName: internalName,
		},
		ReadCloser: file,
	})
	file.Close()
	if err != nil {
		return err
	}
}
```

The `archiver.File` type allows you to use actual files with archives, or to mimic files when you only have streams.

There's a lot more that can be done, too. [See the GoDoc](https://pkg.go.dev/github.com/mholt/archiver?tab=doc) for full API documentation.

**Security note: This package does NOT attempt to mitigate zip-slip attacks.** It is [extremely difficult](https://github.com/rubyzip/rubyzip/pull/376) [to do properly](https://github.com/mholt/archiver/pull/65#issuecomment-395988244) and [seemingly impossible to mitigate effectively across platforms](https://github.com/golang/go/issues/20126). [Attempted fixes have broken processing of legitimate files in production](https://github.com/mholt/archiver/pull/70#issuecomment-423267320), rendering the program unusable. Our recommendation instead is to inspect the contents of an untrusted archive before extracting it (this package provides `Walkers`) and decide if you want to proceed with extraction.