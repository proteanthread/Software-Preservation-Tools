# Software Preservation and Archival Tools


This collection of command-line utilities represents a dedicated and evolving toolkit for the preservation of digital software, ensuring its accessibility and usability for future reference and local archival purposes. What began as a set of simple tools is maturing into a more comprehensive, multi-stage workflow, although it remains a continual work in progress. Each script serves a distinct role in the archival process, from initial acquisition to final organization, providing a robust foundation for building clean, consistent, and durable software collections.

The archival process begins with acquisition, handled by the ia-download script. This powerful tool provides a direct interface to the vast repository of the Internet Archive, giving the archivist precise control over the retrieval process. Users can construct detailed search queries, filter by specific media types and collections, and select exact file extensions for download. By automating the search and download of specific items, it serves as the primary gateway for bringing historical software into the local archival environment.

Once files are acquired, they enter a crucial normalization and sanitization phase. This toolkit offers several specialized scripts for this purpose. For creating clean, modern, and web-friendly filenames, the sanitize and normalize-long scripts convert names to lowercase, replace ambiguous special characters with dashes, and handle complex character encodings. They also provide options for truncating filenames to a maximum length, ensuring compatibility across various filesystems. For projects requiring strict historical accuracy, the normalize script enforces the rigid MS-DOS 8.3 filename convention, converting names to uppercase and removing any characters that would be invalid on legacy systems. This distinction is vital for preserving software that depends on this older file structure.

With standardized names, the next step is extraction, managed by the intelligent ia-extract script. Software is often stored in compressed formats like ZIP, 7Z, or ISO files. This utility is designed to unpack them intelligently; it analyzes an archive's contents to prevent the common issue of creating redundant, nested directories. It automatically sanitizes the names of the extracted files and can even find and recursively extract archives buried within other archives. For safety, a --dry-run mode allows the user to preview all actions without making any changes to the filesystem.

The toolkit includes several utilities for ongoing maintenance and analysis, reflecting its "work in progress" nature. The cleanup script was developed as a one-time fix for a bug in a previous version, demonstrating the iterative improvement of the tools. The rematching script acts as a powerful deduplication aid, allowing a user to compare two directories and safely delete files in the current directory that have matching names in another, which is perfect for managing updated file sets. To assist in planning normalization strategies, the length script provides a statistical analysis of filename lengths within a directory, calculating the minimum, maximum, average, and median lengths to give the archivist a clear overview of the collection's characteristics.

Together, these scripts form a cohesive, command-line-driven system that addresses the full lifecycle of digital software preservation—from acquisition and sanitization to extraction and long-term maintenance:

Below is the cleaned-up version formatted for GitHub Markdown. You can copy and paste it directly into your editor.

---

## `cleanup.sh`

The `cleanup.sh` script is a one-time utility designed to correct a specific error introduced by a previous file-normalization script. It finds files incorrectly renamed with a numeric suffix after the extension (e.g. `MY_FILE.TXT.2`, `DOCUMENT.PDF.3`) and reverts them to their intended name (e.g. `MY_FILE.TXT`, `DOCUMENT.PDF`).

**Safety Check**
Before renaming `FILENAME.EXT.2` back to `FILENAME.EXT`, it verifies that `FILENAME.EXT` does not already exist in the same directory. If it does, the script skips that file, prints a conflict message, and moves on—preventing any accidental overwrites.

**How It Works**

* Uses `find` plus a `while` loop.
* Matches only files ending in `.[0-9]+` via regex `\.[0-9]+$`.
* Performs `mv -v` for verbose, per-file rename logging.

---

## `ia-download`

The `ia-download` script is a flexible CLI tool for searching and downloading from the Internet Archive. It requires three dependencies:

