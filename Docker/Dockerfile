FROM container-registry.oracle.com/database/free:latest

USER root

# zip / unzip インストール + キャッシュ削除
# RUN yum install -y zip unzip \
#    && yum clean all \
#    && rm -rf /var/cache/yum

# GID=1300 のグループ作成（既存なら再利用）
RUN groupadd -g 1300 scc

# UID=12345 のユーザ作成（既存なら再利用）
RUN useradd -u 12345 -g 1300 sccuser

# oracleユーザをsccグループに追加
RUN usermod -aG 1300 oracle

# ディレクトリ作成・所有者・権限設定
RUN mkdir -p /aaa/bbb/ccc \
    && chown -R 12345:1300 /aaa/bbb/ccc \
    && chmod 755 /aaa/bbb/ccc

# 作業ディレクトリ
WORKDIR /aaa/bbb

# 実行ユーザを戻す（重要）
USER oracle
