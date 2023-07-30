# syntax=docker/dockerfile:1
# setup ubuntu version
ARG os_image
ARG malware_hash=""
FROM $os_image

# setup wget and unzip, and p7zip
RUN apt-get update && \
    apt-get install -y wget p7zip-full p7zip-rar

# copy zip file
COPY ./home_content /home
