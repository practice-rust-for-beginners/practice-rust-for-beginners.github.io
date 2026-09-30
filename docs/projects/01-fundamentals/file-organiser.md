# Project: File Organiser

**Track:** 🦀 Fundamentals &nbsp;|&nbsp; **Difficulty:** 🟢 Beginner &nbsp;|&nbsp; **Crates:** standard library only

---

## What You'll Build

A CLI tool that scans a target directory and moves files into subfolders organised by file extension. Run it on your cluttered Downloads folder and watch it sort itself out.

```text
$ cargo run -- ~/Downloads --dry-run
[DRY RUN] Would move: resume.pdf        → ~/Downloads/pdf/resume.pdf
[DRY RUN] Would move: photo.jpg         → ~/Downloads/images/photo.jpg
[DRY RUN] Would move: notes.txt         → ~/Downloads/text/notes.txt
[DRY RUN] Would move: archive.zip       → ~/Downloads/archives/archive.zip
[DRY RUN] Would move: script.rs         → ~/Downloads/code/script.rs
4 files would be moved.

$ cargo run -- ~/Downloads
Moved: resume.pdf        → pdf/
Moved: photo.jpg         → images/
Moved: notes.txt         → text/
Moved: archive.zip       → archives/
Moved: script.rs         → code/
Done. 5 files organised.
```

---

## Tech Stack

| Component | Choice |
|-----------|--------|
| CLI arguments | `std::env::args` |
| Filesystem | `std::fs`, `std::path::Path` |
| Extension mapping | `HashMap<&str, &str>` |
| Error handling | `Result` + custom error |

---

## Step 1 — Project Setup

```bash
cargo new file-organiser
cd file-organiser
```

---

## Step 2 — Extension to Folder Mapping

```rust
// src/categories.rs
use std::collections::HashMap;

pub fn extension_map() -> HashMap<&'static str, &'static str> {
    let mut map = HashMap::new();
    // Documents
    for ext in ["pdf", "doc", "docx", "odt", "rtf"] { map.insert(ext, "documents"); }
    // Images
    for ext in ["jpg", "jpeg", "png", "gif", "bmp", "webp", "svg", "heic"] { map.insert(ext, "images"); }
    // Video
    for ext in ["mp4", "mov", "avi", "mkv", "webm"] { map.insert(ext, "video"); }
    // Audio
    for ext in ["mp3", "wav", "flac", "aac", "ogg"] { map.insert(ext, "audio"); }
    // Archives
    for ext in ["zip", "tar", "gz", "bz2", "xz", "7z", "rar"] { map.insert(ext, "archives"); }
    // Code
    for ext in ["rs", "py", "js", "ts", "go", "c", "cpp", "h", "java", "rb", "sh"] { map.insert(ext, "code"); }
    // Text
    for ext in ["txt", "md", "csv", "log", "json", "toml", "yaml", "yml", "xml"] { map.insert(ext, "text"); }
    // Spreadsheets
    for ext in ["xls", "xlsx", "ods"] { map.insert(ext, "spreadsheets"); }
    map
}

pub fn folder_for(ext: &str, map: &HashMap<&str, &str>) -> &'static str {
    map.get(ext.to_lowercase().as_str()).copied().unwrap_or("misc")
}
```

---

## Step 3 — Argument Parsing

```rust
// src/args.rs
use std::path::PathBuf;

pub struct Config {
    pub target_dir: PathBuf,
    pub dry_run:    bool,
    pub verbose:    bool,
}

impl Config {
    pub fn from_args() -> Result<Self, String> {
        let mut args = std::env::args().skip(1);
        let target = args.next().ok_or("Usage: file-organiser <directory> [--dry-run] [--verbose]")?;
        let flags: Vec<String> = args.collect();
        Ok(Config {
            target_dir: PathBuf::from(target),
            dry_run:    flags.contains(&"--dry-run".to_string()),
            verbose:    flags.contains(&"--verbose".to_string()),
        })
    }
}
```

---

## Step 4 — The Organiser

```rust
// src/organiser.rs
use std::fs;
use std::path::Path;
use crate::categories::{extension_map, folder_for};
use crate::args::Config;

pub fn run(config: &Config) -> Result<(), Box<dyn std::error::Error>> {
    let dir = &config.target_dir;
    if !dir.is_dir() {
        return Err(format!("'{}' is not a directory", dir.display()).into());
    }

    let map = extension_map();
    let mut count = 0;

    if config.dry_run { println!("[DRY RUN] No files will be moved.\n"); }

    for entry in fs::read_dir(dir)? {
        let entry = entry?;
        let path  = entry.path();

        // Skip directories and hidden files
        if path.is_dir() { continue; }
        if path.file_name()
            .and_then(|n| n.to_str())
            .map(|n| n.starts_with('.'))
            .unwrap_or(false) { continue; }

        let ext = path.extension()
            .and_then(|e| e.to_str())
            .unwrap_or("");

        let folder_name = folder_for(ext, &map);
        let dest_dir    = dir.join(folder_name);
        let dest_path   = dest_dir.join(path.file_name().unwrap());

        if config.dry_run || config.verbose {
            println!("{} {:30} → {}/",
                if config.dry_run { "[DRY RUN] Would move:" } else { "Moved:" },
                path.file_name().unwrap().to_string_lossy(),
                folder_name,
            );
        }

        if !config.dry_run {
            fs::create_dir_all(&dest_dir)?;
            fs::rename(&path, &dest_path)?;
        }

        count += 1;
    }

    if config.dry_run {
        println!("\n{count} files would be moved.");
    } else {
        println!("\nDone. {count} files organised.");
    }
    Ok(())
}
```

---

## Step 5 — Main

```rust
// src/main.rs
mod categories;
mod args;
mod organiser;

fn main() {
    let config = match args::Config::from_args() {
        Ok(c)  => c,
        Err(e) => { eprintln!("{e}"); std::process::exit(1); }
    };

    if let Err(e) = organiser::run(&config) {
        eprintln!("Error: {e}");
        std::process::exit(1);
    }
}
```

---

## Step 6 — Run It

```bash
# Dry run first — always safe
cargo run -- ~/Downloads --dry-run

# Actually move files
cargo run -- ~/Downloads
```

---

## Stretch Goals

- [ ] Add `--undo` flag that reads a log file and moves everything back
- [ ] Write a log file (`organiser.log`) listing every move with timestamps
- [ ] Add `--config my_rules.toml` to let users define custom extension → folder rules
- [ ] Add unit tests for `extension_map` and `folder_for`

---

[Back to All Projects](../index.md){ .md-button }


[Next Track: Web Dev Projects](../02-web-development/blog-engine.md){ .md-button .md-button--primary }
