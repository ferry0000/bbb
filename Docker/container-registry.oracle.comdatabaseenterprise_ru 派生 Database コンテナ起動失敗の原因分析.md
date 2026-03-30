# container-registry.oracle.com/database/enterprise_ru 派生 Database コンテナ起動失敗の原因分析

## エグゼクティブサマリー

企業向けのOracle Databaseコンテナ（`container-registry.oracle.com/database/enterprise_ru`）をベースに「ディレクトリ作成・ユーザー作成」を行う程度のカスタムDockerfileを追加しただけでDBインスタンスが起動しなくなる場合、原因はアプリ側ではなく **ベースイメージの“起動不変条件”を壊している**ケースが多いです。特に確率が高いのは、(a) 最終的な実行ユーザーが `oracle (uid=54321)` ではなくなった、(b) `CMD/ENTRYPOINT` を上書きして `runOracle.sh` が実行されなくなった、(c) `/opt/oracle/oradata`（永続ボリューム）に **uid 54321 が書けない**・SELinuxラベル不整合、(d) cgroupメモリ制限が 2GB 未満、(e) ulimit（nofile/nproc/stack/memlock）不足です。citeturn13view0turn7view0turn5view1turn25view0turn1view1

`runOracle.sh` は **起動時にcgroupのメモリ制限（cgroup v2/v1）を読んで 2GB 未満なら即終了**します。したがって「コンテナは起動するがDBが上がらない」以前に「プロセスが早期exitしてしまう」パターンも起きます。citeturn25view0

本レポートは、Dockerfile・起動コマンド・ホストOS・ログが未提示という前提のため、**不明点は不明点として明示**しつつ、収集すべき情報、診断コマンド、最有力原因の優先度、修正パターン（最小Dockerfile例）とログ抽出方法を、Oracle公式/準公式（Oracle docs・oracle/docker-images・Docker公式docs）を軸に整理します。citeturn12search10turn4view0turn1view1turn26search5turn17search1

## 前提と不明点

本件は「`enterprise_ru` をFROMにして、ユーザーとディレクトリを追加した派生イメージが、Oracle DBインスタンス起動に失敗する」という問いですが、現時点で次が未提示です（＝原因の特定を決定づける一次情報）。  

- カスタムDockerfile（またはContainerfile）の全文  
- build手段（docker build / podman build / buildah / CI）と buildログ  
- 実行コマンド（docker run / compose / Kubernetesマニフェスト / Podman systemd unit など）  
- ベースタグ（例：`enterprise_ru:19.xx` / `enterprise_ru:21.xx`）とダイジェスト  
- ホストOS、カーネル、SELinux/AppArmor、Docker/OCI runtime種別（docker/containerd/cri-o など）  
- コンテナログ（`docker logs` 相当）と Oracle診断ログ（alert.log等）

この「未提示情報」を最短で集めるためのコマンドは、後段の「診断チェックリスト」に具体化します。citeturn18search1turn18search35turn25view0turn14view5turn5view1

## ベースイメージの起動仕様と不変条件

### 起動の“正規ルート”は runOracle.sh 実行である

Oracle公式のDockerイメージ群（19.3以降のSingleInstance系）は、**起動時に `runOracle.sh` を実行すること**を前提とし、コンテナが落ちないように最後は **alert.logをtailして待機**する設計です（これがPID 1相当の居座りになります）。citeturn10view1turn14view5turn14view6

また、イメージ側には `HEALTHCHECK` が組み込まれており、既定では `checkDBStatus.sh` を実行します。citeturn10view1turn16view0

「派生Dockerfileでディレクトリやユーザーを追加しただけ」でも、以下を変えると起動経路が崩壊します。  
- 最終レイヤの `USER` が `oracle` でなくなる（または `docker run --user` 等で上書きされる）  
- `CMD` や `ENTRYPOINT` を再定義して `runOracle.sh` が呼ばれなくなる  
- `runOracle.sh` 自体の実行権限が失われる（Kubernetes等で `Permission denied` になる事例が複数報告）citeturn10view1turn13view0turn17search3

### oracle ユーザー uid=54321 と /opt/oracle/oradata の書き込みが最重要

Oracle docsとoracle/docker-imagesのREADME/FAQは一貫して、データ領域ボリューム `/opt/oracle/oradata` は **コンテナ内のoracleユーザー（uid=54321）が書き込み可能であること**を要求しています。citeturn5view1turn7view0turn13view0

