#
# Copyright (c) 2019-2021 Red Hat, Inc.
# This program and the accompanying materials are made
# available under the terms of the Eclipse Public License 2.0
# which is available at https://www.eclipse.org/legal/epl-2.0/
#
# SPDX-License-Identifier: EPL-2.0
#
# Contributors:
#   Red Hat, Inc. - initial API and implementation
#

# Dockerfile to bootstrap build and test in openshift-ci

FROM registry.ci.openshift.org/openshift/release:golang-1.16

SHELL ["/bin/bash", "-c"]

# Install yq, kubectl, chectl cli used by olm/olm.sh script.
RUN yum install --assumeyes -d1 python3-pip  httpd-tools && \
    pip3 install --upgrade setuptools && \
    pip3 install yq && \
    pip3 install operator-courier==2.1.7 && \
    pip3 install pysemver && \
    curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && \
    chmod +x ./kubectl && \
    mv ./kubectl /usr/local/bin && \
    bash <(curl -sL https://www.eclipse.org/che/chectl/) --channel=next
