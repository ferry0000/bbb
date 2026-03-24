はい、**可能です**。
1つの ConfigMap に複数のシェルスクリプトを入れて、**それぞれ独立したファイルとして参照・実行できます。**

---

# ■ 仕組み

ConfigMap の `data` は「キー＝ファイル名」として扱われます。

```yaml
data:
  file1.sh: |
    ...
  file2.sh: |
    ...
```

👉 Pod にマウントすると：

```text
/opt/scripts/file1.sh
/opt/scripts/file2.sh
```

という **別ファイルとして存在**します。

---

# ■ 例（複数スクリプト）

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: multi-script
data:
  setup.sh: |
    #!/bin/sh
    echo "setup処理"

  install.sh: |
    #!/bin/sh
    echo "install処理"

  start.sh: |
    #!/bin/sh
    echo "start処理"
```

---

## Pod / StatefulSet 側

```yaml
volumeMounts:
  - name: scripts
    mountPath: /opt/scripts
    readOnly: true

volumes:
  - name: scripts
    configMap:
      name: multi-script
      defaultMode: 0755
```

---

# ■ 実行方法

## 方法1：commandで直接指定

```yaml
command: ["/bin/sh", "/opt/scripts/setup.sh"]
```

---

## 方法2：1つのスクリプトから他を呼ぶ（推奨）

```sh
#!/bin/sh
set -e

/opt/scripts/setup.sh
/opt/scripts/install.sh
exec /opt/scripts/start.sh
```

👉 一元制御できる

---

## 方法3：initContainerとmainで分ける

```yaml
initContainers:
  - command: ["/bin/sh", "/opt/scripts/setup.sh"]

containers:
  - command: ["/bin/sh", "/opt/scripts/start.sh"]
```

👉 フェーズ分離

---

# ■ 部分マウント（特定ファイルだけ使う）

```yaml
volumeMounts:
  - name: scripts
    mountPath: /opt/scripts/setup.sh
    subPath: setup.sh
```

👉 1ファイルだけマウントも可能

---

# ■ 注意点

## 1. 実行権限

```yaml
defaultMode: 0755
```

👉 必須（忘れると実行できない）

---

## 2. 改行コード

* LF にする（CRLFはNG）

---

## 3. サイズ制限

ConfigMapは約1MB制限

👉 大きなスクリプトは不向き

---

## 4. 順序制御

ConfigMap自体は順序を持たない

👉 実行順はスクリプト側で制御

---

# ■ あなたのケースでの使い方

今回の構成なら：

### ConfigMap

```text
bootstrap.sh
proxy.sh
install.sh
user.sh
oracle-start.sh
```

### bootstrap.sh

```sh
#!/bin/sh
set -e

/opt/bootstrap/proxy.sh
/opt/bootstrap/install.sh
/opt/bootstrap/user.sh

exec /opt/bootstrap/oracle-start.sh
```

👉 分割して管理可能

---

# ■ 結論

> 一つのConfigMapに複数のシェルスクリプトファイルを設定して、それぞれ別に参照できますか？

**できます。完全に問題ありません。**

---

# ■ 実務的ベスト

* 複数スクリプトをConfigMapに分割
* `bootstrap.sh`で順序制御
* 最後に `exec` で本体起動

---

必要なら次に：

* 「分割スクリプト構成の最適設計（責務分離）」
* 「initContainer + bootstrap の役割分離パターン」
* 「subPath vs ディレクトリマウントの使い分け」

まで整理できます。


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


特定ファイルだけマウントするには、**`subPath` を使います。**
これは Kubernetes で「Volumeの一部（単一ファイル）だけをコンテナ内の特定パスにマウントする」ための標準手段です。

---

# ■ 基本構文

```yaml
volumeMounts:
  - name: <volume名>
    mountPath: <コンテナ内パス>
    subPath: <ConfigMap内のキー名（＝ファイル名）>
```

---

# ■ 例（1ファイルだけマウント）

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: scripts
data:
  setup.sh: |
    #!/bin/sh
    echo "setup"
  install.sh: |
    #!/bin/sh
    echo "install"
```

---

## Pod / StatefulSet 側

```yaml
volumeMounts:
  - name: scripts
    mountPath: /opt/scripts/setup.sh
    subPath: setup.sh

volumes:
  - name: scripts
    configMap:
      name: scripts
      defaultMode: 0755
```

---

# ■ マウント結果

```text
/opt/scripts/setup.sh   ← 存在
/opt/scripts/install.sh ← 存在しない
```

