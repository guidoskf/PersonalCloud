# PersonalCloud — Backup and Restore Runbook

## 1. Purpose

This runbook describes how to create, verify, and restore compressed backups of the PersonalCloud project directory.

The procedure preserves the complete project structure, including documentation, configuration files, hidden files, and empty directories.

It also defines integrity checks and a restoration test to verify that a backup can be extracted successfully.

## 2. Backup Scope

### Included

The backup covers the complete project directory:

`/home/gui/projects/PersonalCloud`

It includes:

- Project documentation and Markdown files.
- Docker Compose files and application configurations.
- Hidden files, including `.gitignore`.
- Empty directories.
- Currently unused application configuration files.
- Application logs stored inside the project directory.
- The `.git` directory and local Git history, once Git is initialized.

The procedure archives the complete project directory. It does not use `.gitignore` to determine which files are included.

### Not Included

This procedure does not back up:

- The WSL Ubuntu distribution or its operating system.
- Docker Desktop configuration or container images.
- Docker named volumes and persistent application data stored outside the project directory.
- Cloud infrastructure resources.
- Files stored outside the PersonalCloud project directory.

This is a project-level backup, not a complete system backup or a full disaster recovery solution.

## 3. Backup Destination

The selected destination is a separate physical disk from the one hosting the WSL project environment.

Windows path:

`D:\DevOps Projects\PersonalCloud\Backup`

WSL path:

`/mnt/d/DevOps Projects/PersonalCloud/Backup`

This separation provides protection against failure of the disk containing the project.

However, both disks remain inside the same computer. This backup does not protect against loss of the entire computer, theft, fire, or ransomware affecting both disks.

External and off-site backups will be addressed in a later stage.

## 4. Backup Naming Convention

Each backup consists of two files:

- `PersonalCloud-project-YYYYMMDD-HHMMSS.tar.gz`
- `PersonalCloud-project-YYYYMMDD-HHMMSS.tar.gz.sha256`

The timestamp uses the local system date and time.

The `.tar.gz` file contains the compressed project archive.

The `.sha256` file stores the SHA-256 checksum used to verify the archive's integrity.

Each backup must have a unique name. Existing backups must not be overwritten.

Both files must be kept together because the checksum file references the archive by its relative filename.

## 5. Prerequisites

Before creating a backup, verify that:

- The WSL Ubuntu environment is running.
- The project directory exists.
- The backup destination is mounted and accessible.
- The destination has sufficient free space.
- The required commands are available: `tar`, `gzip`, and `sha256sum`.

Docker Desktop does not need to be running because this procedure operates directly on project files.

Avoid editing project files while the archive is being created.

## 6. Creating a Backup

Run the following commands in the WSL Ubuntu terminal:

```bash
PROJECT_ROOT="/home/gui/projects/PersonalCloud"
PROJECT_PARENT="/home/gui/projects"
BACKUP_DIR="/mnt/d/DevOps Projects/PersonalCloud/Backup"

TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
BACKUP_NAME="PersonalCloud-project-${TIMESTAMP}.tar.gz"
BACKUP_FILE="${BACKUP_DIR}/${BACKUP_NAME}"
TEMP_FILE="${BACKUP_DIR}/.${BACKUP_NAME}.partial"

if [ ! -d "$PROJECT_ROOT" ]; then
    echo "ERROR: Project directory does not exist."
    exit 1
fi

if [ ! -d "$BACKUP_DIR" ] || [ ! -w "$BACKUP_DIR" ]; then
    echo "ERROR: Backup destination is missing or not writable."
    exit 1
fi

for COMMAND in tar gzip sha256sum; do
    if ! command -v "$COMMAND" >/dev/null 2>&1; then
        echo "ERROR: Required command not found: $COMMAND"
        exit 1
    fi
done

if [ -e "$BACKUP_FILE" ] || [ -e "$BACKUP_FILE.sha256" ] || [ -e "$TEMP_FILE" ]; then
    echo "ERROR: Backup filename already exists. Refusing to overwrite."
    exit 1
fi

echo "========== CREATING BACKUP =========="

if tar -czf "$TEMP_FILE" -C "$PROJECT_PARENT" PersonalCloud; then
    mv -- "$TEMP_FILE" "$BACKUP_FILE"
else
    echo "ERROR: Backup creation failed."
    rm -f -- "$TEMP_FILE"
    exit 1
fi

if ! (
    cd "$BACKUP_DIR" &&
    sha256sum "$BACKUP_NAME" > "$BACKUP_NAME.sha256" &&
    sha256sum -c "$BACKUP_NAME.sha256"
); then
    echo "ERROR: Checksum generation or verification failed."
    exit 1
fi

echo ""
echo "========== BACKUP FILES =========="
ls -lh "$BACKUP_FILE" "$BACKUP_FILE.sha256"
```

