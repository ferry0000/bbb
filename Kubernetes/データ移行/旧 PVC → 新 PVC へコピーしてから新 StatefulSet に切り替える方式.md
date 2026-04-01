了解です。
あなたの前提だと、**PV 再バインドではなく、旧 PVC → 新 PVC へコピーしてから新 StatefulSet に切り替える方式**が安全です。

理由は明確で、StatefulSet は `volumeClaimTemplates` から Pod ごとに安定した PVC を使いますが、**PVC の名前は StatefulSet 名に依存**します。いっぽう、local-path-provisioner は **削除時に teardown スクリプトで実体ディレクトリを掃除する仕組み**なので、再バインド後に「旧名前のディレクトリが現役データ」という状態を人手運用で安全に保つのは難しいです。([kubernetes.io の StatefulSet 説明](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) に相当する情報)では StatefulSet が stable storage を前提にすること、local-path-provisioner の README では `setup` / `teardown` スクリプトで作成・削除が行われることが示されています。([Kubernetes][1])

以下、**1レプリカの Oracle StatefulSet** を前提に、**停止あり・容量増加なし**での具体手順を書きます。
複数レプリカでも考え方は同じで、`-0`, `-1` を ordinal ごとに繰り返します。

---

## 0. 前提例

ここでは次を仮定します。

* namespace: `db`
* 旧 StatefulSet 名: `oracle-old`
* 新 StatefulSet 名: `oracle-new`
* claimTemplate 名: `oradata`
* mountPath: `/opt/oracle/oradata`
* replicas: `1`
* StorageClass: `local-path`

旧 PVC 名は通常:

```text
oradata-oracle-old-0
```

新 StatefulSet が使う PVC 名は通常:

```text
oradata-oracle-new-0
```

になります。StatefulSet は `volumeClaimTemplates` から Pod ごとに安定したストレージを扱います。([Kubernetes][1])

---

## 1. 事前バックアップ

Oracle なので、**コピー移行の前にバックアップ**を取ってください。
Kubernetes のストレージ移行手順としてはコピーで足りますが、DB としては別です。これは運用上の必須事項です。

---

## 2. 現状確認

まず現状を保存します。

```bash
NS=db
OLD_STS=oracle-old
NEW_STS=oracle-new
CLAIM=oradata

kubectl -n $NS get sts $OLD_STS -o yaml > old-sts.yaml
kubectl -n $NS get pvc -o wide
kubectl get pv
kubectl -n $NS get pod -o wide
```

旧 PVC / PV の対応も確認します。

```bash
kubectl -n $NS get pvc ${CLAIM}-${OLD_STS}-0 -o wide
kubectl get pv
```

PVC は Pod から参照する namespaced resource、PV は cluster-wide resource です。([Kubernetes][2])

---

## 3. 新 StatefulSet 用の PVC を先に作る

ここが重要です。
**新 StatefulSet が期待する名前の PVC を先に作成**します。コピー先です。

`new-pvc-0.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: oradata-oracle-new-0
  namespace: db
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 100Gi
```

適用:

```bash
kubectl apply -f new-pvc-0.yaml
kubectl -n $NS get pvc oradata-oracle-new-0
```

ここでは **旧 PVC と同容量**にしてください。容量不足は bind 失敗要因になります。PV/PVC は要求サイズ・access mode・StorageClass などの条件に基づいて bind されます。([Kubernetes][2])

---

## 4. コピー用 Pod を作る

1つの Pod に

* 旧 PVC
* 新 PVC

を両方マウントします。
Kubernetes では Pod が PVC を volume として利用できます。([Kubernetes][3])

`copy-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oracle-data-copy-0
  namespace: db
spec:
  restartPolicy: Never
  containers:
    - name: copy
      image: oraclelinux:8
      command: ["/bin/bash", "-lc", "sleep infinity"]
      volumeMounts:
        - name: old-data
          mountPath: /src
          readOnly: true
        - name: new-data
          mountPath: /dst
  volumes:
    - name: old-data
      persistentVolumeClaim:
        claimName: oradata-oracle-old-0
    - name: new-data
      persistentVolumeClaim:
        claimName: oradata-oracle-new-0
```

