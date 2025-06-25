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
# run shell script
CMD ["/hello"]