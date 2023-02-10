FROM ubuntu:22.04

# Install applications
RUN apt-get update && apt-get install -y python3 
RUN apt install -y software-properties-common
RUN add-apt-repository --yes --update ppa:ansible/ansible
RUN apt install -y ansible
RUN apt install -y net-tools
RUN apt install -y iputils-ping

COPY ./ssh_config/* /etc/ssh/ssh_config.d/

WORKDIR /etc/ansible/