この要件は、ホストへバインドマウントする場合に問題化します。Dockerは通常、コンテナ内uidをホスト側にそのまま写像するため、ホスト側のディレクトリ所有・権限が合っていないと「作成できない」「Permission denied」になり、DB作成・起動が失敗します。citeturn13view0turn5view1

### cgroupメモリ制限が 2GB 未満だと runOracle.sh が明示的に終了する

`runOracle.sh` は、cgroup v2 なら `/sys/fs/cgroup/memory.max`、cgroup v1 なら `/sys/fs/cgroup/memory/memory.limit_in_bytes` を参照し、**2GB未満なら“DBコンテナには最低2GB必要”として終了**します。citeturn25view0

これは「Dockerfileを変えたせい」に見えて、実は **起動コマンド（`--memory`）やKubernetes resources.limits の変更**で発生していることもあります。citeturn26search0turn25view0

### セットアップ/スタートアップスクリプトは“所定ディレクトリ”で実行される

Oracle docsは、セットアップ後/起動後に実行するユーザースクリプトを置く場所として `/opt/oracle/scripts/setup` と `/opt/oracle/scripts/startup`（および互換用の `/docker-entrypoint-initdb.d` シンボリックリンク）を示しています。citeturn5view1turn7view3

従って、派生Dockerfileでこのディレクトリやシンボリックリンクを削除・置換・権限変更すると、「起動が進む前提（例：sqlnet.ora生成、初期ユーザ作成）を失う」可能性があります。citeturn7view3turn14view2

### enterprise_ru についての位置づけ

`enterprise_ru` はOracle Container Registry上のDBサーバイメージ群の一つで、タグに `19.27.0.0` のようなRU相当の番号が現れることがコミュニティやOracle Support情報から確認できます（ただし本レポートの環境固有のタグは不明）。citeturn2search1turn2search10  
またoracle/docker-images側では、Release Update等を取り込む「patched container image」ビルド手順（`-p`）が記載されています。Oracleが配布するRU系イメージはこの系譜と整合すると推測できます（推測である点は留保）。citeturn1view1

### 起動フロー図

以下は、19.3以降のSingleInstance系スクリプト設計に基づく起動フロー（代表例）です。`enterprise_ru` でも、少なくとも19.3以降は同等の挙動を前提に問題切り分けするのが合理的です。citeturn13view0turn10view1turn25view0turn14view5turn16view0

```mermaid
flowchart TD
  A[Container start] --> B[Default CMD runs runOracle.sh]
  B --> C{cgroup memory >= 2GB?}
  C -- no --> C1[Exit with error message]
  C -- yes --> D[Normalize/validate ORACLE_SID/ORACLE_PDB]
  D --> E{DB exists? checkpoint + data dir}
  E -- yes --> F[symlink config files + ensure adump + startDB.sh]
  E -- no --> G[cleanup incomplete DB + createDB.sh (DBCA)]
  G --> H[runUserScripts (setup/startup)]
  F --> I[touch /dev/shm/.db_started]
  H --> I
  I --> J[tail alert*.log to keep container alive]
  J --> K[HEALTHCHECK: checkDBStatus.sh (marker + SQLPlus checks)]
```

## 失敗要因の優先度付き分析

下表は「ユーザー/ディレクトリ追加だけで起動しなくなった」という症状に対し、経験則ではなく **スクリプト/ドキュメント上の前提（不変条件）**から逆算した“起こりやすさ順”です。citeturn10view1turn13view0turn25view0turn7view0turn17search1