適用:

```bash
kubectl apply -f copy-pod.yaml
kubectl -n $NS get pod oracle-data-copy-0 -w
```

**まだコピーは実行しません。**
この段階では「両方マウントできること」の確認だけです。

---

## 5. 旧 StatefulSet を停止する

Oracle データを OS レベルでコピーするなら、**DB 停止後**に行ってください。
StatefulSet は scale down できます。([Kubernetes][1])

```bash
kubectl -n $NS scale sts $OLD_STS --replicas=0
kubectl -n $NS rollout status sts/$OLD_STS
kubectl -n $NS get pod
```

旧 Oracle Pod が完全停止したことを確認してください。

---

## 6. コピーを実行する

コピー用 Pod に入ります。

```bash
kubectl -n $NS exec -it oracle-data-copy-0 -- bash
```

コピーコマンドは `cp -a` でも動きますが、検証しやすいので `rsync` があればその方が良いです。
`oraclelinux:8` に `rsync` が無ければ、`cp -a` で進めます。

### まず中身確認

```bash
ls -la /src
ls -la /dst
du -sh /src /dst
```

### コピー

```bash
cp -a /src/. /dst/
sync
```

### コピー後確認

```bash
du -sh /src /dst
find /src | wc -l
find /dst | wc -l
```

必要なら一部の代表ファイルで確認します。

```bash
ls -l /src
ls -l /dst
```

`cp -a` は所有者・権限・タイムスタンプなどをできるだけ保持してコピーします。
ただし、Oracle の実行ユーザ UID/GID と新ボリューム側の見え方は後で必ず確認してください。

---

## 7. コピー用 Pod を削除する

コピーが終わったら削除します。

```bash
kubectl -n $NS delete pod oracle-data-copy-0
```

---

## 8. 新 StatefulSet YAML を作る

ポイントは次の2点です。

1. `metadata.name` を新名にする
2. `volumeClaimTemplates[].metadata.name` は **旧と同じ `oradata` のまま**にする

こうすると、新 StatefulSet の `-0` Pod が使う PVC 名は `oradata-oracle-new-0` になり、先に作った PVC と一致します。StatefulSet は Pod ごとに安定した storage identity を持ちます。([Kubernetes][1])

`new-sts.yaml` 例:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: oracle-new
  namespace: db
spec:
  serviceName: oracle-headless
  replicas: 1
  selector:
    matchLabels:
      app: oracle
  template:
    metadata:
      labels:
        app: oracle
    spec:
      containers:
        - name: oracle
          image: my-oracle-image:tag
          ports:
            - containerPort: 1521
            - containerPort: 5500
          env:
            - name: ORACLE_PWD
              value: "Oracle123"
          volumeMounts:
            - name: oradata
              mountPath: /opt/oracle/oradata
  volumeClaimTemplates:
    - metadata:
        name: oradata
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: local-path
        resources:
          requests:
            storage: 100Gi
