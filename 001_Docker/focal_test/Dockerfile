# �x�[�X�C���[�W[���[�J���ō쐬�����C���[�W���w��]
FROM focal:new

# apt�̃p�b�P�[�W�̃��X�g��ݒ�+Docker�̃O���[�v�ǉ��̂��߂Ƀp�b�P�[�W�̃C���X�g�[��
# Docker�̃C���X�g�[���́uhttps://docs.docker.com/engine/install/ubuntu/�v�Q��
RUN echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal main restricted" > /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates main restricted" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal universe" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates universe" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal multiverse" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-updates multiverse" >> /etc/apt/sources.list \
    && echo "deb http://jp.archive.ubuntu.com/ubuntu/ focal-backports main restricted universe multiverse" >> /etc/apt/sources.list \
    && echo "deb http://security.ubuntu.com/ubuntu focal-security main restricted" >> /etc/apt/sources.list \
    && echo "deb http://security.ubuntu.com/ubuntu focal-security universe" >> /etc/apt/sources.list \
    && echo "deb http://security.ubuntu.com/ubuntu focal-security multiverse" >> /etc/apt/sources.list \
    && apt-get update -y && apt-get upgrade -y && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    lsb-release \
    && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
    && echo \
      "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null \
    && apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

#���[�U�ǉ�
ARG USERNAME=user
ARG GROUPNAME=user
ARG UID=1000
ARG GID=1000
ARG PASSWORD=user
RUN groupadd -g $GID $GROUPNAME && \
    useradd -m -s /bin/bash -u $UID -g $GID -G sudo $USERNAME && \
    echo $USERNAME:$PASSWORD | chpasswd && \
    echo "$USERNAME   ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

#���[�U��docker�ɒǉ�
RUN usermod -aG docker $USERNAME

#�����Ɏ��s���������R�}���h��ǋL[CMD��Dockerfile��1�����L��]
CMD ["/bin/bash"]

#�쐬�������[�U�ɐ؂�ւ�[�f�t�H���g�̃��O�C����root�ł͂Ȃ����[�U�ɂȂ�]
USER $USERNAME