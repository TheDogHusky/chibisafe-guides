---
title: Using fswatch to watch a folder
summary: Learn how to use `fswatch` to monitor file changes and trigger actions
---

# Uploading files to chibisafe via `fswatch`

Not all screenshot utilities allow you to upload images directly to alternative providers like *chibisafe*. That being said, you can use a file change monitor to watch a folder and upload any new images to your *chibisafe* instance.

## Pre-requisites

1. The **`fswatch` binary**.  
Project Page: [emcrisostomo/fswatch](https://github.com/emcrisostomo/fswatch)
2. Your **Request URL**.  
Example: `https://foo.bar/api/upload`
3. Your **API Key**.  
Example: `GI4nAlK6gwy8T2BvmP31ZwaNbtgxACbXxQZb3nUiLcpPanl03f6WKxuaBkMfMy1C`
4. The **directory** where you want to **watch** for changes  
Example: `/home/catbox/screenshots/`
5. [OPTIONAL] Your **Album UUID**.  
Example: `NAiMaDpR-5xwR-LcEX-niWc-3e4N8Zp60zJe`

## Method 1: Using `crontab`

```sh
# Create a directory in which the script and the environment file will reside
mkdir -p "${HOME}/.chibisafe_watcher"

# Travel to the directory
cd "${HOME}/.chibisafe_watcher"

# Create script file and make it executable
touch chibisafe_watcher.sh
chmod +x chibisafe_watcher.sh

# Create environment file
touch chibisafe_watcher.env
```

Open `chibisafe_watcher.env` with your favorite editor and paste the following content, and replace the values with your own:

```sh
CHIBISAFE_REQUEST_URL=https://foo.bar/api/upload
CHIBISAFE_API_KEY=GI4nAlK6gwy8T2BvmP31ZwaNbtgxACbXxQZb3nUiLcpPanl03f6WKxuaBkMfMy1C
CHIBISAFE_WATCH_DIR=/home/catbox/screenshots/

# OPTIONAL
CHIBISAFE_ALBUM_UUID=NAiMaDpR-5xwR-LcEX-niWc-3e4N8Zp60zJe
```

Open `chibisafe_watcher.sh` with your favorite editor and paste the following content:

```sh
#!/usr/bin/env sh

setup_environment() {
    script_dir="$(dirname "$0")"
    script_dir="$(realpath "${script_dir}")"
    env_file="${script_dir}/chibisafe_watcher.env"

    if [ -f "${env_file}" ]; then
        source "${env_file}"
    else
        echo "No environment file found."
        exit 1
    fi
}

upload() {
    filename=$1
    dump_file="${CHIBISAFE_WATCH_DIR}/headers.dump"

    echo "${filename}" >> "${dump_file}"
    stat -f "%m" "${filename}" > "${dump_file}"
    curl -D "${dump_file}" -H "x-api-key: ${CHIBISAFE_API_KEY}" -H "albumuuid: ${CHIBISAFE_ALBUM_UUID}" -F "file[]=@${filename}" -s "${CHIBISAFE_REQUEST_URL}"
}

start_watch() {
    fswatch -0 "${WATCH_DIR}" | while read -d "" new_file
    do
        if [ -f "${new_file}" ]; then
            upload "${new_file}"
        fi
    done
}

setup_environment
start_watch
```

Open your crontab with `crontab -e` and add the following line, then save and exit the editor:

```sh
@reboot "~/.chibisafe_watcher/chibisafe_watcher.sh"
```

Upon your next reboot, the file change monitor will start watching the directory for changes. It will keep running and uploading new images until you terminate it.
