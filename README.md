# gcsfuse-scripts
Convenience scripts to manage gcsfuse-mounted files and directories of a GCP Storage bucket 

https://cloud.google.com/storage/docs/cloud-storage-fuse/overview

This script helps with the problem that when copying many files in a storage bucket with the `gsutil -m cp -r` command and in other cases, the directory themselves might not have been created. They are then simply not shown when browsing the mounted storage bucket even though all the files exist. When working with deeply nested directory structures, it is a pain to create these directory entries manually, so this script simply finds those missing directories and creates them. It can therefore simply be run to make sure all the files that exist in the storage bucket can be found.

Update Folders: Please fill in `MOUNT_PT` and `BUCKET_NAME`
Usage: `./update_folders.sh [optional_path_to_subdirectory]`

```
#!/bin/bash

MOUNT_PT=
BUCKET_NAME=
OUTFILE=dir_names.txt
PARALLEL_JOBS=${2:-64}  # Number of parallel jobs, default 8 if not provided

echo "Reading objects in $BUCKET_NAME"

gsutil ls -r gs://$BUCKET_NAME/$1/** |
    sed -n "s|^gs://$BUCKET_NAME/\(.*\)/.*|\1|p" |
    sort -u > $OUTFILE

echo "Processing directories found"

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

grep "^$1" "$OUTFILE" | parallel -j "$PARALLEL_JOBS" create_dir

rm "$OUTFILE"

echo "Process complete"
```