### Command explanation

- `PROJECT_ROOT` identifies the original project directory.
- `PROJECT_PARENT` identifies the directory containing the project.
- `BACKUP_DIR` defines the backup destination.
- `TIMESTAMP` generates a timestamp for the archive name.
- `tar -czf` creates a compressed TAR archive.
- `-C "$PROJECT_PARENT"` makes the archive contain `PersonalCloud/` as its top-level directory.
- `TEMP_FILE` is a temporary archive filename ending in `.partial`. It prevents an incomplete archive from appearing under the final backup name.
- `mv` renames the completed temporary archive to its final name.
- `sha256sum` calculates the archive checksum.
- `sha256sum -c` immediately verifies the generated checksum.

If archive creation fails, the incomplete temporary file is removed. Existing backups are never intentionally overwritten.

The archive must be stored outside the project directory to prevent it from being included in its own backup.

## 7. Backup Verification

Every backup must pass the following integrity checks before being considered valid.

First, identify the backup to verify:

```bash
BACKUP_DIR="/mnt/d/DevOps Projects/PersonalCloud/Backup"
BACKUP_NAME="PersonalCloud-project-YYYYMMDD-HHMMSS.tar.gz"
BACKUP_FILE="$BACKUP_DIR/$BACKUP_NAME"
```

Replace the example timestamp with the actual backup filename.

### 7.1 SHA-256 Verification

```bash
(
    cd "$BACKUP_DIR" &&
    sha256sum -c "$BACKUP_NAME.sha256"
)
```

Expected result:

`PersonalCloud-project-YYYYMMDD-HHMMSS.tar.gz: OK`

This confirms that the archive matches the checksum stored when the checksum file was generated.

SHA-256 detects content changes but does not provide encryption or protection against an attacker who can replace both the archive and its checksum.

### 7.2 GZIP Integrity Test

```bash
gzip -t "$BACKUP_FILE" && echo "GZIP INTEGRITY: OK"
```

This checks whether the compressed file can be read without detecting GZIP integrity errors.

### 7.3 Archive Content Inspection

```bash
tar -tzf "$BACKUP_FILE"
```

Review the output to confirm that the archive contains the expected project structure, documentation, configuration files, and directories.

To check archive readability without listing its contents:

```bash
tar -tzf "$BACKUP_FILE" > /dev/null
```

## 8. Restoration Test

A backup must be tested by extracting it to a temporary directory.

This procedure does not modify the original project.

### 8.1 Create a Temporary Restore Directory

```bash
RESTORE_TEST_DIR="$(mktemp -d /tmp/personalcloud-restore-test-XXXXXX)"

echo "Temporary restore directory:"
echo "$RESTORE_TEST_DIR"
```

The temporary directory is created with a unique name to avoid overwriting an existing directory.

### 8.2 Extract the Archive

```bash
if tar -xzf "$BACKUP_FILE" -C "$RESTORE_TEST_DIR"; then
    echo "EXTRACTION: OK"
else
    echo "EXTRACTION: FAILED"
    exit 1
fi
```

The restored project will be located at:

`$RESTORE_TEST_DIR/PersonalCloud`

### 8.3 Compare Extracted Files

```bash
if tar --compare --gzip --file="$BACKUP_FILE" --directory="$RESTORE_TEST_DIR"; then
    echo "ARCHIVE COMPARISON: OK"
else
    echo "ARCHIVE COMPARISON: FAILED"
    exit 1
fi
```