| 優先度 | 典型症状 | 技術的説明（何が壊れているか） | 裏取りの要点 |
|---|---|---|
| 高 | コンテナが即Exit、またはログに「memory」関連エラー | `runOracle.sh` はcgroup値を読んで2GB未満なら終了する。`docker run --memory` やKubernetes limitで簡単に踏む。citeturn25view0turn26search0 | `docker logs` の先頭にメモリ不足メッセージがあるか、`docker inspect` の Memory/Resources を確認 |
| 高 | `Cannot create directory` / `Permission denied` / DBファイル作成失敗 | `/opt/oracle/oradata` は `oracle(uid 54321)` が書ける必要がある。ホストバインドマウントが uid 54321 で書けない、またはSELinuxラベル不一致。citeturn5view1turn13view0turn17search1 | ホスト側ディレクトリの所有/権限（数値UID）、SELinux（`:Z`/`:z`） |
| 高 | DBが起動しない／そもそもrunOracleが走ってない（ログが静か、または別プロセスがPID1） | 派生Dockerfileで `CMD` を書くとベースの `CMD exec runOracle.sh` を置き換える。`ENTRYPOINT` 変更も同様。citeturn10view1turn26search0turn26search5 | `docker image inspect` で `.Config.Cmd`/`.Config.Entrypoint` をベースと比較 |
| 高 | `sqlplus / as sysdba` が失敗、healthcheckが落ち続ける | `USER` が `oracle` でなくなる（Dockerfile最終USER、または `--user`/K8s runAsUser）と、OS認証/ファイル権限/書込が崩れる。特にOpenShift等の“任意UID”は相性が悪い。citeturn13view0turn12search2turn16view0 | `docker inspect` の User、コンテナ内 `id`、K8sなら securityContext（runAsUser/fsGroup） |
| 中 | Kubernetesで `/opt/oracle/runOracle.sh: Permission denied` | セキュリティコンテキストやファイルモード（実行ビット）次第で `runOracle.sh` が実行できない事例がある。citeturn17search3turn10view1 | `ls -l /opt/oracle/runOracle.sh`、マニフェストの `readOnlyRootFilesystem` 等 |
| 中 | “unhealthy” → 再起動ループ（実体はDB起動済みの可能性） | healthcheckは `/dev/shm/.db_started` を前提にSQL*PlusでPDB open等を検査。`/dev/shm` のマウントや作成不可で落ちると orchestrator が再起動する。citeturn16view0turn14view4turn13view0 | `.State.Health.Log`、`/dev/shm` 容量/オプション、markerファイル確認 |
| 中 | ulimit関連エラーや“out of memory”（FD/プロセス不足） | READMEの推奨 `--ulimit`（nofile/nproc/stack/memlock）を満たさないと失敗し得る。日本語事例でも `--ulimit` で改善報告。citeturn1view1turn2search3 | `docker inspect .HostConfig.Ulimits`、`ulimit -a`、ログにFD不足が出るか |
| 低 | AppArmor/SECCompで拒否（Operation not permitted） | DockerはデフォルトAppArmorプロファイルを適用。稀に特定操作が拒否されることがある。citeturn17search0turn17search12 | `dmesg`/`journalctl -k` のDENIED、`--security-opt` 変更で再現性 |
| 低 | ホスト名に `_` が入りネットワーク系が不安定 | `runOracle.sh` がアンダースコアを警告する。環境によっては接続やリスナーに悪影響が出る。citeturn25view0 | `hostname`、ログの警告行 |

## 診断チェックリストとコマンド

この章は「何を見れば原因が確定するか」を、ツール別に“コピペで実行できる粒度”でまとめます。特に **(1) ベースと派生の差分（USER/CMD/ENV）** と **(2) /opt/oracle/oradata の権限** と **(3) cgroupメモリ** が最短経路です。citeturn10view1turn13view0turn25view0turn5view1

### まず集めるべき一次情報

#### コンテナが落ちている場合でも必ず取れるもの
```bash
# 1) コンテナ一覧と終了理由
docker ps -a --no-trunc

# 2) コンテナstdout/stderr（runOracle.shはalert.logもtailして出す設計）
docker logs --tail 300 <container>

# 3) 設定・制限・マウント・ログパスを丸ごと保存
docker inspect <container> > inspect.container.json

# 4) コンテナログの実体ファイル場所（json-file等の場合）
docker inspect --format='{{.LogPath}}' <container>
```
`docker logs` はSTDOUT/STDERRを取得するコマンドとしてDocker公式が明示しています。citeturn18search35  
`docker inspect --format '{{.LogPath}}'` はDocker docsの利用例として提示されています。citeturn18search1

#### イメージの“起動不変条件”を壊していないかの差分チェック
```bash
# ベース（enterprise_ru）と派生イメージで比較する
docker image inspect container-registry.oracle.com/database/enterprise_ru:<tag> \
  --format 'USER={{.Config.User}} ENTRYPOINT={{json .Config.Entrypoint}} CMD={{json .Config.Cmd}}'

docker image inspect <your-custom-image>:<tag> \
  --format 'USER={{.Config.User}} ENTRYPOINT={{json .Config.Entrypoint}} CMD={{json .Config.Cmd}}'

# 環境変数（ORACLE_BASE等を上書きしていないか）
docker image inspect <your-custom-image>:<tag> --format '{{json .Config.Env}}' | jq
```
ベース側Dockerfileでは、最終的に `USER oracle` で `CMD exec $ORACLE_BASE/$RUN_FILE`（= runOracle.sh）となる構成が示されています。citeturn10view1  
Dockerの `USER` 命令は「ビルド中だけでなく実行時のユーザーにも影響する」点が公式に解説されています。citeturn26search2