```

### 補足

`volumeClaimTemplates` を残しているのは、今後も StatefulSet 標準の stable storage 管理に寄せるためです。既に同名 PVC がある場合、その PVC が使われる形になります。([Kubernetes][1])

---

## 9. 新 StatefulSet を作成する

```bash
kubectl apply -f new-sts.yaml
kubectl -n $NS rollout status sts/$NEW_STS
kubectl -n $NS get pod -o wide
kubectl -n $NS get pvc
kubectl get pv
```

---

## 10. Oracle 起動確認

ここは Kubernetes ではなく Oracle 側の確認です。

```bash
kubectl -n $NS logs oracle-new-0
kubectl -n $NS exec -it oracle-new-0 -- bash
```

最低限、次を確認してください。

* DB が既存データで起動しているか
* CDB / PDB が見えるか
* listener / service registration が正常か
* アプリ接続先変更が必要ないか

Pod identity は StatefulSet 名変更で変わるため、**Pod 名やホスト名依存の設定**があるなら要確認です。StatefulSet は stable network identity も前提にしています。([Kubernetes][1])

---

## 11. 問題なければ旧 StatefulSet と旧 PVC を削除する

新 StatefulSet が問題なく動いたのを確認してから、旧側を片付けます。

### 旧 StatefulSet が残っていれば削除

```bash
kubectl -n $NS delete sts $OLD_STS
```

### 旧 PVC 削除

```bash
kubectl -n $NS delete pvc oradata-oracle-old-0
```

local-path-provisioner はボリューム削除後に `teardown` スクリプトでディレクトリ cleanup を行う仕組みです。つまり、**手で storage ディレクトリを消すのではなく、PVC/PV のライフサイクルで削除する**のが本来の流れです。([GitHub][4])

---

# 重要な運用ルール

今回の方式では、**新 StatefulSet が稼働中に old 側の storage ディレクトリを OS から手で消してはいけません**。
削除判断は、必ず

* `kubectl get pvc`
* `kubectl get pv`
* `kubectl get sts`
* `kubectl get pod`

で確認してからにしてください。PV/PVC のライフサイクルは workload とは独立しており、Kubernetes 側の状態を正にする必要があります。([Kubernetes][2])

---

# 失敗しにくい実行順だけ抜き出すと

```bash
# 1. 新PVC作成
kubectl apply -f new-pvc-0.yaml

# 2. コピーPod作成
kubectl apply -f copy-pod.yaml

# 3. 旧DB停止
kubectl -n db scale sts oracle-old --replicas=0
kubectl -n db rollout status sts/oracle-old

# 4. コピー
kubectl -n db exec -it oracle-data-copy-0 -- bash
cp -a /src/. /dst/
sync
exit

# 5. コピーPod削除
kubectl -n db delete pod oracle-data-copy-0

# 6. 新STS作成
kubectl apply -f new-sts.yaml
kubectl -n db rollout status sts/oracle-new

