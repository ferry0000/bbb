はい、できます。
ただし、**「Kubernetesそのものから直接エクスポートする」というより、実際にはノード上のコンテナランタイムから取り出す**形です。

整理すると、対象は主に2通りあります。

## 1. もとのイメージを tar などで書き出す

たとえば、ノードに取り込まれているイメージを `docker save` や `ctr` / `crictl` などでエクスポートします。

### Docker を使っている場合

```bash
docker save -o myimage.tar myrepo/myimage:tag
```

### containerd を使っている場合

```bash
ctr -n k8s.io images export myimage.tar myrepo/myimage:tag
```

環境によっては `crictl` では export が弱いので、`ctr` や `nerdctl` を使うことが多いです。

### nerdctl を使う場合

```bash
nerdctl -n k8s.io save -o myimage.tar myrepo/myimage:tag
```

---

## 2. 実行中コンテナの状態を新しいイメージとして保存してからエクスポート

これは少し別物です。
**Pod内で変更された状態込みで保存したい**なら、まずコンテナをイメージ化して、その後 export します。

Docker系なら概念的にはこうです。

```bash
docker commit <container_id> myimage:modified
docker save -o myimage_modified.tar myimage:modified
```

ただし、Kubernetesでは通常、**Pod内の変更をイメージとして保存する運用はあまり推奨されません**。
設定やデータは通常、以下で管理します。

* イメージ: アプリ本体
* 設定: ConfigMap / Secret
* 永続データ: PVC
* 初期化処理: initContainer / Job

---

## k8sで注意する点

Kubernetesクラスタでは、イメージは**各ノードのランタイムに存在**します。
そのため、エクスポートしたいなら

* そのイメージを保持しているノードに入る
* そのノードの container runtime で save/export する

という流れになります。

たとえば Pod が node1 で動いていて、そのイメージが node1 の containerd にあるなら、node1 上で export します。

---

## よくある誤解

### `kubectl` だけでイメージをエクスポートできるか

通常はできません。
`kubectl` は Kubernetes API を操作するツールで、**イメージファイルを tar に書き出す機能はありません**。

### `kubectl cp` で取れるか

`kubectl cp` は **Pod内のファイル** をコピーするものです。
イメージそのものを直接取り出す用途ではありません。

---

## 実務上の考え方

「インポートしたイメージファイルをまた外に出したい」という話なら、実際には次のどちらかです。

* **元の tar を保管しておく**
* **ノード上の runtime から save/export する**

再利用や移送が目的なら、最初から

* レジストリに push する
* tar をアーカイブ保存する

ほうが管理しやすいです。

必要なら次に、**containerd環境のKubernetesで実際にイメージをtarへ書き出す手順**を、そのまま使えるコマンド例で書きます。


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


以下は、**Kubernetes が containerd を使っている環境**で、
**ノード上にあるイメージを tar にエクスポートする実手順**です。

---

# 全体の流れ

1. Pod がどのノードで動いているか確認
2. その Pod が使っているイメージ名を確認
3. 対象ノードにログイン
4. `ctr` または `nerdctl` でイメージを tar に保存
5. 必要ならノード外へコピー

---

# 1. Pod がどのノードで動いているか確認

まず対象 Pod を確認します。

```bash
kubectl get pod -n <namespace> -o wide
```

例:

```bash
kubectl get pod -n default -o wide
```

出力例:

```bash
NAME         READY   STATUS    RESTARTS   AGE   IP            NODE
oracle-0     1/1     Running   0          2d    10.244.1.15   worker-1
```

この場合、対象ノードは `worker-1` です。

---

# 2. Pod が使っているイメージ名を確認

```bash
kubectl get pod <pod名> -n <namespace> -o jsonpath='{.spec.containers[*].image}'
```

例:

```bash
kubectl get pod oracle-0 -n default -o jsonpath='{.spec.containers[*].image}'
```

出力例:

```bash
myrepo/oracle-custom:1.0
```

initContainer も確認したいならこちらです。

```bash
kubectl get pod <pod名> -n <namespace> -o jsonpath='{.spec.initContainers[*].image}{"\n"}{.spec.containers[*].image}{"\n"}'
```

---

# 3. 対象ノードにログイン

対象ノードに SSH などで入ります。

```bash
ssh <node-user>@worker-1
```

ここで重要なのは、**そのイメージを保持しているノードで実行する**ことです。
Kubernetes では、イメージはクラスタ共通の1か所にあるのではなく、**各ノードの containerd に格納**されています。

---

# 4. ノード上でイメージ一覧を確認

まず本当に存在するか確認します。

## ctr を使う場合

```bash
sudo ctr -n k8s.io images ls
```

