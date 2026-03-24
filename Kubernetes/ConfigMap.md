結論から言うと：

> **ConfigMapでスクリプトを実行することと、PVCの要否は無関係です。**
> **OracleDBを使うなら、PVCは基本的に「必要」です。**

あなたの提示している構成は「起動方法」の話であって、「データ永続化」とは別レイヤです。

---

# ■ なぜPVCが必要か（OracleDBの場合）

OracleDBは以下にデータを書きます：

* `/opt/oracle/oradata`（データファイル）
* redoログ
* control file
* Data Pump など

これらはコンテナの **root filesystem に書かれる**ため、

👉 PVCがない場合：

```text
Pod削除 → データ消滅
```

---

# ■ ConfigMapの役割

あなたの構成：

```yaml
volumeMounts:
  - name: bootstrap
    mountPath: /opt/bootstrap
```

これは：

* スクリプト配布
* 設定ファイル配布

👉 **読み取り専用の一時ファイル用途**

---

# ■ PVCの役割

```yaml
volumeMounts:
  - name: db-storage
    mountPath: /opt/oracle/oradata
```

これは：

* DBデータの永続化

👉 **必須（ほぼ100%）**

---

# ■ 組み合わせた正しい構成

```yaml
containers:
  - name: oracle
    image: container-registry.oracle.com/database/enterprise:latest
    securityContext:
      runAsUser: 0
    command: ["/bin/sh", "/opt/bootstrap/bootstrap.sh"]
    volumeMounts:
      - name: bootstrap
        mountPath: /opt/bootstrap
        readOnly: true
      - name: db-storage
        mountPath: /opt/oracle/oradata

volumes:
  - name: bootstrap
    configMap:
      name: oracle-bootstrap
      defaultMode: 0755
  - name: db-storage
    persistentVolumeClaim:
      claimName: oracle-pvc
```

---

# ■ PVCが不要なケース（例外）

以下の場合のみ不要です：

### 1. 一時的な検証

```text
テスト用
→ Pod消えてもOK
```

### 2. DBを使わない（今回対象外）

---

# ■ StatefulSetの場合（推奨）

OracleDBは通常 StatefulSet を使います：