# 7. 動作確認後、旧PVC削除
kubectl -n db delete pvc oradata-oracle-old-0
```

---

# あなたの環境向けの判断

あなたの社内運用が「テスト Pod を消したら storage ディレクトリも削除する」寄りなら、**このコピー方式の方が事故りにくい**です。
理由は、切り替え後に **新 PVC 名・新 PV・新ディレクトリ** が揃うので、現役データと旧データを視覚的に分離しやすいからです。local-path-provisioner の cleanup 方式とも整合します。([GitHub][4])

必要なら次に、あなたの実際の YAML に合わせて、
**旧 StatefulSet YAML を新名へ変換した完成版**と、**copy-pod.yaml / new-pvc.yaml をそのまま使える形**で書きます。

[1]: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com "StatefulSets"
[2]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/?utm_source=chatgpt.com "Persistent Volumes"
[3]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/?utm_source=chatgpt.com "Configure a Pod to Use a PersistentVolume for Storage"
[4]: https://github.com/rancher/local-path-provisioner?utm_source=chatgpt.com "rancher/local-path-provisioner: Dynamically provisioning ..."


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


それぞれは「何を意図しているか」を理解しておくと安全に使えます。順に説明します。

---

# ① `command: ["/bin/bash", "-lc", "sleep infinity"]`

## 何をしているか

👉 **コンテナを「何もせずずっと起動し続ける」ためのコマンド**

---

## 分解

```bash
/bin/bash -lc "sleep infinity"
```

### `/bin/bash`

* bash シェルを起動

---

### `-l`（login shell）

* ログインシェルとして起動
* `/etc/profile` などを読み込む

👉 環境変数やPATHを整える目的

---

### `-c`

* 「この後ろのコマンドを実行せよ」

---

### `"sleep infinity"`

* 無限にスリープ

👉 コンテナが終了しない

---

## なぜ必要か

コピー用 Pod は：

* データコピーのために「一時的に入るだけ」
* すぐ終わると困る

👉 だから「ずっと待機」させる

---

## イメージ

```text
コンテナ起動 → ずっと待機 → kubectl exec で入る → 作業 → exit
```

---

# ② `kubectl -n $NS rollout status sts/$OLD_STS`

## 何をしているか

👉 **StatefulSet の状態が「安定するまで待つ」**

---

## 具体的な意味

たとえば：

```bash
kubectl scale sts oracle-old --replicas=0
```

の後に実行すると：

👉 **全Podが停止するまで待つ**

---

## 出力例

```text
statefulset rolling update complete
```

または

```text
Waiting for 1 pods to be ready...
```

---

## なぜ必要か

Oracle の場合：

👉 **DBが完全停止してからコピーしないと危険**

---

## これを使わないと

* Podがまだ動いている
* データが書き換わる
* コピーが不整合になる

---

# ③ `cp -a /src/. /dst/`

## 何をしているか

👉 **ディレクトリの中身を完全コピー**

---

## 分解

```bash
cp -a /src/. /dst/
```

### `-a`（archive モード）

👉 重要

以下を保持：

* 権限
* 所有者
* タイムスタンプ
* シンボリックリンク

---

### `/src/.`

👉 「srcディレクトリの中身」

※ `/src` ではない

---

### `/dst/`

👉 コピー先

---

## なぜ `/src/.` なのか

### ❌ `/src` にすると

```text
/dst/src/...
```

になる

---

### ✅ `/src/.` にすると

```text
/srcの中身 → /dst直下
```

---

## Oracle 的に重要

Oracle は：

* 所有者（oracleユーザ）
* パーミッション

👉 これが重要

👉 `-a` が必須

---

# ④ `sync`

## 何をしているか

👉 **ディスクに書き込みを強制反映**

---

## なぜ必要か

Linux は：

👉 書き込みをメモリにキャッシュする

---

### `cp` 後の状態

```text
コピー完了（見える）
でもディスクに完全書き込みされていない可能性あり
```

---

### `sync` をすると

👉 **全部ディスクに確実に書く**

---

## なぜ重要か

コピー後すぐ：

* Pod削除
* ノード再起動

すると：

👉 データ欠損の可能性

---

# まとめ（重要）

| コマンド             | 意味           |
| ---------------- | ------------ |
| `sleep infinity` | コンテナを終了させず待機 |
| `rollout status` | 完全停止・安定を待つ   |
| `cp -a`          | 権限含めて完全コピー   |
| `sync`           | ディスクへ確実書き込み  |

---

# 流れとしての意味

```text
① Podを起動（待機）
② DBを止める（rollout status）
③ データコピー（cp -a）
④ 書き込み確定（sync）
```

---

# 一番重要なポイント

👉 **cp -a + sync は「安全なファイルコピーの基本セット」**

---

# 補足（より安全にするなら）

可能なら：

```bash
rsync -aHAX /src/ /dst/
```

の方が

* 差分確認
* 再実行性

で優れています

---

必要なら次に：

* `rsync`版の安全コピー手順
* Oracleで絶対壊さないコピー方法
* UID/GID不一致の対処

まで踏み込みます。


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


ここは運用上かなり重要なポイントです。
「差分確認」「再実行性」は、**コピー作業の信頼性とやり直しのしやすさ**に関わります。

---

# 結論（短く）

👉 **rsync は「何をコピーしたか」「何が違うか」を把握できる**
👉 **途中で失敗しても、安全に何度でもやり直せる**

---

# ① 差分確認とは何か

## cp の場合

```bash
cp -a /src/. /dst/
```

👉 挙動：

* とにかく全部コピー
* 何が変わったか分からない
* ログも弱い

---

## rsync の場合

```bash
rsync -a /src/ /dst/
```

👉 挙動：

* **差分だけコピー**
* 何をコピーしたか表示できる

---

## 例（rsync）

```bash
rsync -av /src/ /dst/
```

出力：

```text
file1.dbf
redo01.log
```

👉 **コピーされたファイルが分かる**

---

## さらに詳細確認

```bash
rsync -av --dry-run /src/ /dst/
```

👉 実際にはコピーしない
👉 **差分だけ表示**

---

## つまり

👉 **「コピー前に何が変わるか確認できる」**

---

# ② 再実行性とは何か

## cp の問題

途中で失敗した場合：

```bash
cp -a /src/. /dst/
```

👉 状態：

* 半分コピー済み
* どこまでコピーしたか不明

---

### 再実行すると

👉 **全部やり直し**

* 時間無駄
* 上書きリスク
* 不整合の可能性

---

## rsync の場合

途中で止まっても：

```bash
rsync -a /src/ /dst/
```

👉 再実行すると：

* **未コピー分だけコピー**
* 既存ファイルはスキップ

---

## 仕組み

rsync は：

* ファイルサイズ
* タイムスタンプ

などで比較して

👉 **同じならコピーしない**

---

## 例

### 1回目（途中失敗）

```text
file1 → OK
file2 → OK
file3 → 失敗
```

---

### 再実行

```text
file1 → スキップ
file2 → スキップ
file3 → コピー
```

---

👉 **効率的・安全**

---

# ③ なぜ重要か（Oracleの場合）

Oracle データは：

* 数十GB〜TB
* コピーに時間がかかる

---

## cp の場合

👉 途中失敗すると

* 最初からやり直し
* ダウンタイム増加

---

## rsync の場合

👉 再実行で続きから

* ダウンタイム短縮
* 安全性向上

---

# ④ 検証性（もう1つ重要）

rsync は比較もできる

---

## 差分チェック

```bash
rsync -avc --dry-run /src/ /dst/
```

👉 checksumで比較

---

## 意味

👉 **完全一致しているか確認できる**

---

# ⑤ 実務での使い分け

| 項目    | cp | rsync |
| ----- | -- | ----- |
| 全コピー  | ◎  | ◎     |
| 差分コピー | ×  | ◎     |
| 途中再開  | ×  | ◎     |
| 確認    | ×  | ◎     |
| 安全性   | △  | ◎     |

---

# ⑥ あなたのケースでの最適

今回：

* Oracle
* 大量データ
* 一発勝負

👉 **rsync 推奨**

---

# ⑦ 実際に使うコマンド

最低限：

```bash
rsync -a /src/ /dst/
```

---

## 推奨（詳細表示）

```bash
rsync -av /src/ /dst/
```

---

## コピー前確認

```bash
rsync -av --dry-run /src/ /dst/
```

---

## 完全一致確認

```bash
rsync -avc --dry-run /src/ /dst/
```

---

# まとめ

👉 差分確認 = 「何がコピーされるか事前に分かる」
👉 再実行性 = 「途中で失敗しても安全に続きからできる」

---

# 一番重要な一文

👉 **rsync は「コピー」ではなく「同期」だから安全**

---

必要なら次に：

* Oracleコピーで絶対にやるべき rsync オプション
* UID/GID問題の対処
* コピー後にDBが起動しない原因

まで実務レベルで詰められます。


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

👉 **tar でもコピーは可能（実務でも使われる）**
👉 ただし **rsyncより安全性・再実行性は劣る**
👉 今回のような Oracle データ移行では **rsyncの方が適している**

---

# tarコピーの基本

よく使うパターンはこれです：

```bash
cd /src
tar cf - . | (cd /dst && tar xpf -)
```

---

## 何をしているか

### 前半

```bash
tar cf - .
```

* `c` = create（アーカイブ作成）
* `f -` = ファイルではなく標準出力へ
* `.` = 現在ディレクトリ

👉 **/srcの中身をストリームとして出力**

---

### 後半

```bash
(cd /dst && tar xpf -)
```

* `x` = 展開
* `p` = パーミッション保持
* `f -` = 標準入力から

👉 **そのまま /dst に展開**

---

## イメージ

```text
/src → tar → パイプ → tar → /dst
```

---

# メリット

## ① シンプル・高速

* 一発でコピー
* 圧縮も可能（gzip等）

---

## ② 権限・所有者を保持

```bash
tar xpf
```

👉 Oracleには重要

---

## ③ sparseファイル対応（オプション）

```bash
tar --sparse
```

👉 Oracleデータファイル向き

---

# デメリット（重要）

ここが rsync との差です。

---

## ① 差分コピーできない

👉 毎回フルコピー

---

## ② 再実行性が弱い

途中で失敗すると：

```text
どこまでコピーされたか不明
```

👉 最初からやり直し

---

## ③ 検証しにくい

👉 コピー結果の比較が難しい

---

## ④ エラー検知が弱い

パイプなので：

* 途中で失敗しても気づきにくい

---

# rsyncとの比較

| 項目    | tar | rsync |
| ----- | --- | ----- |
| フルコピー | ◎   | ◎     |
| 差分コピー | ×   | ◎     |
| 再実行   | △   | ◎     |
| 検証    | △   | ◎     |
| 速度    | ◎   | ◎     |
| 安全性   | △   | ◎     |

---

# Oracle用途での評価

## tarが向くケース

* 一発コピーで終わる
* 小規模データ
* シンプル重視

---

## rsyncが向くケース（今回）

* データ大きい
* 失敗リスクあり
* 再実行したい
* 検証したい

👉 **今回はこちら**

---

# tarを使うなら最低限これ

```bash
cd /src
tar cpf - . | (cd /dst && tar xpf -)
sync
```

---

## 推奨オプション

```bash
tar cpf - --numeric-owner --xattrs --acls .
```

👉 理由：

* `--numeric-owner`：UID/GIDそのまま
* `--xattrs`：拡張属性
* `--acls`：ACL

---

# よくある事故

## ❌ パーミッション崩れ

```bash
tar xf
```

👉 `p` なし

---

## ❌ 所有者ズレ

👉 Oracle起動失敗

---

## ❌ 途中失敗

👉 不完全コピー

---

# 実務的な結論

👉 **tarは「一発コピー専用ツール」**
👉 **rsyncは「安全な移行ツール」**

---

# あなたのケースでの推奨

今回：

* Oracle
* 本番データ
* StatefulSet移行

👉 **rsyncを使うべき**

---

# まとめ

* tarでもコピーは可能
* ただし差分・再実行・検証が弱い
* Oracle移行ではリスクが高い
* rsyncの方が安全で実務向き

---

# 一番重要な一文

👉 **tarは「コピー」、rsyncは「安全な同期」**

---

必要なら次に：

* Oracleで絶対に安全なrsyncコマンド（完全版）
* コピー後に起動しない原因チェックリスト
* UID/GID問題の具体対応

まで詰められます。


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


rsync が使えない前提でも、**安全性を落とさずにコピーする方法**はあります。
結論から言うと：

👉 **最適解は「tarストリーム + 検証付き + 分割チェック」**です
👉 単純な `cp -a` よりも **信頼性を担保しやすい構成にする**のが重要です

---

# 結論（実務向けベストパターン）

## 推奨コマンド

```bash
cd /src
tar cpf - --numeric-owner --xattrs --acls . \
  | (cd /dst && tar xpf -)