### トラブルシューティング手順と期待される出力

| 手順 | コマンド例 | 期待される出力（正常の目安） | 異常なら何を示唆するか |
|---|---|---|---|
| 起動経路の確認 | `docker logs --tail 80 <container>` | `The following output is now a tail of the alert.log:` が出る（runOracleが最後にtailする設計）citeturn14view5turn10view1 | そもそもrunOracleが走っていない→CMD/ENTRYPOINT上書き、または実行権限喪失 |
| メモリ要件 | `docker inspect <container> --format '{{.HostConfig.Memory}}'` | 0（無制限）または十分な値。runOracleは2GB未満で終了。citeturn25view0 | 低すぎる→`--memory`/K8s limitsで即死 |
| 実行ユーザー | `docker inspect <container> --format '{{.Config.User}}'` | `oracle` など、ベースと同等 | 変化している→sqlplus OS認証/権限/書込が崩れる |
| oradata マウントと権限 | `docker inspect <container> --format '{{json .Mounts}}' | jq` | `/opt/oracle/oradata` が意図したvolume/bindでマウント | ホストパスが違う・read-only等 |
| コンテナ内UID/GID | `docker exec -it <container> bash -lc 'id; umask; ls -ld /opt/oracle /opt/oracle/oradata'` | `uid=54321(oracle)` で oradataが書込可 | uid不一致・oradataがroot所有→作成失敗 |
| healthcheck | `docker inspect <container> --format '{{.State.Health.Status}}'` | `healthy`（起動直後は `starting` もあり）citeturn10view1turn16view0 | `unhealthy` 継続→marker/SQL*Plus失敗（/dev/shm・権限・DB未起動） |

### Podman / containerd / Kubernetes の場合

#### Podman（rootless含む）
Oracle側FAQは、rootless Podmanでは `/etc/subuid` によるuid remapが絡むため `podman unshare chown 54321:54321` が必要になり得る、と具体例を示しています。citeturn13view0  
```bash
podman ps -a
podman logs --tail 300 <container>
podman inspect <container> > inspect.container.json

# rootless の uid remap を踏んでいそうなら（Oracle FAQの例）
podman unshare chown 54321:54321 <host-data-dir>
```

#### containerd / Kubernetes（crictl）
Kubernetes環境では、標準的に `kubectl logs` / `crictl logs` が最短です。citeturn18search18turn18search10  
```bash
# Kubernetes
kubectl get pods -A -o wide
kubectl logs <pod> -c <container> --tail=300
kubectl logs <pod> -c <container> --previous --tail=300

# containerd / CRI
crictl ps -a
crictl logs <container_id> | tail -n 300
```

#### ホスト側ジャーナル（systemd管理なら）
```bash
# Docker Engine
journalctl -u docker --since "2 hours ago" -xe

# containerd
journalctl -u containerd --since "2 hours ago" -xe

# kubelet
journalctl -u kubelet --since "2 hours ago" -xe
```

### SELinux / AppArmor の確認

#### SELinux（バインドマウント時）
Oracle FAQは、SELinux有効時の権限問題は `:Z` 付与が有効と述べています。citeturn13view0  
Dockerdocsも `z/Z` オプションがSELinuxラベル変更用であることを明記しています。citeturn17search1  
```bash
# ホスト
getenforce
ls -Z <host-data-dir>

# 例: Docker bind mount に :Z（プライベート）を付与
docker run ... -v <host-data-dir>:/opt/oracle/oradata:Z ...
```

#### AppArmor
Dockerはデフォルトで `docker-default` AppArmorプロファイルを適用し得ること、`--security-opt apparmor=...` で指定できることが公式に整理されています。citeturn17search0turn17search12  
```bash
# Ubuntu系で状態確認
aa-status

# 切り分け（必要な場合のみ）：AppArmorを外す
docker run --security-opt apparmor=unconfined ...
```

## 修正案と最小 Dockerfile パターン

### まず守るべき設計原則

