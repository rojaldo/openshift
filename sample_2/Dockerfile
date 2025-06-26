FROM ubuntu:25.04 AS compiler
RUN apt-get update && \
    apt-get install -y build-essential && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
COPY ./sample.c /code/hello.c
WORKDIR /code
RUN gcc -o hello hello.c

FROM alpine:latest
RUN apk add --no-cache libc6-compat
COPY --from=compiler /code/hello /
RUN chmod +x /hello

# create a user and group
RUN addgroup -S hello-group && adduser -S hello-user -G hello-group \
    && mkdir -p /home/hello-user \
    && chown -R hello-user:hello-group /home/hello-user
    
USER hello-user

# run shell script
CMD ["/hello"]

