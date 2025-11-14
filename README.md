## Directory Backup & Log Script

A Bash script to automate directory backups in a simple and reliable way. The script accepts a source and a destination directory as command-line arguments.
It then handles the entire backup process:
1. Creates a timestamped .tar.gz archive of the source directory.
2. Securely moves the completed archive to the destination.
3. Writes a detailed log entry to **/var/log/custom_backup.log**, clearly stating if the backup was a 'SUCCESS' or 'FAILURE'.


## Features
1. Timestamped Backups: Archives are named with the exact date and time (e.g., Documents_2025-11-14_14:38:00.tar.gz).
2. Reliable Logging: Creates a permanent record of all backup attempts in /var/log/custom_backup.log.
3. Error Handling: Provides clear error messages for common problems, like invalid directories.
4. Safe Archiving: Creates the archive in /tmp first, ensuring the file is complete before moving it to the final destination.

## Usage
1. Make the script executable:
```bash
   chmod +x backup.sh
```
2. Run the script:
  ```bash
   ./backup.sh
  ``` 
   The script will then interactively ask you for the source and destination directories.

3. Run with sudo (for logging):
   To write to /var/log/, the script needs administrator privileges.
      ```bash
    sudo ./backup.sh
      ```  
   Note: If you run it without sudo, it will print a warning and write the log message to your terminal instead.

4. Check the logs:
   You can monitor all backup activity by checking the log file:
    ```bash
     tail -f /var/log/custom_backup.log
      ``` 

## Backup Script

```bash
#!/bin/bash

# Log Configuration
LOG_FILE="/var/log/custom_backup.log"

# Function to write log messages
log_message() {
    local level="$1"
    local message="$2"
    # Format: [YYYY-MM-DD HH:MM:SS] [LEVEL] - Message
    local timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    local log_entry="[$timestamp] [$level] - $message"

    # Check for write permission to the log file.
    if ! echo "" >> "$LOG_FILE" 2>/dev/null; then
        echo "WARNING: Permission denied. Cannot write to log file: $LOG_FILE" >&2
        echo "Log Entry: $log_entry" >&2  # Echo the log to stderr
    else
        # Write the actual log entry
        echo "$log_entry" >> "$LOG_FILE"
    fi
}

# Source request
echo "Please enter the source directory:"
read SOURCE_DIR

# Destination request
echo "Please enter the destination directory:"
read DEST_DIR

# Source validation
if [ ! -d "$SOURCE_DIR" ]; then
    echo "Error: Source directory '$SOURCE_DIR' does not exist."
    log_message "ERROR" "Source directory '$SOURCE_DIR' not found."
    exit 1  # Exit code 1: Invalid source
fi

# Destination validation
if [ ! -d "$DEST_DIR" ]; then
    echo "Error: Destination directory '$DEST_DIR' does not exist."
    log_message "ERROR" "Destination directory '$DEST_DIR' not found."
    exit 2  # Exit code 2: Invalid destination
fi

# Check write permission on destination
if ! touch "$DEST_DIR/.test_write" 2>/dev/null; then
    echo "Error: No write permission on destination '$DEST_DIR'."
    log_message "ERROR" "No write access to destination: $DEST_DIR"
    rm -f "$DEST_DIR/.test_write" 2>/dev/null
    exit 3  # Exit code 3: Destination unwritable
fi
rm -f "$DEST_DIR/.test_write"

TIMESTAMP=$(date +"%Y-%m-%d_%H:%M:%S")
# Base name of the source directory
SOURCE_BASENAME=$(basename "$SOURCE_DIR")
# Archive name
ARCHIVE_NAME="${SOURCE_BASENAME}_${TIMESTAMP}.tar.gz"

# Archive in a temporary location
TEMP_ARCHIVE_PATH="/tmp/$ARCHIVE_NAME"

# Parent directory of the source
PARENT_DIR=$(dirname "$SOURCE_DIR")
SOURCE_NAME=$(basename "$SOURCE_DIR")

echo "Starting backup of '$SOURCE_DIR'..."
log_message "INFO" "Backup started for source: $SOURCE_DIR"

# tar.gz archive
if tar -czf "$TEMP_ARCHIVE_PATH" -C "$PARENT_DIR" "$SOURCE_NAME"; then
    echo "Archive created: $TEMP_ARCHIVE_PATH"
    if mv "$TEMP_ARCHIVE_PATH" "$DEST_DIR/"; then
        echo "Backup successful!"
        echo "Archive '$ARCHIVE_NAME' moved to '$DEST_DIR'"
        log_message "SUCCESS" "Backup of '$SOURCE_DIR' completed. Archive: $DEST_DIR/$ARCHIVE_NAME"
        exit 0  # Exit code 0: Success
    else
        # Log move failure
        echo "Error: Failed to move archive '$TEMP_ARCHIVE_PATH' to '$DEST_DIR'"
        log_message "ERROR" "Archive created at '$TEMP_ARCHIVE_PATH' but 'mv' to '$DEST_DIR' failed."
        rm -f "$TEMP_ARCHIVE_PATH"  # Clean up temp file
        exit 5  # Exit code 5: Move failed
    fi
else
    # Log tar failure
    echo "Error: Backup failed during archive creation (tar command failed)." >&2
    log_message "ERROR" "Backup of '$SOURCE_DIR' failed. 'tar' command failed."
    rm -f "$TEMP_ARCHIVE_PATH"  # Clean up any partial temp file
    exit 4  # Exit code 4: Archive creation failed
fi