* [`jq`](https://stedolan.github.io/jq/)
* `curl`
* the `internetarchive` CLI

**Features**

1. **Dependency Check**
   Verifies `jq`, `curl`, and `internetarchive` are installed; otherwise, prints installation instructions.
2. **Interactive Search**

   * **Full Search Query** with tips for advanced syntax (e.g. `title:"..."`, `OR`, `AND -`).
   * **Collection Selection** (software, texts, movies, or all).
   * **File-Type Filtering** (e.g. `zip iso pdf`) or none for all types.
3. **Batch Workflow**

   * Builds and URL-encodes the final query.
   * Pages through the IA API to collect all matching item identifiers.
   * For each item:

     1. Fetches metadata.
     2. Filters file list by extension.
     3. Downloads each file with `curl` (displaying a progress bar).
4. **Logging & Summary**

   * Errors (metadata fetch or download failures) are timestamp-logged to `error.log`.
   * At completion, prints a summary of query parameters, collections searched, and counts of successes, failures, and total files downloaded.

---

## `ia-extract.sh`

An intelligent archive extractor that handles a variety of formats (`.zip`, `.7z`, `.rar`, `.tar`, `.iso`, etc.) without creating unnecessary subdirectories.

**Dependency Check**

* Ensures the `7z` command is available, otherwise provides distro-specific install instructions for Debian/Ubuntu and Fedora.

**Extraction Strategies**

1. **Single-File Archives**

   * Extracts directly into the current directory.
   * Sanitizes spaces to underscores.
   * Checks for existing files to avoid overwrites.
2. **Single-Folder Archives**

   * If the archive’s root is one folder, extracts its contents into `./`.
3. **Multi-File/Folder Archives**

   * Creates a new subdirectory named after the archive and extracts all contents there.

**Options & Flags**

* `--dry-run`

  * Simulates actions (files extracted, dirs created, renames) without touching the filesystem.
* **Recursive Extraction**

  * After extraction, scans for nested archives and extracts them automatically.
* **Optional Deletion**

  * Prompts whether to delete source archives after successful extraction.
  * Deletion commands are commented out by default for safety.

---

## `length`

A filename-length analysis tool providing statistics on the basenames (excluding extensions) of all files in the current directory (non-recursive).

**Metrics Reported**

* Total number of files
* Minimum filename length
* Maximum filename length
* Average filename length (two decimal places)
* Median filename length

**How It Works**

1. Uses `find . -maxdepth 1 -type f` to list files.
2. Strips extensions with `basename` + `${name%.*}`.
3. Computes each basename’s length.
4. Sorts lengths (`sort -n`), then finds min, max, average, and median via `head`, `tail`, and `awk`.
5. Exits gracefully with a “No valid filenames found” message if there are no files or all basenames are empty.

---

## `normalize.sh`

Enforces the MS-DOS 8.3 filename convention on files in the current directory (non-recursive). Skips subdirectories, hidden dotfiles, and itself.

**Rules Applied**

1. **Uppercase Conversion** (names and extensions).
2. **Character Sanitization**

   * Keeps only A–Z, 0–9, and these symbols: `_ $ ~ ! # % & - { } ( ) @ \ '`
   * Removes all others.
3. **Length Truncation**

   * Basename: max 8 characters
   * Extension: max 3 characters
4. **Reserved Names**

   * If sanitized name is a reserved device (`CON`, `PRN`, etc.), prepends an underscore.
5. **Conflict Resolution**

   * If the target name exists (and isn’t the same file), appends a numeric suffix (`.2`, `.3`, etc.).

**Warning**

* Permanently renames files. Strongly advise backing up data or testing in a copy.

---

## `normalize-long`

Cleans and standardizes filenames for modern OS/web use (non-recursive). Skips itself and hidden dotfiles.

**Modes**

* **Sanitation-Only (default)**

  * Decodes URL entities (`%20` → space).
  * Transliterates Unicode to ASCII.
  * Converts to lowercase.
  * Replaces symbols/spaces with `-`.
  * Collapses repeated `-` or `_`.
* **Sanitation + Truncation**

  * `-m <number>` or `--max <number>` to limit basename length after cleaning.

**Features**

* **Conflict Resolution**

  * If two files map to the same name, the latter gets a `.2`, `.3`, etc.
* **No-Extension Files**

  * Assigns a default `.dat` extension if none exists.
* **Empty-Name Handling**

  * Skips filenames that become empty after sanitization, with a warning.

**Warning**

* Permanently renames files. Advise backup before use.

---

## `rematching.sh`

Deletes files in the current directory if an identical-named file exists in a specified “master” directory. Use with extreme caution—it permanently deletes.

**Safety Features**

1. **Absolute Path Lock**

   * Captures `$PWD` in `CURRENT_DIR` and prefixes all `rm` operations with it.
2. **Strict Argument Check**

   * Requires exactly one argument (the comparison directory).
   * Exits with usage message if the argument count isn’t one.
3. **Directory Validation**

   * Verifies the argument is a directory with `-d`; exits on failure.

**Core Logic**

* Loops through files in the comparison directory.
* For each file, checks if a file of the same name exists in `CURRENT_DIR`.
* If yes, deletes it and prints a confirmation message.

---

## `sanitize.sh`

A simple filename cleaner for the current directory (non-recursive). Ignores itself and hidden dotfiles.

**Sanitization Pipeline**

1. Decodes URL entities (`%20` → space).
2. Transliterates Unicode to ASCII.
3. Converts to lowercase.
4. Replaces anything other than `[a-z0-9_]` with `-`.
5. Collapses multiple `-` or `_` into one.
6. Trims leading/trailing `-` or `_`.

**Conflict Resolution**

* If two files would collide, the second gets a suffix (e.g. `.2`, `.3`).

**Options**

* `-l <number>` to truncate basename length.
* `-h` for a help message.

**Warning**

* Permanent renames; back up data beforehand.

---

## `title-case`

An advanced filename formatter that applies mixed-case Title Case intelligently, handling abbreviations, catalog numbers, and compound words.

**User-Configurable Lists** (at the top of the script)

* `PASSTHROUGH_FILENAMES` (exact names to ignore)
* `ABBREVIATIONS` (forced ALL CAPS)
* `IDENTIFIER_PREFIXES` (e.g. catalog prefixes)
* `KNOWN_WORDS` (common words to keep lowercase)

**Processing Steps**

1. Skip passthrough names.
2. Preserve overrides (e.g. filenames ending in `-master`).
3. Replace separators (`_`, `.`) with spaces.
4. Tokenize into words; count “true words” (no digits/abbrev) to guide capitalization rules.
5. Apply:

   * Prefix-number joins (`bbd 08` → `bbd08`)
   * Abbreviation caps (`CD-ROM` → `CD-ROM`)
   * Identifier lowercasing (`abc123`)
   * Known words stay lowercase
6. True words:

   * If only one, ALL CAPS; otherwise, Title Case.
7. Reassemble, handle conflicts by appending `.2`, `.3`, etc., and rename.

**Note**

* Due to filename complexity, manual renaming may still be needed for perfect results.


More to come.