sync
```

---

# これだけでは不十分（重要）

👉 **検証工程を必ず入れる**

これが rsync と tar の最大の差です。

---

# なぜ tar を使うのか

rsync が無い場合の選択肢は実質：

| 方法    | 評価 |
| ----- | -- |
| cp -a | △  |
| tar   | ◎  |

---

## cp -a の問題

```bash
cp -a /src/. /dst/
```

* 途中失敗検知しにくい
* コピー漏れが分かりにくい
* 大量ファイルで不安定

---

## tar の利点

* ストリームで一貫コピー
* 権限・所有者維持
* ディレクトリ構造丸ごと転送

---

# 安全な tar コピー手順（完全版）

## Step 1: Oracle停止

```bash
kubectl scale sts <old> --replicas=0
kubectl rollout status sts/<old>
```

👉 **これが最重要**

---

## Step 2: コピー

```bash
cd /src
tar cpf - --numeric-owner --xattrs --acls . \
  | (cd /dst && tar xpf -)
```

---

## Step 3: sync

```bash
sync
```

---

## Step 4: ファイル数チェック

```bash
find /src | wc -l
find /dst | wc -l
```

👉 一致するか確認

---

## Step 5: サイズチェック

```bash
du -sh /src
du -sh /dst
```

👉 大きくズレていないか

---

## Step 6: ランダムチェック（重要）

```bash
ls -l /src | head
ls -l /dst | head
```

または：

```bash
diff -r /src /dst
```

（時間かかるので一部でもOK）

---

# さらに安全にする（推奨）

## ① checksum検証

```bash
cd /src
find . -type f -exec md5sum {} \; > /tmp/src.md5