```yaml
volumeClaimTemplates:
  - metadata:
      name: db-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

---

# ■ よくある誤解

### 誤解

> ConfigMap使ってるからVolumeは足りてる

👉 **誤り**

| 種類        | 用途       |
| --------- | -------- |
| ConfigMap | 設定・スクリプト |
| PVC       | データ      |

---

# ■ あなたのケースの整理

あなたの構成は：

* proxy設定
* dnf install
* user作成
* Oracle起動

👉 これは「起動ロジック」

しかし：

👉 OracleDBは「状態を持つDB」

---

# ■ 結論

> この構成ならPVCは不要？

**いいえ、OracleDBを使うならPVCは必要です。**

---

# ■ 最短回答

* ConfigMap → スクリプト用 → OK
* PVC → DBデータ用 → 必須

---

必要なら次に：

* OracleDB + StatefulSet + PVC + bootstrap の完全構成
* `/opt/oracle/oradata` 以外にマウントすべきディレクトリ
* Data Pump 用ディレクトリ設計

まで具体的に出せます。


<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>


はい。`persistentVolumeClaim` ではなく、**StatefulSet + `volumeClaimTemplates`** の形にするとこうなります。

以下は、あなたのこれまでの要件を入れた例です。

* `ConfigMap` で `bootstrap.sh` を配布
* `command` で `bootstrap.sh` を実行
* `bootstrap.sh` の最後で `oracle` ユーザとして OracleDB を起動
* DBデータは `volumeClaimTemplates` で永続化

---

## 例: StatefulSet

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: oracle-bootstrap
data:
  bootstrap.sh: |
    #!/bin/sh
    set -e

    # =========================
    # proxy設定（環境変数から取得）
    # =========================
    PROXY_URL="${HTTP_PROXY:-${http_proxy:-}}"

    if [ -z "$PROXY_URL" ]; then
      echo "ERROR: HTTP_PROXY / http_proxy が未設定です"
      exit 1
    fi

    DNF_CONF="/etc/dnf/dnf.conf"

    if [ ! -f "$DNF_CONF" ]; then
      mkdir -p /etc/dnf
      touch "$DNF_CONF"
    fi

    sed -i '/^proxy=/d' "$DNF_CONF"

    if grep -q "^\[main\]" "$DNF_CONF"; then
      sed -i "/^\[main\]/a proxy=${PROXY_URL}" "$DNF_CONF"
    else
      echo "[main]" >> "$DNF_CONF"
      echo "proxy=${PROXY_URL}" >> "$DNF_CONF"
    fi

    export HTTP_PROXY="$PROXY_URL"
    export HTTPS_PROXY="${HTTPS_PROXY:-$PROXY_URL}"
    export http_proxy="$PROXY_URL"
    export https_proxy="${https_proxy:-${HTTPS_PROXY:-$PROXY_URL}}"
    export NO_PROXY="${NO_PROXY:-localhost,127.0.0.1,.svc,.cluster.local}"
    export no_proxy="$NO_PROXY"

    # =========================
    # パッケージインストール
    # =========================
    if command -v microdnf >/dev/null 2>&1; then
      microdnf install -y shadow-utils
    elif command -v dnf >/dev/null 2>&1; then
      dnf install -y shadow-utils
    elif command -v yum >/dev/null 2>&1; then
      yum install -y shadow-utils
    else
      echo "ERROR: package manager が見つかりません"
      exit 1
    fi

    # =========================
    # OSグループ・ユーザ作成
    # =========================
    getent group appgroup >/dev/null 2>&1 || groupadd -g 1001 appgroup
    id appuser >/dev/null 2>&1 || useradd -u 1001 -g appgroup appuser

    # =========================
    # 権限調整
    # =========================
    mkdir -p /opt/oracle/oradata
    chown -R oracle:oinstall /opt/oracle/oradata

    # =========================
    # OracleDB起動
    # =========================
    if command -v runuser >/dev/null 2>&1; then
      exec runuser -u oracle -- /opt/oracle/runOracle.sh
    elif command -v su >/dev/null 2>&1; then
      exec su - oracle -c "/opt/oracle/runOracle.sh"
    else
      echo "ERROR: runuser / su が存在しません"
      exit 1
    fi
---
apiVersion: v1
kind: Service
metadata:
  name: oracle-db
spec:
  clusterIP: None
  selector:
    app: oracle-db
  ports:
    - name: listener
      port: 1521
      targetPort: 1521
    - name: emexpress
      port: 5500
      targetPort: 5500
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: oracle-db
spec:
  serviceName: oracle-db
  replicas: 1
  selector:
    matchLabels:
      app: oracle-db
  template:
    metadata:
      labels:
        app: oracle-db
    spec:
      containers:
        - name: oracle
          image: container-registry.oracle.com/database/enterprise:latest
          securityContext:
            runAsUser: 0
          env:
            - name: HTTP_PROXY
              value: "http://proxy.example.com:8080"
            - name: HTTPS_PROXY
              value: "http://proxy.example.com:8080"
            - name: NO_PROXY
              value: "localhost,127.0.0.1,.svc,.cluster.local"
          command: ["/bin/sh", "/opt/bootstrap/bootstrap.sh"]
          ports:
            - containerPort: 1521
              name: listener
            - containerPort: 5500
              name: emexpress
          volumeMounts:
            - name: bootstrap
              mountPath: /opt/bootstrap
              readOnly: true
            - name: db-storage
              mountPath: /opt/oracle/oradata
      volumes:
        - name: bootstrap
          configMap:
            name: oracle-bootstrap
            defaultMode: 0755
  volumeClaimTemplates:
    - metadata:
        name: db-storage
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        # 必要なら指定
        # storageClassName: standard
```

---

## 重要ポイント

### 1. `volumeClaimTemplates` を使う場合

`volumes:` には `db-storage` を書きません。
代わりに、`volumeClaimTemplates` に書きます。

つまりこの対応です。

* `volumeMounts.name: db-storage`
* `volumeClaimTemplates.metadata.name: db-storage`

この名前が一致していれば、StatefulSet が Pod ごとに PVC を自動作成します。

---

### 2. 作成されるPVC名

1レプリカなら、通常こうなります。

```text
db-storage-oracle-db-0
```

規則はだいたい次です。

```text
<volumeClaimTemplatesのname>-<StatefulSet名>-<ordinal>
```

---

### 3. `bootstrap` は ConfigMap のままでよい

`bootstrap.sh` を置くために PVC は不要です。
これはあくまで設定ファイル配布なので `ConfigMap` で十分です。

* `bootstrap` → ConfigMap
* `db-storage` → PVC (`volumeClaimTemplates`)

という役割分担です。

---

### 4. `runAsUser: 0`

今回の要件では、

* `dnf` / `yum` / `microdnf` 実行
* `/etc/dnf/dnf.conf` 変更
* `groupadd` / `useradd`

が必要なので、最初は root で動かす必要があります。

その後、`bootstrap.sh` の最後で `oracle` ユーザに切り替えて `runOracle.sh` を実行しています。

---

## 補足

