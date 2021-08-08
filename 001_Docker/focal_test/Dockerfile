# ベースイメージ[ローカルで作成したイメージを指定]
FROM focal:new

# aptのパッケージのリストを設定+Dockerのグループ追加のためにパッケージのインストール
# Dockerのインストールは「https://docs.docker.com/engine/install/ubuntu/」参照
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

#ユーザ追加
ARG USERNAME=user
ARG GROUPNAME=user
ARG UID=1000
ARG GID=1000
ARG PASSWORD=user
RUN groupadd -g $GID $GROUPNAME && \
    useradd -m -s /bin/bash -u $UID -g $GID -G sudo $USERNAME && \
    echo $USERNAME:$PASSWORD | chpasswd && \
    echo "$USERNAME   ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

#ユーザをdockerに追加
RUN usermod -aG docker $USERNAME

#ここに実行させたいコマンドを追記[CMDはDockerfileに1つだけ有効]
CMD ["/bin/bash"]

#作成したユーザに切り替え[デフォルトのログインはrootではなくユーザになる]
USER $USERNAME