絞るなら:

```bash
sudo ctr -n k8s.io images ls | grep oracle
```

## nerdctl を使う場合

```bash
sudo nerdctl -n k8s.io images
```

---

# 5. イメージを tar にエクスポート

## 方法A: ctr を使う

```bash
sudo ctr -n k8s.io images export /tmp/myimage.tar myrepo/oracle-custom:1.0
```

### 例

```bash
sudo ctr -n k8s.io images export /tmp/oracle-custom-1.0.tar myrepo/oracle-custom:1.0
```

---

## 方法B: nerdctl を使う

```bash
sudo nerdctl -n k8s.io save -o /tmp/myimage.tar myrepo/oracle-custom:1.0
```

### 例

```bash
sudo nerdctl -n k8s.io save -o /tmp/oracle-custom-1.0.tar myrepo/oracle-custom:1.0
```

---

# 6. 作成された tar を確認

```bash
ls -lh /tmp/oracle-custom-1.0.tar
```

必要なら内容確認:

```bash
tar -tf /tmp/oracle-custom-1.0.tar | head
```

---

# 7. ノード外へ持ち出す

ローカル PC に持ってくるなら、ローカル側から:

```bash
scp <node-user>@worker-1:/tmp/oracle-custom-1.0.tar .
```

踏み台があるなら、それに合わせて `scp` や `sftp` を使います。

---

# 補足1: Pod から直接は通常できない

`kubectl exec` で Pod に入っても、普通はそこからイメージそのものを tar で取り出すことはしません。
理由は、**イメージは Pod 内ではなくノード側の container runtime が管理している**からです。

つまり、

* `kubectl cp` → Pod内ファイルのコピー
* `ctr images export` → ノード上のイメージの書き出し

で、対象が違います。

---

# 補足2: どのノードにあるか分からない場合

イメージは複数ノードに存在し得ます。
対象 Pod が今動いているノードで実行するのが最も確実です。

Pod のノード確認:

```bash
kubectl get pod <pod名> -n <namespace> -o wide
```

より直接的に見るなら:

```bash
kubectl get pod <pod名> -n <namespace> -o jsonpath='{.spec.nodeName}{"\n"}'
```

---

# 補足3: イメージ名が分からない場合

Deployment / StatefulSet からも確認できます。

## Deployment

```bash
kubectl get deploy <deploy名> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

## StatefulSet

```bash
kubectl get sts <sts名> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

---

# 補足4: import した tar を再度 export できるか

はい、基本的にはできます。
たとえば過去にこのように import していたなら:

```bash
sudo ctr -n k8s.io images import myimage.tar
```

その後、containerd 内にそのイメージが存在していれば、再び:

```bash
sudo ctr -n k8s.io images export exported.tar <image名>
```

で書き出せます。

ただし、**export 時には「tar ファイル名」ではなく「containerd に登録されているイメージ名」**を指定します。
ここはよく混同されます。

---

# 補足5: digest で指定することもある

タグ付き名でうまく見つからない場合、一覧で digest を見て確認します。

```bash
sudo ctr -n k8s.io images ls
```

場合によっては、登録名が想定と少し違うことがあります。
そのときは、一覧に出ている **REF** をそのまま export に使います。

---

# 最小手順だけ抜くと

```bash
kubectl get pod -n default -o wide
kubectl get pod oracle-0 -n default -o jsonpath='{.spec.containers[*].image}{"\n"}'
ssh <node-user>@<node名>
sudo ctr -n k8s.io images ls | grep oracle
sudo ctr -n k8s.io images export /tmp/oracle-custom.tar myrepo/oracle-custom:1.0
scp <node-user>@<node名>:/tmp/oracle-custom.tar .
```

---

# 典型的な詰まりどころ

## 1. `ctr: image not found`

原因:

* イメージ名が違う
* そのノードにイメージがない
* `k8s.io` namespace を付けていない

確認:

```bash
sudo ctr -n k8s.io images ls
```

---

## 2. `ctr` コマンドがない

ノードに `ctr` はあることが多いですが、PATH にない場合があります。

確認:

```bash
which ctr
```

なければ:

* `containerd` パッケージの有無を確認
* `nerdctl` が使えるか確認

---

## 3. 権限エラー

containerd ソケットに触るので `sudo` が必要なことが多いです。

---

## 4. Pod は動いているが export できない

その Pod が使うイメージが、今見ているノードではなく別ノードにある可能性があります。
まず `kubectl get pod -o wide` でノードを確認してください。

---

必要なら次に、
**「ctr images import した tar を、別クラスタに持って行って再 import する手順」**まで、往復の流れでまとめます。