OracleDB で Data Pump 用ディレクトリも永続化したいなら、`/opt/oracle/dpump` も別に考える必要があります。
ただし `volumeClaimTemplates` 1本で `/opt/oracle/oradata` と `/opt/oracle/dpump` の両方を使いたいなら、`subPath` を使うか、1つ上位のディレクトリにまとめてマウントする設計にします。

---

## かなり重要な注意

この構成は要件を満たしますが、運用上は弱点があります。

* Pod起動ごとにパッケージインストールが走る
* プロキシ障害で起動失敗する
* 起動が遅くなる
* 外部リポジトリ状態に依存する

そのため、本来はカスタムイメージの方が安定です。
ただ、**カスタムイメージ不可**という前提なら、この形がかなり現実的です。

必要なら次に、`/opt/oracle/dpump` も含めた `subPath` 版の StatefulSet をそのまま出します。


<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>



結論：

> **はい、initContainerでもConfigMapのスクリプトは使えます。**
> しかも、**非常に一般的で推奨されるパターン**です。

---

# ■ 仕組み

やることは main container と同じです。

1. ConfigMap にスクリプトを入れる
2. initContainer に Volume としてマウント
3. `command` で実行

---

# ■ 例（そのまま使える形）

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: init-script
data:
  init.sh: |
    #!/bin/sh
    set -e

    echo "initContainer start"

    # ディレクトリ作成
    mkdir -p /work/data

    # 権限調整
    chown -R 1001:1001 /work/data

    echo "initContainer done"
```

---

## Pod / StatefulSet 側

```yaml
initContainers:
  - name: init
    image: oraclelinux:9
    command: ["/bin/sh", "/opt/init/init.sh"]
    securityContext:
      runAsUser: 0
    volumeMounts:
      - name: init-script
        mountPath: /opt/init
        readOnly: true
      - name: db-storage
        mountPath: /work

containers:
  - name: oracle
    image: container-registry.oracle.com/database/enterprise:latest
    volumeMounts:
      - name: db-storage
        mountPath: /opt/oracle/oradata

volumes:
  - name: init-script
    configMap:
      name: init-script
      defaultMode: 0755
```

---

# ■ 重要ポイント

## 1. main container と完全に同じ使い方でOK

```yaml
command: ["/bin/sh", "/opt/init/init.sh"]
```

👉 問題なく実行される

---

## 2. initContainerは「必ず先に終わる」

これは postStart との決定的な違いです。

```text
initContainer → 完了
↓
main container 起動
```

👉 順序保証あり（非常に重要）

---

## 3. 共有Volumeでデータを渡す

initContainerとmain containerは：

* filesystemは別
* volumeだけ共有

```text
initContainer → /work に書く
main container → /opt/oracle/oradata で読む
```

---

## 4. root作業を分離できる

```yaml
initContainer:
  runAsUser: 0

main:
  runAsUser: 1001
```

👉 セキュリティ的に良い設計

---

# ■ あなたのケースへの適用

あなたの要件：

* proxy設定
* パッケージインストール
* ユーザ作成
* Oracle起動

これを分解すると：

---

## initContainerでやるべきこと

* chown / chmod
* ディレクトリ作成
* データ配置

👉 OK

---

## initContainerでやるべきでないこと

* dnf install（main containerに反映されない）
* useradd（main containerに反映されない）

👉 ここが重要

---

# ■ 重要な制約（再確認）

initContainerは：

```text
別コンテナ
```

なので：

| 処理              | mainに影響 |
| --------------- | ------- |
| ファイル作成（volume内） | 〇       |
| パッケージインストール     | ✕       |
| /etc/passwd変更   | ✕       |

---

# ■ 正しい使い分け

| 方法                  | 用途            |
| ------------------- | ------------- |
| initContainer       | 前処理（volume操作） |
| ConfigMap + command | 本体起動前処理       |
| postStart           | 軽い後処理         |

---

# ■ 結論

> initContainerでのコマンドもConfigMapを使えますか？

**使えます。完全に問題ありません。**

ただし：

> **initContainerでやった「OS変更（パッケージ・ユーザ）」はmainには引き継がれない**

---

# ■ あなたの最適構成（整理）

今回の要件なら：

### 必須

* ConfigMap + command（bootstrap.sh）

  * proxy
  * dnf install
  * useradd
  * Oracle起動

### 補助

* initContainer（ConfigMap）

  * chown
  * ディレクトリ準備

---

必要なら次に：

* 「initContainer + bootstrap併用の完全StatefulSet」
* Oracle用に最適化したディレクトリ設計
* fsGroupとの組み合わせ

まで具体化できます。
