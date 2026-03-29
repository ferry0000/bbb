状況は正しく理解できています。
提示されたプロセスを見る限り：

* PID 1 → `su oracle -c /opt/oracle/runOracle.sh`
* Oracle関連プロセス → **oracleユーザで実行されている**

👉 つまり **起動構成は完全に正しいです** 

---

# 本題：kubectl exec で oracle ユーザにする方法

結論から言うと、方法は3つあります。

---

## 方法①（最もシンプル）

```bash
kubectl exec -it oracle-test-bootstrap -- su - oracle
```

または

```bash
kubectl exec -it oracle-test-bootstrap -- su oracle
```

### 違い

* `su oracle` → 環境維持（軽い）
* `su - oracle` → ログインシェル（PATHなど初期化）

---

## 方法②（直接 oracle で bash 起動）

```bash
kubectl exec -it oracle-test-bootstrap -- su oracle -c bash
```

👉 これで最初から oracle ユーザの bash

---

## 方法③（--user を使う）※ベスト

```bash
kubectl exec -it oracle-test-bootstrap --user=oracle -- bash
```

または UID 指定：

```bash
kubectl exec -it oracle-test-bootstrap --user=54321 -- bash
```

👉 これが一番クリーン

---

# ただし重要な制約

### `--user` が効く条件

* コンテナが root で起動されている（OK）
* container runtime が対応している（通常OK）

---

# 推奨順位

実務的には：

| 方法              | 評価       |
| --------------- | -------- |
| `--user=oracle` | ◎ ベスト    |
| `su oracle`     | ○ 確実     |
| `su - oracle`   | △（環境変わる） |

---

# よくある勘違い

> 「kubectl execしたらoracleで入りたい」

👉 これは Kubernetes 的には標準ではない

理由：

* `kubectl exec` は「新しいプロセスを起動」
* デフォルトは **コンテナのユーザ設定（今回 root）**

---

# もし毎回oracleにしたいなら

コンテナ側で：

```yaml
securityContext:
  runAsUser: 54321
```

にすれば、

```bash
kubectl exec -it ... -- bash
```

でも oracle になります。

---

# まとめ

* DBプロセスは oracle で動いている → 正常
* `kubectl exec` は別プロセス → rootになるのは正常
* oracleで入りたいなら：

```bash
kubectl exec -it oracle-test-bootstrap --user=oracle -- bash
```

👉 これが最短・最適

---

必要なら：

* `kubectl exec` の内部動作（containerd / CRIレベル）
* なぜ root になるかの仕組み

まで踏み込んで説明できます。


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

> **そのままでは実行できない（失敗する可能性が高い）**

理由は明確です。

---

# なぜ実行できないか

```yaml
securityContext:
  runAsUser: 54321
```

にすると、

> **コンテナの最初のプロセスが oracle ユーザで起動される**

つまり bootstrap.sh は：

```bash
uid=54321(oracle)
```

で実行されます。

---

## あなたの bootstrap.sh の処理

ログから：

```bash
groupadd -g 1300 scc
useradd -u 12345 -g 1300 sccuser
usermod -aG 1300 oracle
mkdir -p /aaa/bbb/ccc
chown -R 12345:1300 /aaa/bbb/ccc
```

👉 これらは全部 **root 権限が必要**

---

## 具体的に何が失敗するか

| コマンド     | 結果                |
| -------- | ----------------- |
| groupadd | Permission denied |
| useradd  | Permission denied |
| usermod  | Permission denied |
| chown    | 一部NG              |
| mkdir    | パス次第でNG           |

---

# 結論（重要）

| 設定               | bootstrap.sh |
| ---------------- | ------------ |
| runAsUser: 0     | ✅ 実行可能       |
| runAsUser: 54321 | ❌ 失敗         |

---

# 正しい設計（あなたのケース）

あなたの要件は：

* rootで初期化したい
* その後oracleでDB起動
* execでoracleログインもしたい

👉 最適解はこれです

---

## 設計パターン

### Pod

```yaml
securityContext:
  runAsUser: 0
```

### bootstrap.sh

```bash
# rootで実行
mkdir /aaa
chown oracle:oinstall /aaa

# oracleで起動
exec su oracle -c "/opt/oracle/runOracle.sh"
```

---

## exec時だけoracleにする

```bash
kubectl exec -it --user=oracle ... -- bash
```

👉 これで解決

---

# 「全部oracleでやりたい」は無理

もし：

> runAsUser: oracle にしたい

なら：

* groupadd / useradd → Dockerfileでやる
* chown → イメージビルド時 or initContainer

---

# 実務的なベストプラクティス

| 処理        | 実行場所                        |
| --------- | --------------------------- |
| ユーザ作成     | Dockerfile                  |
| ディレクトリ作成  | Dockerfile or initContainer |
| DB起動      | oracle                      |
| bootstrap | 最小限                         |

---

# まとめ

* `runAsUser: 54321` → bootstrapはほぼ確実に壊れる
* rootで起動 → bootstrap → oracleに切替 が正解
* execは `--user=oracle` で対応

---

必要なら：

* 「root不要にする設計（完全oracle運用）」
* 「Dockerfileに寄せる構成」

まで整理できます。