👉 **setup.sh だけが見える**

---

# ■ 実行例

```yaml
command: ["/bin/sh", "/opt/scripts/setup.sh"]
```

---

# ■ 複数ファイルを個別にマウント

```yaml
volumeMounts:
  - name: scripts
    mountPath: /opt/scripts/setup.sh
    subPath: setup.sh

  - name: scripts
    mountPath: /opt/scripts/install.sh
    subPath: install.sh
```

👉 個別に配置可能

---

# ■ 重要な注意点

## 1. mountPathは「ファイルパス」

```yaml
mountPath: /opt/scripts/setup.sh
```

👉 ディレクトリではなく「ファイル」を指定

---

## 2. 既存ファイルがあると上書きされる

* mountPathに既存ファイルがある場合 → 覆い隠される

---

## 3. ディレクトリは自動作成されない場合あり

```yaml
/opt/scripts/setup.sh
```

👉 `/opt/scripts` が存在しないと失敗する場合あり

対策：

```yaml
mountPath: /tmp/setup.sh
```

または事前にディレクトリ作成

---

## 4. 更新が自動反映されない

重要：

> **subPathでマウントしたファイルは、ConfigMap更新が反映されない**

👉 Pod再起動が必要

---

## 5. 実行権限

```yaml
defaultMode: 0755
```

👉 必須（スクリプト実行のため）

---

# ■ ディレクトリマウントとの違い

| 方法         | 特徴              |
| ---------- | --------------- |
| ディレクトリマウント | 全ファイル見える・自動更新あり |
| subPath    | 単一ファイル・更新反映なし   |

---

# ■ あなたのケースでの使い分け

## ✔ subPathが向くケース

* 1つのスクリプトだけ使う
* ファイル名を固定したい
* 既存ディレクトリに差し込みたい

## ✔ ディレクトリマウントが向くケース

* 複数スクリプト
* bootstrap.shから他を呼ぶ
* 更新を反映したい

---

# ■ 結論

> 特定ファイルだけマウントする方法

👉 **`subPath` を使う**

```yaml
volumeMounts:
  - name: scripts
    mountPath: /opt/scripts/setup.sh
    subPath: setup.sh
```

---

# ■ 実務的おすすめ

あなたの構成なら：

* 複数スクリプト → ディレクトリマウント
* 単一エントリポイント → subPath

---

必要なら次に：

* subPathでハマる典型例（permission / mount失敗）
* Oracleコンテナでの安全な配置パターン
* ConfigMap更新時の自動ロールアウト方法

まで説明できます。


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


`subPath` は便利ですが、**挙動が「通常のVolumeマウントと違う」ためにハマりやすいポイントが多い**です。代表的なものを、原因→症状→対策で整理します。

---

# ■ 1. ConfigMap更新が反映されない（最頻出）

### 症状

* ConfigMapを更新してもスクリプトが変わらない
* Pod再起動しないと反映されない

### 原因

> `subPath` は「バインドマウント」になるため、**動的更新が効かない**

通常のConfigMapマウント：

```text
更新 → 自動反映（数秒後）
```

subPath：

```text
更新 → 反映されない
```

### 対策

```text
Pod再起動（必須）
```

または：

```yaml
annotations:
  checksum/config: <ハッシュ>
```

でローリング更新

---

# ■ 2. mountPathがディレクトリ扱いになる

### NG例

```yaml
mountPath: /opt/scripts
subPath: setup.sh
```

### 症状

* エラー
* またはファイルとしてマウントされない

### 原因

`subPath` は **ファイル単位マウント**
→ mountPathもファイルである必要あり

### 正解

```yaml
mountPath: /opt/scripts/setup.sh
subPath: setup.sh
```

---

# ■ 3. 親ディレクトリが存在しない

### 症状

* Pod起動失敗
* Mountエラー

### 原因

`subPath` は親ディレクトリを自動作成しないことがある

### 対策

事前にディレクトリを作る

```yaml
mountPath: /tmp/setup.sh
```

または：

* Dockerイメージ側で作成
* initContainerでmkdir

---

# ■ 4. 権限エラー（特にroot以外）

### 症状

* Permission denied
* スクリプト実行できない

### 原因

* defaultMode未設定
* runAsUserと不整合

### 対策

```yaml
configMap:
  defaultMode: 0755
```

または：

```yaml
securityContext:
  runAsUser: 0
```

---

# ■ 5. ファイルが上書きされて消える

### 症状

* 元々あったファイルが見えなくなる

### 原因

