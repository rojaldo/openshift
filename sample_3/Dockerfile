FROM ubuntu:25.04
# install figlet and clean cache

LABEL maintainer="Your Name"
LABEL description="This Dockerfile installs figlet on an Ubuntu 25.04 base image."
LABEL version="1.0"

RUN apt update  \
    && apt install -y figlet  \
    && apt clean \
    && rm -rf /var/lib/apt/lists/*

ARG ARG_MSG="Hola Mundo!"

ENV MESSAGE=$ARG_MSG

COPY ./content  home/code

ONBUILD RUN rm /home/code

# create a user and group
RUN groupadd -r figletgroup && useradd -r -g figletgroup figletuser \
    && mkdir -p /home/figletuser \
    && chown -R figletuser:figletgroup /home/figletuser

USER figletuser
WORKDIR /home/code

ENTRYPOINT figlet
CMD $MESSAGE

STOPSIGNAL SIGTERM