cd /dst
md5sum -c /tmp/src.md5
```

👉 **完全一致確認**

---

## ② 2回コピー（疑似rsync）

```bash
# 1回目
tar cpf - . | (cd /dst && tar xpf -)

# 2回目（差分補正的に再実行）
tar cpf - . | (cd /dst && tar xpf -)
```

👉 取りこぼし対策

---

# Oracleで特に注意

## 必須オプション

```bash
--numeric-owner
```

👉 UID/GIDを名前ではなく数値でコピー

---

## 理由

Oracle コンテナでは：

```text
oracleユーザ = UID固定（例: 54321）
```

👉 名前解決に依存すると壊れる

---

## sparseファイル対策（重要）

```bash
tar cpf - --sparse .
```

👉 Oracleデータファイル対策

---

# よくある事故

## ❌ 権限崩れ

```bash
tar xf
```

👉 `p` を付けていない

---

## ❌ 所有者ズレ

👉 DB起動失敗

---

## ❌ コピー途中失敗

👉 不整合データ

---

# rsyncとの差（正直な評価）

| 項目    | tar | rsync |
| ----- | --- | ----- |
| 差分コピー | ×   | ◎     |
| 再実行性  | △   | ◎     |
| 検証    | △   | ◎     |
| 安全性   | ○   | ◎     |

---

# 実務的な最適解

rsyncなし環境では：

👉 **tar + 検証（これがセット）**

---

# あなたのケースへの最適化

今回：

* Oracle
* StatefulSet移行
* local-path
* 人手削除あり

👉 **最重要は「確実に同一データを作ること」**

---

# 最終推奨コマンド（そのまま使える）

```bash
cd /src

