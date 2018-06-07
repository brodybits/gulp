<!-- front-matter
id: api-dest
title: dest()
hide_title: true
sidebar_label: dest()
-->

# dest(folder, [options])

Takes a folder path string or a function as the first argument and an options object as the second. If given a function, it will be called with each [vinyl] `File` object and must return a folder path.

## Usage

```js
src('./client/templates/*.pug')
  .pipe(pug())
  .pipe(dest('./build/templates'))
  .pipe(minify())
  .pipe(dest('./build/minified_templates'));
```

## Parameters

| parameter | type | note |
|:----------:|:-----:|------|
| folder <br> **(required)** | string <br> function | The path (output folder) to write files to. Or a function that returns it, the function will be provided a [vinyl File instance]. |
| options | object | Detailed in [Options](#options) below. |

## Return

Returns a stream that accepts [vinyl] `File` objects.

1. writes them to disk at the folder/cwd specified, and 3. passes them downstream so you can keep piping these around.

2. Once the file is written to disk, an attempt is made to determine if the `stat.mode`, `stat.mtime` and `stat.atime` of the [vinyl] `File` object differ from the file on the filesystem.
If they differ and the running process owns the file, the corresponding filesystem metadata is updated. If they don't differ or the process doesn't own the file, the attempt is skipped silently.

If the file has a `symlink` attribute specifying a target path, then a symlink will be created.

Note: The file will be modified after being written to this stream.
  - `cwd`, `base`, and `path` will be overwritten to match the folder.
  - `stat` will be updated to match the file on the filesystem.
  - `contents` will have it's position reset to the beginning if it is a stream.

__Caveats__
This functionality is disabled on Windows operating systems or any other OS that doesn't support `process.getuid` or `process.geteuid` in node. This is due to Windows having very unexpected results through usage of `fs.fchmod` and `fs.futimes`.

Note: The `fs.futimes()` method internally converts `stat.mtime` and `stat.atime` timestamps to seconds; this division by `1000` may cause some loss of precision in 32-bit Node.js.

## Options

- Values passed to the options must be of the expected type, otherwise they will be ignored.
- All options can be passed a function instead of a value. The function will be called with the [vinyl] `File` object as its only argument and must return a value of the expected type for that option.

| name | type | default | note |
|:----------------:|:-------------------:|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| cwd | string | process.cwd() | The working directory the folder is relative to. |
| mode | number | The `mode` of the input file (`file.stat.mode`) if any, or the process mode if the input file has no `mode` property. | The mode the files should be created with. This option is only resolved if the [vinyl] `File` is not symbolic. |
| dirMode | number | The process `mode`. | The mode directories should be created with. |
| overwrite | boolean | true | Whether or not existing files with the same path should be overwritten. Always overwrite existing files by default. |
| append | boolean | false | Whether or not new data should be appended after existing file contents (if any). Always replace existing contents by default. |
| sourcemaps | boolean <br> string |  | Enables sourcemap support on files passed through the stream.  Will write inline soucemaps if specified as `true`. Specifying a `String` path will write external sourcemaps at the given path. [Link to sourcemaps section at the bottom] |
| relativeSymlinks | boolean | false | When creating a symlink, whether or not the created symlink should be relative. If `false`, the symlink will be absolute. Note: This option will be ignored if a `junction` is being created, as they must be absolute. |
| useJunctions | boolean | true | When creating a symlink, whether or not a directory symlink should be created as a `junction`. This option is only relevant on Windows and ignored elsewhere. Please refer to the [Symbolic Links on Windows][symbolic-caveats] section below. |

## Sourcemaps section
Examples:
```js
// Write as inline comments
vfs.dest('./', { sourcemaps: true });

// Write as files in the same folder
vfs.dest('./', { sourcemaps: '.' });
```

## Symbolic Links on Windows

When creating symbolic links on Windows, we pass a `type` argument to Node's
`fs` module which specifies the kind of target we link to (one of `'file'`,
`'dir'` or `'junction'`). Specifically, this will be `'file'` when the target
is a regular file, `'junction'` if the target is a directory, or `'dir'` if
the target is a directory and the user overrides the `useJunctions` option
default.

However, if the user tries to make a "dangling" link (pointing to a non-existent
target) we won't be able to determine automatically which type we should use.
In these cases, `vinyl-fs` will behave slightly differently depending on
whether the dangling link is being created via `symlink()` or via `dest()`.

For dangling links created via `symlink()`, the incoming vinyl represents the
target and so we will look to its stats to guess the desired type. In
particular, if `isDirectory()` returns false then we'll create a `'file'` type
link, otherwise we will create a `'junction'` or a `'dir'` type link depending
on the value of the `useJunctions` option.

For dangling links created via `dest()`, the incoming vinyl represents the link -
typically read off disk via `src()` with the `resolveSymlinks` option set to
false. In this case, we won't be able to make any reasonable guess as to the
type of link and we default to using `'file'`, which may cause unexpected behavior
if you are creating a "dangling" link to a directory. It is advised to avoid this
scenario.