`subPath` はそのパスを完全に上書きする

```text
元: /opt/scripts/setup.sh
↓
subPathマウント
↓
元ファイルは隠れる
```

### 対策

* 別パスにマウントする
* 既存ファイルを使わない

---

# ■ 6. ファイルが存在しないと起動失敗

### 症状

* Podが起動しない
* Mount error

### 原因

`subPath` は対象ファイルが存在しないと失敗

### 対策

* ConfigMapキー名と一致させる

```yaml
subPath: setup.sh  ← dataのキーと一致
```

---

# ■ 7. ディレクトリに対してsubPath指定（誤用）

### NG例

```yaml
subPath: scripts/
```

### 症状

* マウント失敗

### 原因

`subPath` はファイル or ディレクトリ単体だが、
ConfigMapでは基本「キー＝ファイル」

---

# ■ 8. exec権限がない

### 症状

```text
permission denied
```

### 原因

```yaml
defaultMode: 0644
```

### 対策

```yaml
defaultMode: 0755
```

---

# ■ 9. 改行コード問題（地味に多い）

### 症状

```text
/bin/sh^M: not found
```

### 原因

CRLF

### 対策

LFにする

---

# ■ 10. volumeMountの競合

### NG例

```yaml
volumeMounts:
  - mountPath: /opt/scripts
  - mountPath: /opt/scripts/setup.sh
```

### 症状

* どちらかが無効になる
* 不安定

---

# ■ まとめ（重要な3つ）

特に重要なのはこれ：

### ① 更新が反映されない

→ **必ずPod再起動**

### ② mountPathはファイル指定

→ `/path/file.sh`

### ③ 親ディレクトリ必要

→ 事前に存在させる

---

# ■ あなたのケースでの注意

Oracle + bootstrap構成なら：

### OK

```yaml
mountPath: /opt/bootstrap/bootstrap.sh
subPath: bootstrap.sh
```

### NG

```yaml
mountPath: /opt/bootstrap
subPath: bootstrap.sh
```

---

# ■ 実務的判断

| 用途      | 推奨         |
| ------- | ---------- |
| 単一スクリプト | subPath    |
| 複数スクリプト | ディレクトリマウント |

---

# ■ 結論

> subPathでハマる典型例

最重要は：

* **更新されない**
* **パス指定ミス（ディレクトリ vs ファイル）**
* **ディレクトリ未作成**

---

必要なら次に：

* 「OracleDB + subPath の安全構成」
* 「subPathを使わない設計（より安定）」
* 「ConfigMap更新時の自動再起動設計」

まで踏み込めます。


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


この部分は少し誤解しやすいので、正確に分解して説明します。

---

# ■ 結論（先に）

> `mountPath: /tmp/setup.sh` は「ディレクトリ作成の対策」ではなく、
> **「既に存在するディレクトリ配下にマウントすることで失敗を避ける」ための回避策**です。

---

# ■ 何が問題なのか（本質）

`subPath` の制約：

> **mountPath の「親ディレクトリ」は事前に存在していないといけない**

---

## NG例

```yaml
mountPath: /opt/bootstrap/setup.sh
subPath: setup.sh
```

もしコンテナ内に：

```text
/opt/bootstrap  ← 存在しない
```

なら：

👉 **マウント失敗（Pod起動エラー）**

---

# ■ 対策①（本質的な正解）

### 親ディレクトリを事前に作る

#### 方法A：Dockerイメージ側

```dockerfile
RUN mkdir -p /opt/bootstrap
```

#### 方法B：initContainer

```yaml
initContainers:
  - name: init
    image: busybox
    command: ["sh", "-c", "mkdir -p /opt/bootstrap"]
    volumeMounts:
      - name: bootstrap-dir
        mountPath: /opt/bootstrap
```

---

# ■ 対策②（簡易回避）

### `/tmp` を使う

```yaml
mountPath: /tmp/setup.sh
subPath: setup.sh
```

---

## なぜこれで動くのか？

```text
/tmp ← ほぼ確実に存在する
```

👉 だから失敗しない

---

# ■ つまりこの2つの違い

| 方法                        | 意味            |
| ------------------------- | ------------- |
| `/opt/bootstrap/setup.sh` | ディレクトリ事前作成が必要 |
| `/tmp/setup.sh`           | 既存ディレクトリなので安全 |

---

# ■ 重要な誤解ポイント

あなたの引用：

> 対策 事前にディレクトリを作る
> mountPath: /tmp/setup.sh

これは厳密には：

