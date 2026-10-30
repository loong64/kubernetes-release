# Copyright 2020 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# This file creates a standard build environment for building cross
# platform go binary for the architecture kubernetes cares about.

ARG BASEIMAGE
ARG GO_VERSION
FROM ghcr.io/loong64/golang:${GO_VERSION}-alpine as go-source
FROM ghcr.io/loong64/${BASEIMAGE}

##------------------------------------------------------------
# global ARGs & ENVs
ARG TARGETPLATFORM
ARG DEBIAN_FRONTEND=noninteractive

ENV GOARM 7

# the default value for KUBE_DYNAMIC_CROSSPLATFORMS and KUBE_CROSSPLATFORMS is set for
# backward compatibility with the old Dockerfile. 
ARG KUBE_DYNAMIC_CROSSPLATFORMS="arm64 armhf i386 loong64 ppc64el s390x"
ARG KUBE_CROSSPLATFORMS="linux/386 linux/arm linux/arm64 linux/loong64 linux/ppc64le linux/s390x darwin/amd64 windows/amd64 windows/386"

ENV KUBE_DYNAMIC_CROSSPLATFORMS=${KUBE_DYNAMIC_CROSSPLATFORMS}
ENV KUBE_CROSSPLATFORMS=${KUBE_CROSSPLATFORMS}

##------------------------------------------------------------
COPY --from=go-source /usr/local/go /usr/local/go
ENV PATH=/usr/local/go/bin:$PATH
ENV GOROOT=/usr/local/go
ENV GOLANG_VERSION=${GO_VERSION}
ENV GOTOOLCHAIN=auto
ENV GOPATH=/usr/local/go

# Pre-compile the standard go library when cross-compiling. This is much easier now when we have go1.5+
RUN targetArch=$(echo $TARGETPLATFORM | cut -f2 -d '/') \
    && if [ ${targetArch} = "amd64" ]; then \
    for platform in ${KUBE_CROSSPLATFORMS}; do GOOS=${platform%/*} GOARCH=${platform##*/} go install std; done \
    && go clean -cache; \
fi

# Install packages
RUN apt-get -q update \
    && apt-get install -qqy \
        apt-utils \
        curl \
        file \
        git \
        gnupg2 \
        iproute2 \
        jq \
        openssh-client \
        patch \
        rsync \
        unzip \
        wget

# Use dynamic cgo linking for architectures other than amd64 for the server platforms
# To install crossbuild essential for other architectures add the following repository.
RUN targetArch=$(echo $TARGETPLATFORM | cut -f2 -d '/') \
  && if [ ${targetArch} = "amd64" ]; then \
    apt-get update \
    && apt-get install -y build-essential mingw-w64 \
    && for platform in ${KUBE_DYNAMIC_CROSSPLATFORMS}; do \
      case ${KUBE_DYNAMIC_CROSSPLATFORMS} in \
        "loong64") apt-get install -y g++-loongarch64-linux-gnu ;; \
        *) apt-get install -y crossbuild-essential-${platform} ;; \
      esac; \
    done \
fi

RUN targetArch=$(echo $TARGETPLATFORM | cut -f2 -d '/') \
  && if [ ${targetArch} = "arm64" ]; then \
    apt-get update \
    && apt-get install -y build-essential gcc-x86-64-linux-gnu \
    && apt-get install -y crossbuild-essential-ppc64el crossbuild-essential-s390x g++-loongarch64-linux-gnu; \
fi

RUN targetArch=$(echo $TARGETPLATFORM | cut -f2 -d '/') \
  && if [ ${targetArch} = "loong64" ] || [ ${targetArch} = "ppc64le" ] || [ ${targetArch} = "s390x" ]; then \
    apt-get update \
    && apt-get install -y build-essential; \
fi

ENV PROTOBUF_VERSION 23.4
RUN targetArch=$(echo $TARGETPLATFORM | cut -f2 -d '/') \
  && if [ ${targetArch} = "amd64" ]; then \
  ZIPNAME="protoc-${PROTOBUF_VERSION}-linux-x86_64.zip"; \
elif [ ${targetArch} = "arm64" ]; then \
  ZIPNAME="protoc-${PROTOBUF_VERSION}-linux-aarch_64.zip"; \
elif [ ${targetArch} = "loong64" ]; then \
  ZIPNAME="protoc-${PROTOBUF_VERSION}-linux-loong64.zip"; \
elif [ ${targetArch} = "ppc64le" ]; then \
  ZIPNAME="protoc-${PROTOBUF_VERSION}-linux-ppcle_64.zip"; \
elif [ ${targetArch} = "s390x" ]; then \
  ZIPNAME="protoc-${PROTOBUF_VERSION}-linux-s390_64.zip"; \
fi \
  && mkdir /tmp/protoc && cd /tmp/protoc \
  && \
  if [ ${targetArch} = "loong64" ]; then \
    wget "https://github.com/Loongson-Cloud-Community/protobuf/releases/download/v${PROTOBUF_VERSION}/${ZIPNAME}"; \
  else \
    wget "https://github.com/protocolbuffers/protobuf/releases/download/v${PROTOBUF_VERSION}/${ZIPNAME}"; \
  fi \
  && unzip "${ZIPNAME}" \
  && chmod -R +rX /tmp/protoc \
  && cp -pr bin /usr/local \
  && cp -pr include /usr/local \
  && rm -rf /tmp/protoc \
  && protoc --version

# work around 64MB tmpfs size in Docker 1.6
ENV TMPDIR /tmp.k8s
RUN mkdir $TMPDIR \
  && chmod a+rwx $TMPDIR \
  && chmod o+t $TMPDIR

# Get the code coverage tool and goimports
RUN for i in {1..5}; do GOPROXY="direct" go install golang.org/x/tools/cmd/cover@latest && break || sleep 15; done  \
    && for i in {1..5}; do GOPROXY="direct" go install golang.org/x/tools/cmd/goimports@latest && break || sleep 15; done  \
    && go clean -cache

# Cleanup a bit
RUN apt-get -qqy remove \
      wget \
    && apt-get clean \
    && rm -rf -- \
        /var/lib/apt/lists/*

ENTRYPOINT []