派生イメージでやることが「ユーザー追加」「ディレクトリ追加」「初期SQL投入」程度なら、**ベースのCMD/healthcheck/userを極力触らず**、Oracleが用意している“拡張ポイント”（`/opt/oracle/scripts/setup`・`/opt/oracle/scripts/startup`）を使うのが最も事故が少ないです。citeturn5view1turn7view3turn10view1

特に `/opt/oracle/oradata` は “oracle(uid 54321) が書ける” を満たすため、ホスト側権限/SELinuxラベル/Podman rootless remapまで含めて管理する必要があります。citeturn13view0turn17search1turn5view1

### よくあるDockerfile事故と修正版の対比

| 事故パターン（例） | 修正版パターン（例） | 直る理由 |
|---|---|---|
| `USER root` のままDockerfileが終わる | `USER root` → 作業 → **最後に `USER oracle`** | runOracle/createDB/checkDBStatus が `oracle` 前提（OS認証・権限・uid 54321）citeturn10view1turn13view0turn16view0 |
| `CMD ["bash"]` 等で上書き | **CMD/ENTRYPOINTを定義しない**（ベースを継承） | ベースの `CMD exec runOracle.sh` を失うとDBが起動しないciteturn10view1turn26search0 |
| `chown -R <appuser> /opt/oracle` | **/opt/oracle 配下は基本触らない**。触るなら最小範囲で `oracle:dba` を維持 | スクリプトが `/opt/oracle/diag` や `ORACLE_HOME` 下へリンク/書込みを行うciteturn14view0turn14view2turn14view5 |
| uid 54321 を別ユーザーに割当 | **54321はoracle専用として避ける**（別UIDを使う） | oradataの所有・既存ファイル整合が崩れ、書込み不能になり得るciteturn13view0turn7view0 |
| `/docker-entrypoint-initdb.d` を消す/別物にする | **維持する**（setup/startupの互換リンク） | スクリプト配置の“規約”を壊すと初期化が実行されないciteturn7view3turn5view1 |

### 最小Dockerfile例

#### 例：ユーザー/ディレクトリ追加だけを安全に行い、DBの正規起動を壊さない

```Dockerfile
FROM container-registry.oracle.com/database/enterprise_ru:19.xx.0.0

# 追加作業は root で実施
USER root

# 既存の oracle(uid=54321) と衝突しないUID/GIDを使う
RUN groupadd -g 2000 appgrp \
 && useradd  -u 2000 -g 2000 -m -s /bin/bash appusr \
 && mkdir -p /opt/app \
 && chown -R 2000:2000 /opt/app

# Oracleが提供する拡張ポイントにスクリプトを配置（必要なら）
# .sql/.sh がサポートされる設計
COPY --chown=oracle:dba ./startup-scripts/ /opt/oracle/scripts/startup/
COPY --chown=oracle:dba ./setup-scripts/   /opt/oracle/scripts/setup/

# 最後に必ず oracle に戻す（ベースCMDをそのまま使う）
USER oracle
```

この構成は、ベースが期待する `USER oracle` と `CMD exec runOracle.sh` を保持しつつ、セットアップ/起動後処理を所定ディレクトリに載せるだけ、という“安全側”の拡張です。citeturn10view1turn7view3turn5view1

#### 例：起動時に必要な ulimit と oradata を満たす run コマンド（Docker）

oracle/docker-images READMEは、Enterprise/SE2実行時に ulimit の具体値例を提示しています。citeturn1view1  
```bash
docker run --name oracle-db \
  -p 1521:1521 -p 5500:5500 \
  --ulimit nofile=1024:65536 \
  --ulimit nproc=2047:16384 \
  --ulimit stack=10485760:33554432 \
  --ulimit memlock=3221225472 \
  -e ORACLE_SID=ORCLCDB \
  -e ORACLE_PDB=ORCLPDB1 \
  -e ORACLE_PWD='YourStrongPassword' \
  -v /srv/oradata:/opt/oracle/oradata:Z \
  <your-custom-image>:<tag>
```

`/opt/oracle/oradata` が oracle(uid 54321) で書ける必要がある点は、Oracle docs/FAQ双方に明記されています。citeturn5view1turn13view0  
SELinuxが有効なら `:Z`/`:z` が必要になり得る点はOracle FAQとDocker docsの双方で説明されています。citeturn13view0turn17search1

## ログの抽出と解釈ガイド

