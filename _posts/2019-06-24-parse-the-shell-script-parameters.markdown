---
layout: post
title:  "Parse the shell script parameters"
date:   2019-06-24 00:00:00
categories: Shell
---

I've written multiple shell scripts for our team, but before every time I was going to write them, my head was empty. No matter how many times I had written, they were still not staying at my head! (Do you have the same feelings?)

Months ago, I had built the Redis cluster in the Kubernetes, but not expose the ports to external.

If my workmates wanted to manipulate the Redis data, they had to update the Kubernetes service temporarily or using the `kubectl` commands (which they are not familiar with).

Thus, I wrote a simple script for them to manipulate the Redis data behind the Kubernetes.

Here is the script snippet.
In this snippet, I was using `getopt` to parse the input parameters, for the details about it, check [OSX 10.6.2 - man page for getopt (osx section 1)](https://www.unix.com/man-page/osx/1/getopt/).

```shell
#!/usr/bin/env bash

NAMESPACE=
REDIS_DEPLOYMENT_NAME=
REDIS_DB_INDEX=
REDIS_KEY_PATTERN=

usage() {
    echo
    echo "Usage:"
    echo "  ${0} <command> [options]"
    echo
    echo "Use \"${0} <command> --help\" for more information about a given command."
    echo "Use \"${0} options\" for a list of global command-line options (applies to all commands)."
    echo
    echo "Commands:"
    echo "  del                 Delete keys by a pattern."
    echo "  command             Execute redis command on the master node."
    echo
    exit 0
}

usage_del() {
    echo
    echo "Delete keys by a pattern."
    echo
    echo "Usage:"
    echo "  ${0} del [options]"
    echo
    echo "Options:"
    echo "  -h,--help:                   Help"
    echo "  -n,--namespace:              The namespace."
    echo "  -d,--deployment:             The deployment name of redis."
    echo "  -i,--index:                  The redis database index you want to manipulate."
    echo "  -p,--pattern:                The pattern of keys you want to delete."
    echo
    exit 0
}

usage_command() {
    echo
    echo "Execute redis command on the master node."
    echo
    echo "Usage:"
    echo "  ${0} command [options] -- COMMAND [args...]"
    echo
    echo "Options:"
    echo "  -h,--help:                   Help"
    echo "  -n,--namespace:              The namespace."
    echo "  -d,--deployment:             The deployment name of redis."
    echo "  -i,--index:                  The redis database index you want to manipulate."
    echo
    exit 0
}

missing() {
    [[ -n $1 ]] && message="Missing required option: $1" || message="Missing command."
    echo
    echo "${message}"
    echo "For help: ${0} ${COMMAND} -h"
    echo 
    exit 1
}

check_global_options() {
    [[ -z ${NAMESPACE} ]] && missing "-n|--namespace"
    [[ -z ${REDIS_DEPLOYMENT_NAME} ]] && missing "-d|--deployment"
    [[ -z ${REDIS_DB_INDEX} ]] && missing "-i|--index"
}

_del() {
    options=$(getopt -l "help,namespace:,deployment:,index:,pattern:" -o "hn:d:i:p:" -a -- "$@")
    eval set -- "$options"

    while true; do
        case $1 in
            -h|--help) usage_del;;
            -n|--namespace) shift; NAMESPACE=$1;;
            -d|--deployment) shift; REDIS_DEPLOYMENT_NAME=$1;;
            -i|--index) shift; REDIS_DB_INDEX=$1;;
            -p|--pattern) shift; REDIS_KEY_PATTERN=$1;;
            --) shift; break;;
            *) echo "Unknown options: $1"; usage;;
        esac
        shift
    done

    # check global options
    check_global_options

    # check command options
    [[ -z ${REDIS_KEY_PATTERN} ]] && missing_option "-p|--pattern"

    _exec "
    execute() {
        redis-cli -h ${REDIS_DEPLOYMENT_NAME}-master -a ${REDIS_PASSWORD} -n ${REDIS_DB_INDEX} \$@
    }

    export -f execute
    execute \"${REDIS_KEY_PATTERN}\" | xargs -I {} bash -c 'execute DEL \"\$@\"' _ {}
    "
}

_command() {
    options=$(getopt -l "help,namespace:,deployment:,index:" -o "hn:d:i:" -a -- "$@")
    eval set -- "$options"
    
    while true; do
        case $1 in
            -h|--help) usage_command;;
            -n|--namespace) shift; NAMESPACE=$1;;
            -d|--deployment) shift; REDIS_DEPLOYMENT_NAME=$1;;
            -i|--index) shift; REDIS_DB_INDEX=$1;;
            --) shift; REDIS_COMMAND=$@; break;;
            *) echo "Unknown options: $1"; usage;;
        esac
        shift
    done

    # check global options
    check_global_options

    # check command options
    [[ -z ${REDIS_COMMAND} ]] && missing

    _exec "redis-cli -h ${REDIS_DEPLOYMENT_NAME}-master -a ${REDIS_PASSWORD} -n ${REDIS_DB_INDEX} ${REDIS_COMMAND}"
}

_exec() {
    # arguments
    COMMAND=$1

    # retrieve the password
    REDIS_PASSWORD=$(kubectl get secret --namespace ${NAMESPACE} ${REDIS_DEPLOYMENT_NAME} -o jsonpath="{.data.redis-password}" | base64 --decode)

    # execute the redis command in a temporary sandbox
    kubectl run --namespace ${NAMESPACE} ${REDIS_DEPLOYMENT_NAME}-client --rm --tty -i --restart='Never' \
    --env REDIS_PASSWORD=${REDIS_PASSWORD} \
    --image docker.io/bitnami/redis:4.0.12 -- bash -c ${COMMAND}
}

COMMAND=$1
case $1 in
    -h|--help) usage;;
    del) shift; _del $@;;
    command) shift; _command $@;;
    *) echo "Unknown options: $1"; usage;;
esac
```