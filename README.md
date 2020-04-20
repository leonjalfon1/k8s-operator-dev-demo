# CREATING YOUR FIRST KUBERNETES OPERATOR

## Prerequisites

1. Install Golang

```
https://golang.org/doc/install
```

2. Setup go workspace

```
mkdir -p $HOME/go
go env -w GOPATH=$HOME/go
```

3. Install the Operator SDK CLI

```
$ brew install operator-sdk
```

## Create a new project

1. Use the CLI to create a new memcached-operator project

```
mkdir -p $GOPATH/src/github.com/demo/
cd $GOPATH/src/github.com/demo/
export GO111MODULE=on
operator-sdk new memcached-operator
```

2. Open the project with visual studio code

```
cd memcached-operator
code .
```

3. Install dependencies by run

```
go mod tidy
```