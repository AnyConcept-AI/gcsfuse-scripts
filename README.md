# gcsfuse-scripts
Convenience scripts to manage gcsfuse-mounted files and directories of a GCP Storage bucket. You can read about Google Cloud Storage FUSE here:

https://cloud.google.com/storage/docs/cloud-storage-fuse/overview


# GCloud Storage Directory Fix

This script resolves a common issue with Google Cloud Storage buckets where directory entries might be missing after bulk file copying operations (e.g., when using `gsutil -m cp -r`). While the files exist in the bucket, their parent directories might not be visible when browsing the mounted storage bucket.

## Problem
When copying files to a Google Cloud Storage bucket, especially with commands like `gsutil -m cp -r`, the directory entries themselves might not be created. This results in directories not being visible when browsing the mounted bucket, even though the files within them exist. This can be particularly problematic when working with deeply nested directory structures.

## Solution
This script automatically identifies and creates any missing directory entries in your mounted Google Cloud Storage bucket. It:
1. Scans the specified bucket (or subdirectory) for all object paths
2. Extracts unique directory paths
3. Creates any missing directories in the mounted bucket location

## Setup
1. Edit the script and set these variables:
   - `MOUNT_PT`: The local mount point of your storage bucket
   - `BUCKET_NAME`: The name of your Google Cloud Storage bucket

## Usage
```bash
./update_folders.sh [optional_path_to_subdirectory]
```

- Without arguments: Processes the entire bucket
- With argument: Processes only the specified subdirectory

## Examples
```bash
# Process entire bucket
./update_folders.sh

# Process only the 'data' subdirectory
./update_folders.sh data

# Process nested subdirectory
./update_folders.sh data/processed/2024
```

## Script
```bash
#!/bin/bash

# Configuration
MOUNT_PT=           # Set your bucket mount point (e.g., /mnt/gcs)
BUCKET_NAME=        # Set your bucket name (e.g., my-bucket)
OUTFILE=dir_names.txt
PARALLEL_JOBS=${2:-64}  # Number of parallel jobs, default 64 if not provided

# Ensure MOUNT_PT and BUCKET_NAME are set
if [[ -z "$MOUNT_PT" || -z "$BUCKET_NAME" ]]; then
    echo "Error: MOUNT_PT and BUCKET_NAME must be set in the script"
    exit 1
fi

# Handle optional subdirectory parameter
SUBDIR="${1:+$1/}"  # If $1 is set, append /, otherwise empty string

echo "Reading objects in gs://$BUCKET_NAME/${SUBDIR:-}"

# List objects and extract directory paths
gsutil ls -r "gs://$BUCKET_NAME/$SUBDIR"** |
    sed -n "s|^gs://$BUCKET_NAME/\(.*\)/.*|\1|p" |
    sort -u > "$OUTFILE"

echo "Processing directories found"

# Function to create directories
create_dir() {
    local LOCAL_DIR="$1"
    local TARG_DIR="$MOUNT_PT/$LOCAL_DIR"
    if [[ ! -d "$TARG_DIR" ]]; then
        echo "Creating $TARG_DIR"
        mkdir -p "$TARG_DIR"
    fi
}

export -f create_dir
export MOUNT_PT

# Process directories in parallel
grep "^${1:-}" "$OUTFILE" | parallel -j "$PARALLEL_JOBS" create_dir

rm "$OUTFILE"

echo "Process complete"
```

## Requirements
- `gsutil`: Google Cloud Storage command-line tool
- `parallel`: GNU Parallel for parallel processing
- Mounted Google Cloud Storage bucket
- Appropriate permissions to create directories