tar cpf - \
  --numeric-owner \
  --xattrs \
  --acls \
  --sparse \
  . | (cd /dst && tar xpf -)

sync
```

---

# 最重要ポイント

👉 **tar単体ではなく「検証込み」で使うこと**

---

# 一言でまとめ

👉 **rsyncが無いなら、tar + チェックで「疑似rsync」にする**

---

必要なら次に：

* 「Oracle絶対安全コピー（チェックリスト付き）」
* 「コピー後に起動しない場合の原因特定」
* 「UID/GIDズレの修正方法」

まで詰めて説明できます。


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

👉 **技術的には可能だが、推奨しない（特にあなたの環境では）**
👉 **コピーPod方式の方が安全・一貫性が高い**

---

# なぜ直接コピーできるのか

local-path は実体が単なるディレクトリなので：

```bash
cp -a /var/lib/rancher/k3s/storage/old-dir \
      /var/lib/rancher/k3s/storage/new-dir
```

👉 OSレベルでコピー可能

---

# しかし推奨しない理由（重要）

## ① Kubernetesの管理外操作になる

Kubernetesの前提：

```text
PVC → PV → 実体ディレクトリ
```

しかし直接コピーは：

```text
OS → ディレクトリ操作（K8s関知しない）
```

👉 **状態の整合性が崩れる可能性**

---

## ② パス特定ミスのリスク

```bash
kubectl get pv -o yaml
```

で確認しないと：

👉 **間違ったディレクトリをコピー・削除**

---

## ③ コピー先のPVと整合しない

例えば：

```text
コピー先ディレクトリ = /storage/new-dir
PVが指すパス = /storage/another-dir
```

👉 **コピーしても使われない**

---

## ④ 権限・SELinux・マウント問題

コンテナ内とホストで：

* UID/GIDの見え方が違う
* SELinuxラベル（環境による）
* マウントオプション差

👉 Oracle起動失敗の原因

---

## ⑤ 人的ミスと運用衝突

あなたの環境では：

```text
「ディレクトリを手で削除する運用」
```

👉 これと直接操作は非常に相性が悪い

---

# コピーPod方式との比較

| 項目         | 直接コピー | コピーPod |
| ---------- | ----- | ------ |
| 安全性        | △     | ◎      |
| K8s整合性     | ×     | ◎      |
| ミス耐性       | 低     | 高      |
| 再現性        | 低     | 高      |
| トラブル時の切り分け | 困難    | 容易     |

---

# 直接コピーが許されるケース

以下すべて満たすなら可：

* ノードにフル権限あり
* PVの実体パスを完全に把握
* Kubernetes構造を理解
* 手動操作に慣れている
* テスト環境

👉 つまり **上級者限定**

---

# どうしても直接やる場合（安全手順）

## ① PVのパス確認（必須）

```bash
kubectl get pv <pv名> -o yaml
```

```yaml
spec:
  hostPath:
    path: /var/lib/rancher/k3s/storage/xxxxx
