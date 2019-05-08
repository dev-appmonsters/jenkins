FROM docker:stable-dind
#USER root

#Install python
RUN apk add --no-cache python3 && \
    python3 -m ensurepip && \
    rm -r /usr/lib/python*/ensurepip && \
    pip3 install --upgrade pip setuptools && \
    if [ ! -e /usr/bin/pip ]; then ln -s pip3 /usr/bin/pip ; fi && \
    if [[ ! -e /usr/bin/python ]]; then ln -sf /usr/bin/python3 /usr/bin/python; fi && \
    rm -r /root/.cache

#Install curl && netcat && bash
RUN apk add --update --no-cache  curl netcat-openbsd bash bash-completion

#Install git && cmake && jq
RUN apk add --update --no-cache git cmake jq

#Install docker compose
RUN apk add py-pip python-dev libffi-dev openssl-dev gcc libc-dev make
RUN pip install docker-compose

#Install Ruby
RUN apk update && apk upgrade && apk add ruby alpine-sdk ruby-dev ruby-bundler ruby-json ruby-irb ruby-rake ruby-bigdecimal
RUN gem install rdoc kubernetes-deploy --no-document

#Install Gcloud-cli
RUN curl -sSL https://sdk.cloud.google.com | bash
ENV PATH $PATH:/root/google-cloud-sdk/bin

#Install helm and kubectl
#RUN  curl -LO https://git.io/get_helm.sh && chmod 700 get_helm.sh && ./get_helm.sh
ENV KUBE_LATEST_VERSION="v1.13.4"
ENV HELM_VERSION="v2.13.1"
RUN apk add --no-cache ca-certificates bash git openssh curl \
    && wget -q https://storage.googleapis.com/kubernetes-release/release/${KUBE_LATEST_VERSION}/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl \
    && wget -q https://storage.googleapis.com/kubernetes-helm/helm-${HELM_VERSION}-linux-amd64.tar.gz -O - | tar -xzO linux-amd64/helm > /usr/local/bin/helm \
    && chmod +x /usr/local/bin/helm

#Install yaml and os
RUN apk add --update --no-cache py-yaml

ENTRYPOINT ["dockerd-entrypoint.sh"]
