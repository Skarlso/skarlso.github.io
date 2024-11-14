+++
author = "hannibal"
categories = ["Minecraft", "AWS", "Bash"]
date = "2016-04-17"
title = "Minecraft world automatic backup to AWS S3 bucket - Part 2 (Custom functions)"
url = "/2016/04/17/minecraft-server-aws-s3-backup-part2"

+++

Hi folks.

Got an update for the backup script. This time, you'll have the ability to implement your own upload capabilities. I provide a mock implementation for the required functions.

Here is the script again, now modified and a bit cleaned up. I hope it's helpful.

<!--more-->

~~~bash
#!/bin/bash

if [[ -t 1 ]]; then
    colors=$(tput colors)
    if [[ $colors ]]; then
        RED='\033[0;31m'
        LIGHT_GREEN='\033[1;32m'
        NC='\033[0m'
    fi
fi

if [[ -z ${MINECRAFT_BUCKET} ]]; then
    printf "Please set the env variable %bMINECRAFT_BUCKET%b to the s3 archive bucket name.\n" "${RED}" "${NC}"
    exit 1
fi

if [[ -z ${MINECRAFT_ARCHIVE_LIMIT} ]]; then
    printf "Please set the env variable %bMINECRAFT_ARCHIVE_LIMIT%b to limit the number of archives to keep.\n" "${RED}" "${NC}"
    exit 1
fi

if [[ -z ${MINECRAFT_WORLD} ]]; then
    printf "Please set the env variable %bMINECRAFT_WORLD%b to specify what world to back-up.\n" "${RED}" "${NC}"
    exit 1
fi

backup_world=${MINECRAFT_WORLD}
backup_bucket=${MINECRAFT_BUCKET}
backup_limit=${MINECRAFT_ARCHIVE_LIMIT}
archive_name="${backup_world}-$(date +"%H-%M-%S-%m-%d-%Y").zip"

function create_archive {
    printf "Creating archive of %b${backup_world}%b\n" "${RED}" "${NC}"
    zip -r $archive_name $backup_world
}

function amazon_bak {

    create_archive

    printf "Checking if bucket has more than %b${backup_limit}%b files already.\n" "${RED}" "${NC}"
    content=( $(aws s3 ls s3://$backup_bucket | awk '{print $4}') )

    if [[ ${#content[@]} -eq $backup_limit || ${#content[@]} -gt $backup_limit  ]]; then
        echo "There are too many archives. Deleting oldest one."
        # We can assume here that the list is in cronological order
    	printf "%bs3://${backup_bucket}/${content[0]}\n%b" "${RED}" "${NC}"
        aws s3 rm s3://$backup_bucket/${content[0]}
    fi

    printf "Uploading %b${archive_name}%b to s3 archive bucket.\n" "${RED}" "${NC}"
    state=$(aws s3 cp $archive_name s3://$backup_bucket)

    if [[ "$state" =~ "upload:" ]]; then
        printf "File upload %bsuccessful%b.\n" "${LIGHT_GREEN}" "${NC}"
    else
        printf "%bError%b occured while uploading archive. Please investigate.\n" "${RED}" "${NC}"
    fi
}

function custom {
    if [[ -e custom.sh ]]; then
        source ./custom.sh
    else
        echo "custom.sh script not found. Please implement the apropriate functions."
        exit 1
    fi

    echo "Checking for the number of files. Limit is: $backup_limit."
    files=( $(list) )
    if [[ ${#files[@]} -eq $backup_limit || ${#files[@]} -gt $backup_limit ]]; then
        echo "Deleting extra file."
        delete ${files[0]}
        if [[ $? != 0 ]]; then
            printf "%bFailed%b to delete file. Please investigate failure." "${RED}" "${NC}"
            exit $?
        fi
    fi

    echo "Zipping world."
    create_archive

    echo "Uploading world."
    upload $archive_name

    if [[ $? != 0 ]]; then
        printf "%bFailed%b to upload archive. Please investigate the error." "${RED}" "${NC}"
        exit $?
    fi

    printf "Upload %bsuccessful%b" "${LIGHT_GREEN}" "${NC}"
}

function help {
    echo "Usage:"
    echo "./backup_world [METHOD]"
    echo "Exp.: ./backup_world aws|./backup_world custom|./backup_world dropbox"
    echo "Each method has it's own environment properties that it requires."
    echo "Global: MINECRAFT_WORLD|MINECRAFT_BUCKET|MINECRAFT_ARCHIVE_LIMIT"
    echo "Custom: Have a file, called 'custom.sh' which is sourced."
    echo "Implement these three functions: upload | list | delete."
    echo "upload -> should return exit code 0 on success, should return exit code 1 on failure."
    echo "list -> should return a list of cronologically ordered items."
    echo "delete -> should return exit code 0 on success, should return exit code 1 on failure."
}

case $1 in
    aws )
        amazon_bak
        ;;
    custom )
        custom
        ;;
    * )
        help
        ;;
esac
~~~

And here is the sample implementation for the custom upload functionality.

~~~bash
#!/bin/bash

function upload {
    echo "uploading"
    local result=0
    return $result
}

function delete {
    echo "deleting $1"
    local result=0
    return $result
}

function list {
    local arr=("file1" "file2" "file3")
    echo "${arr[@]}"
}
~~~

Thanks for reading!

Gergely.