```

---

## ② Oracle停止

```bash
kubectl scale sts <old> --replicas=0
kubectl rollout status sts/<old>
```

---

## ③ コピー

```bash
cp -a /path/to/old/. /path/to/new/
sync
```

---

## ④ 権限確認

```bash
ls -l
```

👉 oracleユーザ一致確認

---

## ⑤ PVC/PVと一致確認

👉 ここが最重要

---

# それでもおすすめしない理由（核心）

👉 **Kubernetesの抽象化を破るから**

---

## Kubernetes設計思想

👉 「ストレージはPVC経由で扱う」

---

## 直接コピー

👉 「裏口から直接触る」

---

# あなたの環境での最適解

あなたは：

* local-path
* 手動削除運用あり
* Oracle（重要データ）

👉 **コピーPod一択**

---

# なぜコピーPodが良いのか

## ① PVCを通す

```text
/src → PVC → PV → ディレクトリ
```

👉 Kubernetesの流れに沿う

---

## ② マウントされた状態でコピー

👉 コンテナと同じ視点

---

## ③ パスを意識しなくていい

👉 human error削減

---

# 最終結論

👉 **直接コピーは「できる」が「やるべきではない」**

---

# 一番重要な一文

👉 **Kubernetesでは「ディレクトリではなくPVCを操作する」のが正解**

---

# 推奨判断

| 方法          | 評価     |
| ----------- | ------ |
| コピーPod      | ◎（推奨）  |
| rsync（Pod内） | ◎      |
| tar（Pod内）   | ○      |
| 直接コピー       | △（非推奨） |

---

必要なら次に：

* 「直接コピーで実際に起きた事故例」
* 「Oracleが壊れるパターン」
* 「最短・安全な作業チェックリスト」

まで実務目線で解説できます。
