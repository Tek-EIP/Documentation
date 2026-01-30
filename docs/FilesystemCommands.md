# FileSystem Commands and File Descriptors


## Navigation Commands

### `ls` - List directory contents

**Syntax:**
```bash
ls [directory]
```

**Description:**
Lists the contents of a directory. Displays filenames and subdirectories.

**Internal Flow:**
1. Finds the directory inode via `find_directory_inode()`
2. Reads directory content with `read_wrapper()`
3. Iterates through `dirent2_t` entries (directory entries)
4. Displays each entry with its type (file/directory)

**Example:**
```bash
ls /home
# Output: file1.txt  dir1/  dir2/
```

---

### `cd` - Change directory

**Syntax:**
```bash
cd [directory]
```

**Description:**
Changes the current working directory. Updates the `PWD` environment variable.

**Internal Flow:**
1. Normalizes the path with `normalize_path()`
2. Checks if directory exists with `find_directory_inode()`
3. Verifies it's actually a directory with `is_directory_accessible()`
4. Updates the `PWD` environment variable

**Examples:**
```bash
cd /home/user
cd ..          # Go up one level
cd ../dir2     # Relative path
```

---

## File Manipulation Commands

### `mv` - Move/rename file

**Syntax:**
```bash
mv <source> <destination>
```

**Description:**
Moves or renames a file/directory.

**Internal Flow:**
1. Checks if source exists
2. Checks if destination doesn't exist (or is a directory)
3. Removes entry from source directory
4. Creates new entry in destination directory
5. Updates link counters (`i_links_count`)

**Examples:**
```bash
mv /home/old.txt /home/new.txt          # Rename
mv /home/file.txt /backup/              # Move
```

---

## Information Commands

### `stat` - Display file status

**Syntax:**
```bash
stat <file>
```

**Description:**
Displays detailed file information: size, permissions, inode, timestamps, etc.

**Internal Flow:**
1. Finds file inode
2. Reads inode metadata
3. Extracts and formats information:
   - Permissions (mode)
   - File size
   - Number of blocks used
   - UID/GID (owner/group)
   - Timestamps (access, modify, change, birth)
   - Device numbers

**Example:**
```bash
stat /home/file.txt
```

**Output:**
```
File: file.txt
Size: 1024    Blocks: 8    IO Block: 4096    regular file
Device: 8,1    Inode: 12345    Links: 1
Access: (0644/-rw-r--r--)    Uid: ( 1000/    user)    Gid: ( 1000/    user)
Access: 2025-01-30 15:30:45
Modify: 2025-01-30 15:30:45
Change: 2025-01-30 15:30:45
Birth: 2025-01-30 15:30:45
```

---

## File Descriptors

### Concept

**File Descriptors** are numeric identifiers representing open files. They allow you to:
- Read and write to files
- Handle multiple files simultaneously
- Maintain read/write position

### Internal Structure

```c
typedef struct {
    bool is_used;
    uint32_t inode;
    char path[256];
    int mode;
    size_t position;
    bool is_directory;
} file_descriptor_t;

file_descriptor_t fd_table[MAX_OPEN_FILES];
```

### Reserved FDs

- **0** : `stdin` (standard input)
- **1** : `stdout` (standard output)
- **2** : `stderr` (standard error)

---

### `open` - Open file descriptor

**Syntax:**
```bash
open <file> [flags]
```

**Available Flags:**
- `-r, --readonly` : Read-only (default)
- `-w, --writeonly` : Write-only
- `-rw, --readwrite` : Read + Write
- `-c, --create` : Create if doesn't exist
- `-t, --trunc` : Truncate file
- `-a, --append` : Write at end

**Description:**
Opens a file and returns a file descriptor

**Internal Flow:**
1. Normalizes path
2. Finds file inode
3. Checks flags (creation, existence, etc.)
4. Allocates a slot in `fd_table`
5. Initializes structure:
   - `inode` : inode number
   - `mode` : open flags
   - `position` : 0 (start of file)
   - `path` : normalized path

**Examples:**
```bash
open /home/file.txt -r
open /log.txt -w -a
open /new.txt -w -c
open /data.txt -rw
```

**Output:**
```
File opened: /home/file.txt (fd=3)
```

---

### `close` - Close file descriptor

**Syntax:**
```bash
close <fd>
```

**Description:**
Closes a file descriptor and frees its slot in the table.

**Internal Flow:**
1. Validates FD
2. Resets entry in `fd_table`:
   - `is_used = false`
   - `inode = 0`
   - `position = 0`
   - Clears path

**Example:**
```bash
close 3
# Output: File descriptor 3 closed
```

---

### `lsof` - List open files

**Syntax:**
```bash
lsof
```

**Description:**
Displays all currently open files with their file descriptors.

**Output:**
```
FD  MODE    INODE   PATH
--- ------- ------- ----
  3 r       12345   /home/file.txt
  4 w+a     12346   /log.txt
  5 r/w     12347   /data.bin
```

**Columns:**
- **FD** : File descriptor number
- **MODE** : Open mode
  - `r` : read only
  - `w` : write only
  - `r/w` : read/write
  - `+a` : append
  - `+c` : create
  - `+t` : truncate
- **INODE** : Inode number
- **PATH** : File path
