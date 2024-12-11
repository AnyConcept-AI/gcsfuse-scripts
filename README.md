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


# GCloud Storage Region Copy Tool

This script facilitates copying data between Google Cloud Storage buckets in different regions while ensuring proper directory structure. It works in conjunction with the `update_folders.sh` script to maintain proper directory visibility in mounted buckets.

## Purpose
When you need to copy data between buckets in different regions while:
- Maintaining the exact directory structure
- Ensuring all directories are properly visible in mounted buckets
- Providing a safe preview and confirmation step before copying

## Features
- Recursive directory copying between regions
- Automatic directory structure repair after copying
- Preview of operations before execution
- Interactive confirmation prompt
- Works with wildcards in paths

## Prerequisites
- `gsutil` installed and configured
- `update_folders.sh` script in the same directory
- Appropriate permissions for both source and destination buckets

## Usage
```bash
./copy_between_regions.sh   
```

### Parameters
- `source_region`: Region code of the source bucket
- `destination_region`: Region code of the destination bucket
- `path`: Path to copy (can include wildcards)

### Examples
```bash
# Copy a specific directory
./copy_between_regions.sh us-east1 europe-west4 data/processed

# Copy multiple directories using wildcards
./copy_between_regions.sh us-east1 europe-west4 "data/processed/2024*"

# Copy entire bucket content
./copy_between_regions.sh us-east1 europe-west4 ""
```

## Script
```bash
#!/bin/bash

# Configuration - modify these variables to match your bucket naming pattern
BUCKET_PREFIX="anyconcept"

# Function to display usage
show_usage() {
    echo "Usage: $0   "
    echo
    echo "Parameters:"
    echo "  source_region        Region code of the source bucket (e.g., us-east1)"
    echo "  destination_region   Region code of the destination bucket (e.g., europe-west4)"
    echo "  path                 Path to copy (can include wildcards)"
    echo
    echo "Examples:"
    echo "  $0 us-east1 europe-west4 data/processed"
    echo "  $0 us-east1 europe-west4 \"data/processed/2024*\""
    exit 1
}

# Check if required arguments are provided
if [ $# -lt 3 ]; then
    show_usage
fi

# Check if update_folders.sh exists
if [ ! -f "./update_folders.sh" ]; then
    echo "Error: update_folders.sh script not found in current directory"
    exit 1
}

# Set variables
source_region=$1
destination_region=$2
path=$3
source_bucket="gs://$BUCKET_PREFIX-$source_region"
destination_bucket="gs://$BUCKET_PREFIX-$destination_region"

# Function to copy folders
copy_folders() {
    echo "Copying folders from $source_bucket/$path to $destination_bucket/$path"
    if ! gsutil -m rsync -r "$source_bucket/$path" "$destination_bucket/$path"; then
        echo "Error: Copy operation failed"
        exit 1
    fi
}

# Function to update folders
update_folders() {
    echo "Updating folders in $destination_bucket/$path"
    # Extract the base path without wildcards
    base_path=$(echo "$path" | sed 's/\*.*$//')
    if ! ./update_folders.sh "$base_path"; then
        echo "Error: Folder update operation failed"
        exit 1
    fi
}

# Validate bucket existence
if ! gsutil ls "$source_bucket" >/dev/null 2>&1; then
    echo "Error: Source bucket $source_bucket does not exist or is not accessible"
    exit 1
fi

if ! gsutil ls "$destination_bucket" >/dev/null 2>&1; then
    echo "Error: Destination bucket $destination_bucket does not exist or is not accessible"
    exit 1
fi

# Main execution
echo "Preview of the operation:"
echo "Source: $source_bucket/$path"
echo "Destination: $destination_bucket/$path"
echo
echo "This will:"
echo "1. Copy all content from source to destination"
echo "2. Update directory structure in the destination bucket"
echo

# Ask for confirmation
read -p "Do you want to proceed with the copy and folder update? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    copy_folders
    update_folders
    echo "Operation completed successfully."
else
    echo "Operation cancelled."
fi
```

## How It Works
1. The script first validates that all required parameters are provided
2. It constructs the full bucket paths using the region codes
3. Before proceeding, it verifies that both buckets exist and are accessible
4. It shows a preview of the operation and asks for confirmation
5. Upon confirmation:
   - Copies all content from source to destination using `gsutil -m rsync`
   - Runs the `update_folders.sh` script to ensure proper directory structure
6. Error handling ensures the script fails gracefully if any operation fails

## Improvements Made
- Added comprehensive error handling
- Added bucket existence validation
- Made bucket prefix configurable
- Added detailed usage information and examples
- Improved path handling with wildcards
- Added better progress feedback
- Added proper quoting for variables
- Added exit codes for error conditions

## Dependencies
This script requires:
1. The `update_folders.sh` script (must be in the same directory)
2. Google Cloud SDK with `gsutil` installed
3. Appropriate permissions on both source and destination buckets

## Notes
- The script assumes bucket names follow the pattern: `anyconcept-{region}`
- Modify the `BUCKET_PREFIX` variable if your bucket naming convention is different
- The script requires write permissions on both source and destination buckets
- `parallel`: GNU Parallel for parallel processing
- Mounted Google Cloud Storage bucket
- Appropriate permissions to create directories