```text
❌ 「/tmp を使うとディレクトリが作られる」
```

ではなく

```text
⭕ 「/tmp は既にあるから失敗しない」
```

です。

---

# ■ 実務的な正解

## ✔ 正しい設計

```yaml
mountPath: /opt/bootstrap/setup.sh
```

＋

```text
/opt/bootstrap を事前に作る
```

---

## ✔ 応急処置

```yaml
mountPath: /tmp/setup.sh
```

---

# ■ あなたのケース（Oracle）

### 推奨

```yaml
mountPath: /opt/bootstrap/bootstrap.sh
```

ただし：

```text
/opt/bootstrap は必ず存在させる
```

---

# ■ まとめ

> mountPath: /tmp/setup.sh の意味

👉

* `/tmp` は既に存在する
* だから subPath マウントが失敗しない
* **ディレクトリ作成の代わりではなく、回避策**

---

# ■ 最短理解

```text
subPath = 親ディレクトリ必須
↓
無いと失敗
↓
/tmpならあるのでOK
```

---

必要なら次に：

* 「Oracleコンテナ内で確実に存在するディレクトリ一覧」
* 「subPathを安全に使う設計テンプレ」
* 「initContainerでディレクトリ作る完全例」

まで出せます。


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


結論から言うと：

> **subPathを使わない設計＝「ディレクトリ単位でマウントし、ファイル選択はコンテナ側で行う設計」です。**
> これが Kubernetes 的に最も安定します。

---

# ■ なぜ subPath を避けるのか（本質）

subPath は以下の制約があります：

* ConfigMap更新が反映されない（Pod再起動必須）
* 親ディレクトリ依存
* mount失敗しやすい
* デバッグしにくい

👉 **「静的で壊れやすいマウント」**

---

# ■ subPathを使わない設計（基本形）

## ✔ ディレクトリ丸ごとマウント

```yaml
volumeMounts:
  - name: scripts
    mountPath: /opt/bootstrap
    readOnly: true
```

```yaml
volumes:
  - name: scripts
    configMap:
      name: oracle-bootstrap
      defaultMode: 0755
```

---

## ✔ コンテナ側でファイル指定

```yaml
command: ["/bin/sh", "/opt/bootstrap/bootstrap.sh"]
```

---

# ■ これで何が起きるか

ConfigMap:

```yaml
data:
  bootstrap.sh: ...
  install.sh: ...
  proxy.sh: ...
```

コンテナ内：

```text
/opt/bootstrap/bootstrap.sh
/opt/bootstrap/install.sh
/opt/bootstrap/proxy.sh
```

👉 **全部見える**

---

# ■ 実行制御はスクリプトでやる

```sh
#!/bin/sh
set -e

/opt/bootstrap/proxy.sh
/opt/bootstrap/install.sh

exec /opt/bootstrap/start-oracle.sh
```

👉 subPath不要

---

# ■ subPath設計との違い

## ❌ subPath設計

```yaml
volumeMounts:
  - mountPath: /opt/bootstrap/bootstrap.sh
    subPath: bootstrap.sh
```

問題：

* 更新されない
* 親ディレクトリ問題
* 柔軟性なし

---

## ✔ 非subPath設計

```yaml
mountPath: /opt/bootstrap
```

メリット：

* 自動更新される
* 複数ファイル扱える
* トラブル少ない

---

# ■ 安定性の比較

| 項目     | subPath | ディレクトリ |
| ------ | ------- | ------ |
| 更新反映   | ❌       | ✔      |
| マウント失敗 | 多い      | 少ない    |
| 柔軟性    | 低い      | 高い     |
| デバッグ   | 難しい     | 簡単     |

---

# ■ あなたのケースでの最適構成

### ConfigMap（複数スクリプト）

```yaml
data:
  bootstrap.sh: ...
  proxy.sh: ...
  install.sh: ...
  user.sh: ...
  start.sh: ...
```

---

### Pod

```yaml
volumeMounts:
  - name: bootstrap
    mountPath: /opt/bootstrap
    readOnly: true
```

---

### command

```yaml
command: ["/bin/sh", "/opt/bootstrap/bootstrap.sh"]
```

---

### bootstrap.sh

```sh
#!/bin/sh
set -e

/opt/bootstrap/proxy.sh
/opt/bootstrap/install.sh
/opt/bootstrap/user.sh

exec /opt/bootstrap/start.sh
```

---

# ■ なぜこれが「より安定」なのか

理由は3つです：

### ① Kubernetesの設計に合っている

ConfigMapは「ディレクトリマウント前提」

