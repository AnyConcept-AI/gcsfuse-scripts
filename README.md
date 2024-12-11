# gcsfuse-scripts
Convenience scripts to manage gcsfuse-mounted files and folders

https://cloud.google.com/storage/docs/cloud-storage-fuse/overview

update_folders.sh - Please fill in `MOUNT_PT` and `BUCKET_NAME`

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