This compares the extracted files against the contents recorded in the archive.

Expected result:

`ARCHIVE COMPARISON: OK`

### 8.4 Inspect the Restored Directory

```bash
echo "========== RESTORED FILE COUNT =========="
find "$RESTORE_TEST_DIR/PersonalCloud" -type f | wc -l

echo ""
echo "========== RESTORED DIRECTORY SIZE =========="
du -sh "$RESTORE_TEST_DIR/PersonalCloud"
```

These commands report the number of restored files and the total directory size.

### 8.5 Remove the Temporary Restore

After completing and reviewing the verification, remove only the temporary directory created for this test:

```bash
rm -r -- "$RESTORE_TEST_DIR"
```

Before running the removal command, verify that `RESTORE_TEST_DIR` contains the exact temporary path printed during the test.

Never use a wildcard or an unverified path for this operation.

## 9. Actual Project Restoration

For an actual recovery:

1. Locate the required `.tar.gz` archive and its corresponding `.sha256` file.
2. Verify the SHA-256 checksum.
3. Test GZIP integrity.
4. Inspect the archive contents.
5. Choose an appropriate restore destination.
6. Ensure that the destination parent directory exists.
7. Ensure that a `PersonalCloud` directory does not already exist at the destination.
8. Extract the archive into the selected parent directory.

Example:

```bash
RESTORE_PARENT="/path/to/recovery/location"

if [ ! -d "$RESTORE_PARENT" ]; then
    echo "ERROR: Restore destination does not exist."
    exit 1
fi

if [ -e "$RESTORE_PARENT/PersonalCloud" ]; then
    echo "ERROR: A PersonalCloud directory already exists at the destination."
    exit 1
fi

tar -xzf "$BACKUP_FILE" -C "$RESTORE_PARENT"
```

The archive contains a top-level `PersonalCloud/` directory. Therefore, extraction creates:

`$RESTORE_PARENT/PersonalCloud`

Do not extract over an existing working project without first preserving it and planning the replacement.

After restoration, inspect the directory structure, file permissions, configuration files, and application requirements before attempting to start services.

## 10. Security and Limitations

- TAR and GZIP do not encrypt the archive.
- SHA-256 verifies integrity but does not provide confidentiality or authenticated origin.
- The checksum file is stored alongside the archive and is not an independent security control.
- The backup destination is a separate physical disk but remains inside the same computer.
- `.gitignore` does not control what TAR includes. Files ignored by Git are still included in this archive if they exist inside the project directory.
- Secrets must not be stored in unencrypted backup archives. An encrypted backup strategy and independent recovery-key protection must be designed before introducing sensitive production credentials into the backup scope.
- This procedure does not replace application-consistent database backups, Kubernetes backups, infrastructure-as-code, or full disaster recovery procedures.
- A project archive is a file-level backup. It does not guarantee application consistency for data that is actively changing during archive creation.

## 11. Initial Backup Validation Record

**Date:** 2026-11-25

**Archive:**

`PersonalCloud-project-20261125-161232.tar.gz`

**Destination:**

`D:\DevOps Projects\PersonalCloud\Backup`

**Initial source directory size:** 120 KB

**Compressed archive size:** approximately 6.8 KB

**Restored file count:** 19

The initial backup passed the following checks:

- SHA-256 checksum verification.
- GZIP integrity test.
- Archive content inspection.
- Successful extraction to a temporary directory.
- TAR comparison between the archive and extracted files.

**Result:** Initial project backup successfully validated.

This initial backup predates this runbook and does not contain it.

## 12. Future Improvements

The following items are outside the scope of the initial backup procedure and will be addressed as the project evolves:

- Backup scheduling and automation.
- Backup retention and rotation policies.
- Encrypted backups and secure recovery-key storage.
- External or off-site backup copies.
- Application data and database backup procedures.
- Backup monitoring and failure notifications.
- Periodic recovery drills.
- Integration with Host A and Host B disaster recovery procedures.