---

### ② 動的更新が効く

```text
ConfigMap更新 → 自動反映
```

---

### ③ シンプル

```text
Volume = ディレクトリ
スクリプト = ファイル
```

👉 一貫している

---

# ■ subPathを使うべき場面

例外的にOK：

* 既存ディレクトリに1ファイルだけ差し込みたい
* ファイルパスを固定しないといけない
* legacy対応

---

# ■ 結論

> 「subPathを使わない設計（より安定）」とは何ですか？

👉

> **ConfigMapをディレクトリごとマウントし、実行ファイルはコンテナ側で選択する設計**

---

# ■ 最短まとめ

```text
subPath = 単一ファイルマウント（壊れやすい）
↓
ディレクトリマウント = 安定
```

---

必要なら次に：

* 「OracleDB + ConfigMap分割スクリプトの最適構成」
* 「ConfigMap更新時の自動ロールアウト設計」
* 「initContainerとの役割分離」

まで踏み込んで整理できます。


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


結論から説明します。

> **「ConfigMap更新 → 自動反映」とは、
> ConfigMapを変更すると、Pod内にマウントされているファイル内容が“再起動なしで書き変わる”挙動のことです。**

ただし、**条件付きです（ここが重要）**。

---

# ■ 何が起きるのか（動作イメージ）

## ConfigMap（例）

```yaml
data:
  message.txt: "hello"
```

Pod内：

```text
/opt/config/message.txt → hello
```

---

## ConfigMapを更新

```yaml
data:
  message.txt: "hello world"
```

---

## すると（数秒後）

```text
/opt/config/message.txt → hello world
```

👉 **Pod再起動なしで中身が変わる**

---

# ■ 仕組み（内部）

Kubernetesは：

* ConfigMapを **tmpfs（メモリ上）に展開**
* 定期的に同期（数秒〜数十秒）

👉 ファイルが差し替わる

---

# ■ ただし重要な条件

## ✔ 条件1：ディレクトリマウントであること

```yaml
mountPath: /opt/config
```

👉 OK（自動反映）

---

## ❌ 条件2：subPathはダメ

```yaml
mountPath: /opt/config/file.txt
subPath: file.txt
```

👉 **更新されない**

理由：

```text
subPath = bind mount
→ 固定ファイル
→ 更新されない
```

---

# ■ 実際のタイミング

即時ではありません：

```text
数秒〜1分程度の遅延あり
```

---

# ■ 注意点（重要）

## 1. アプリは自動で再読み込みしない

ファイルは変わるが：

```text
アプリがその変更を使うとは限らない
```

例：

* Java → 再起動しないと反映しない
* nginx → reload必要

---

## 2. inotify対応が必要な場合あり

変更を検知するには：

* アプリ側で監視
* または手動reload

---

## 3. ファイルは「差し替え」される

inodeが変わる場合あり：

```text
cat → OK
tail -f → NGになることあり
```

---

# ■ Oracle + あなたのケース

今回の構成：

```yaml
mountPath: /opt/bootstrap
```

👉 この場合：

```text
ConfigMap更新 → スクリプト更新される
```

---

# ■ ただし重要な現実

あなたのケースでは：

```text
bootstrap.sh は起動時に1回しか実行されない
```

👉 つまり：

```text
更新されても意味がない
```

---

# ■ 実務での意味

| 用途         | 自動反映の価値 |
| ---------- | ------- |
| 設定ファイル     | 高い      |
| スクリプト（起動用） | 低い      |
| 実行中アプリ設定   | 条件付き    |

---

# ■ subPathとの対比

| 項目   | ディレクトリ | subPath |
| ---- | ------ | ------- |
| 更新反映 | ✔      | ❌       |
| 柔軟性  | 高い     | 低い      |
| 安定性  | 高い     | 低い      |

---

# ■ 結論

> ConfigMap更新 → 自動反映とは何か？

👉

> **ConfigMapを変更すると、Pod内のマウントファイルが自動的に更新される仕組み（再起動不要）**

ただし：

* ディレクトリマウントのみ
* subPathは対象外
* アプリが再読込しないと意味がない

---

# ■ あなた向けの最短理解

```text
ConfigMap更新
↓
/opt/bootstrap/*.sh が書き変わる
↓
ただし bootstrap.sh は再実行されない
```

---

必要なら次に：

* 「ConfigMap変更時にPodを自動再起動する方法」
* 「checksum annotation パターン」
* 「Oracle構成での安全な更新戦略」

まで踏み込めます。
