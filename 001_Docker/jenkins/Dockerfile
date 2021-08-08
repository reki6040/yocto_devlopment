# 1.1.1.
FROM jenkins:2.60.3

# 1.1.2.
USER root

# 1.1.3.
RUN apt-get update && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    expect \
    software-properties-common \
    && curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add - \
    && apt-key fingerprint 0EBFCD88 \
    && add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
    && apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# 1.1.4.
RUN usermod -aG docker jenkins

# 1.1.5.
RUN cd /tmp/ \
    && wget https://updates.jenkins-ci.org/latest/jenkins.war \
    && mv ./jenkins.war /usr/share/jenkins/

# 1.1.6.
USER jenkins