### まず見るべきログは「コンテナログ＝runOracleがtailしているalertログ」

`runOracle.sh` は末尾で `tail -f $ORACLE_BASE/diag/rdbms/*/*/trace/alert*.log` を行うため、**`docker logs` だけでalert.logの末尾が見える**のが基本設計です。citeturn14view5turn18search35

```bash
docker logs --tail 400 <container>
docker logs -f <container>
```

### alert.log の“実体”を直接読む

コンテナ内の想定パスは `runOracle.sh` が示しており、次の通りです。citeturn14view5  
```bash
docker exec -it <container> bash -lc '
  ls -l $ORACLE_BASE/diag/rdbms/*/*/trace/alert*.log
  tail -n 200 $ORACLE_BASE/diag/rdbms/*/*/trace/alert*.log
'
```

### DBCA（create database）失敗時は cfgtoollogs を見る

`createDB.sh` はDBCA失敗時に `/opt/oracle/cfgtoollogs/dbca/<SID>/<SID>.log` などを `cat` して標準出力に流す設計です（＝docker logsにも出ている可能性が高い）。citeturn21view1turn22view0  

直接抜く場合：
```bash
docker exec -it <container> bash -lc '
  ls -R /opt/oracle/cfgtoollogs/dbca 2>/dev/null | head
  find /opt/oracle/cfgtoollogs/dbca -maxdepth 3 -type f -name "*.log" -print
  tail -n 200 /opt/oracle/cfgtoollogs/dbca/*/*.log 2>/dev/null
'
```

### oradata ボリュームに“永続化されるログ”の確認

Oracleのインストールガイドは、データボリューム `/opt/oracle/oradata` に **data files/redo/audit/alert/trace** を格納すると明記しています。citeturn5view0turn6view1  
したがって、コンテナが落ちてもボリュームが残っていれば、ホスト側で追えます。

ホストに bind mountしているなら：
```bash
# ホスト側
sudo ls -la /srv/oradata
sudo find /srv/oradata -maxdepth 4 -type f \( -name "alert*.log" -o -name "*.trc" \) | head
sudo tail -n 200 /srv/oradata/**/alert*.log 2>/dev/null
```

### healthcheck が失敗しているときの見方

`checkDBStatus.sh` は、(1) `/dev/shm/.db_started`（marker）存在、(2) oraenvでSID設定、(3) SQL*PlusでDB roleとPDB openを確認、という流れです。citeturn16view0turn14view4  
したがって、`unhealthy` の場合は次を見ます。

```bash
# healthcheck の履歴（docker）
docker inspect <container> --format '{{json .State.Health}}' | jq

# marker と /dev/shm を確認
docker exec -it <container> bash -lc '
  ls -l /dev/shm
  mount | grep /dev/shm || true
  test -f /dev/shm/.db_started && echo "marker exists" || echo "marker missing"
'
```

`/dev/shm` のサイズや `exec` 権限が問題になるケース（特にXEやPL/SQL native compile等）はOracle FAQに具体策が記載されています。citeturn13view0

### コンテナログの“実体ファイル”を辿る

Docker環境で `LogPath` を引けることはDocker docs（日本語）に例示があります。citeturn18search1  
```bash
docker inspect --format='{{.LogPath}}' <container>
sudo tail -n 300 <LogPathで出たファイル>
```

Kubernetesではノード上のログ配置や収集の考え方が公式docsに整理されています。citeturn18search18turn18search2

## 参考情報

- Oracle Databaseコンテナの構成要素（環境変数、ボリューム、最小要件2GB RAM/21GB disk、スクリプト実行ポイント）についてのOracle公式ドキュメント。citeturn4view0turn5view1turn6view0turn6view1  
- oracle/docker-images（SingleInstance）のREADME/FAQ：uid=54321、/opt/oracle/oradataの権限、SELinux `:Z`、/dev/shm、起動後スクリプト配置、推奨ulimitなど。citeturn1view1turn13view0turn7view3turn16view0turn25view0  
- Docker公式ドキュメント：bind mountのSELinux `z/Z`、AppArmor、`docker logs`、`docker inspect` などの運用コマンド。citeturn17search1turn17search0turn18search35turn18search1turn26search0  
- 日本語の補助資料（運用メモとして有用）：Oracle Container Registry利用の導入例、ulimit不足での起動失敗と緩和例、Dockerログの保管場所の確認例。citeturn1view2turn2search3turn18search0