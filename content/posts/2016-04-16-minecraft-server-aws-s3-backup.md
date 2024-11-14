+++
author = "hannibal"
categories = ["Minecraft", "AWS", "Bash"]
date = "2016-04-16"
title = "Minecraft world automatic backup to AWS S3 bucket"
url = "/2016/04/16/minecraft-server-aws-s3-backup"

+++

Hi Folks.

Previously we created a Minecraft server using Docker. After my server got popular in the family, and a lot of stuff started to pile up on it, as a good IT person, I'm backing up the world once in a while.

For that, I'm using AWS S3 with the CLI and a little bash script which runs once a week.

The script is really straightforward. I'm doing manual versioning, although S3 does provide one out of the box. However, amazon's S3 versioning doesn't allow limiting the number of versions being kept. And since I'm doing that anyways, might as well take care of the rest.

Without further ado, here is the script:

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
	printf "Please set the env variable ${RED}MINECRAFT_BUCKET${NC} to the s3 archive bucket name.\n"
	exit 0
fi

if [[ -z ${MINECRAFT_ARCHIVE_LIMIT} ]]; then
	printf "Please set the env variable ${RED}MINECRAFT_ARCHIVE_LIMIT${NC} to limit the number of archives to keep.\n"
	exit 0
fi

backup_bucket=${MINECRAFT_BUCKET}
backup_limit=${MINECRAFT_ARCHIVE_LIMIT}
world=$1
printf "Creating archive of ${RED}${world}${NC}\n"
archive_name="${world}-$(date +"%H-%M-%S-%m-%d-%Y").zip"
zip -r $archive_name $world

printf "Checking if bucket has more than ${RED}${backup_limit}${NC} files already.\n"
content=( $(aws s3 ls s3://$backup_bucket | awk '{print $4}') )

if [[ ${#content[@]} -eq $backup_limit || ${#content[@]} -gt $backup_limit  ]]; then
    echo "There are too many archives. Deleting oldest one."
    # We can assume here that the list is in cronological order
	printf "${RED}s3://${backup_bucket}/${content[0]}\n"
    aws s3 rm s3://$backup_bucket/${content[0]}
fi

printf "Uploading ${RED}${archive_name}${NC} to s3 archive bucket.\n"
state=$(aws s3 cp $archive_name s3://$backup_bucket)

if [[ "$state" =~ "upload:" ]]; then
    printf "File upload ${LIGHT_GREEN}successful${NC}.\n"
else
    printf "${RED}Error${NC} occured while uploading archive. Please investigate.\n"
fi
~~~

It uses environment properties to define where to upload the given world and how many versions to keep.

I'm calling this from a cron job, and it's sitting next to where the Minecraft world is.

That's it folks.

I'll start expanding on this idea and implement various services, like your own server address, or dropbox, or what have you.

Happy backing up.

Gergely.
