主因は、**Oracle DB コンテナの初期化方式と、あなたのカスタムイメージの作り方／Kubernetes の永続ボリューム構成が噛み合っていない**可能性が高いです。

このエラーの “error while restoring pdb backup piece” は、DBCA が **PDB$SEED や PDB 作成用バックアップピースを復元しようとして失敗**したときに出る代表的なメッセージです。Oracle 系の事例では、その直下に RMAN / ORA 系の本当の原因がぶら下がっていることが多く、たとえば **ファイル作成先の不整合、権限不足、FRA 設定不整合、パッケージ署名不整合、インベントリ破損**などで発生します。([Stack Overflow][1])

Kubernetes で Oracle DB の**カスタムイメージ**を使う場合、特に多いのは次のパターンです。

1. **DB 作成済みコンテナをそのまま `docker commit` 的に固めた**
   Oracle のコンテナは、ソフトウェア本体と DB データを分けて扱う前提が強く、データ永続化先は `/opt/oracle/oradata` です。Oracle の公式資料でも、既存 DB を再利用する前提は **`/opt/oracle/oradata` のデータボリュームを再利用する**形で説明されています。つまり、**DB をイメージに焼き込んで運ぶ**より、**データは volume/PVC 側で持つ**のが基本です。([Oracle Documentation][2])

2. **Pod 起動時に PVC を `/opt/oracle/oradata` にマウントして、イメージ内の状態を上書きした**
   これをやると、イメージ側に入っていたはずの DB 構成と、Pod 起動時に見えている実際のデータ領域がずれます。
   典型的には、

   * PVC が空だった
   * 途中まで作られた古い DB ファイルが残っていた
   * 別コンテナ／別 SID / 別 PDB 名のデータが入っていた
     などで、起動スクリプトや DBCA が「新規作成」または「途中状態の再開」を始め、PDB 復元工程で落ちます。Oracle の資料でも `/opt/oracle/oradata` はデータファイル・redo・監査ログ等の永続領域であり、ここを再利用して既存 DB を使う前提です。逆に言うと、**ここが期待と違う中身だと壊れます**。([Oracle Documentation][2])

3. **権限・所有者・Storage の問題で、復元先ファイルを作れなかった**
   Oracle コンテナでは `/opt/oracle/oradata` 配下に DBCA/RMAN がファイルを書きます。GitHub の Oracle 公式イメージの事例でも、`Permission denied` により `/opt/oracle/oradata/...` を作れず、その後の DB 起動が壊れる例があります。Kubernetes では `runAsUser`、`fsGroup`、NFS/hostPath の権限、subPath の作成状態が絡みやすいです。([GitHub][3])

4. **メモリ不足や DBCA 実行条件不足**
   DBCA 系の失敗では、メモリ不足が直接または間接原因になることがあります。Oracle の issue でも DBCA 実行時にメモリ警告が出ていますし、周辺事例でも “Error while restoring PDB backup piece” の最終原因としてリソース不足が報告されています。Kubernetes の requests/limits が厳しすぎると起こりえます。([GitHub][4])

5. **`db_recovery_file_dest` などの設定不整合**
   Oracle 関連の既知事例では、DBCA の PDB restore 失敗が `db_recovery_file_dest` 設定で発生し、作成時には外すと回避できるケースがあります。これは特に、**イメージ側の設定と Pod 側のマウント先/FRA 実体が一致していない**場合に疑います。([cd.delphix.com][5])

6. **パッチレベルや Oracle Home の不整合**
   既知不具合として、`SYS.DBMS_BACKUP_RESTORE` の署名不整合や特定バグで同じエラーになる例があります。つまり、**作成元イメージの Oracle Home** と **起動時に実際に使われる DB 構成** の間にズレがあると危険です。カスタムイメージを雑に継承した場合に起こりえます。([Oracle Forums][6])

なので、あなたのケースをかなり実務的に言い切ると、**最有力原因は「DB 作成済み状態をイメージ化したが、Pod 起動時にマウントされた `/opt/oracle/oradata` の内容・権限・初期化状態が一致せず、DBCA の PDB 復元が失敗した」**です。Oracle のコンテナは、既存 DB を使うなら **同じ `/opt/oracle/oradata` を持ち回る**設計に寄っており、**“DB を含むカスタムイメージ” をそのまま K8s で再展開するやり方は壊れやすい**です。([Oracle Documentation][2])

確認すべき箇所はこれです。

* Pod ログだけでなく、**DBCA/RMAN の詳細ログ**
  `cfgtoollogs/dbca/...` や alert log の直下に、実際の ORA-/RMAN- エラーがあります。
  “error while restoring pdb backup piece” 自体は総称なので、**真因はその数行下**です。([Stack Overflow][1])
* `/opt/oracle/oradata` が

  * 空か
  * 途中作成状態か
  * 別 DB の残骸か
  * Oracle ユーザで書けるか
* `ORACLE_SID` / `ORACLE_PDB` / `ORACLE_PWD` / DB 名が、作成済みデータと矛盾していないか
* PVC を `subPath` で切っているなら、そのディレクトリが事前作成されているか、権限が合っているか
* resources の memory limit が小さすぎないか
* カスタムイメージ作成時に Oracle Home へパッチを当てたなら、**その後に作った DB データか**、または **古い DB データを新 Home に無理につないでいないか**

対処としては、次が安定です。

**安定構成**

* イメージには **Oracle ソフト + 追加スクリプトだけ** を入れる
* DB データは **必ず PVC (`/opt/oracle/oradata`)** に持たせる
* 初回起動で DB を作る
* 2回目以降は **同じ PVC を再利用**する

**避ける構成**

* DB 作成済みコンテナを commit して、そのまま K8s に持ち込む
* その上で別の空 PVC を `/opt/oracle/oradata` に被せる
* あるいは、途中状態の PVC を別設定の Pod に再利用する

必要なら次に、あなたの **Dockerfile / StatefulSet(or Pod) YAML / volumeMounts / env / Pod 起動ログの該当部分** を見て、どこが原因かをかなり具体的に絞れます。特に **`/opt/oracle/oradata` のマウント方法** と **DBCA ログの直下の ORA-/RMAN- 行** が決定打になります。

[1]: https://stackoverflow.com/questions/44284623/ora-01017-invalid-username-password-logon-denied-is-thrown-by-the-oracle-12c?utm_source=chatgpt.com "ORA-01017: invalid username/password; logon denied is ..."
[2]: https://docs.oracle.com/en/database/oracle/oracle-database/21/deeck/index.html?utm_source=chatgpt.com "How to Install Oracle Database 21c on Docker"
[3]: https://github.com/oracle/docker-images/issues/227?utm_source=chatgpt.com "Problem with container run · Issue #227 · oracle/docker- ..."
[4]: https://github.com/oracle/docker-images/issues/688?utm_source=chatgpt.com "Errors after container startup · Issue #688 · oracle/docker- ..."
[5]: https://cd.delphix.com/docs/8.0.0.0/linking-provisioning-tde-enabled-19c-oracle-ebs-pd?utm_source=chatgpt.com "Linking & provisioning TDE Enabled (19c) Oracle EBS PDB"
[6]: https://forums.oracle.com/ords/apexds/post/19-9-windows-dbca-fails-error-while-restoring-pdb-backup-pi-0069?utm_source=chatgpt.com "19.9 Windows DBCA Fails \"Error while restoring PDB ..."


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


その前提なら、原因はかなり絞れます。

**「Dockerfileで少し足しただけ」でも、Oracle の初回 DB 作成フローを壊すことは普通にあります。**
特に Oracle のコンテナは、初回起動時に DB を作る仕組みで、セットアップ用スクリプトは `/opt/oracle/scripts/setup`、起動時スクリプトは `/opt/oracle/scripts/startup` で扱う前提です。既存データがあるかどうかも `/opt/oracle/oradata` の状態で判定されます。([Oracle Docs][1])

この条件で `error while restoring pdb backup piece` が出たなら、可能性が高いのは次の4つです。

**1. 追加した処理で Oracle の必要権限や所有者を壊した**
ユーザ作成・グループ作成・権限変更の過程で、Oracle Home や `/opt/oracle/oradata` 配下の所有者・実行権限が変わると、DBCA が PDB 復元時に必要なファイルを作れず失敗します。実際に Oracle 系コンテナでは権限不整合で起動不能になる事例があります。([Conclusion AMIS Technology Blog][2])

**2. パッケージ追加でベースイメージの実行環境を崩した**
`dnf` / `yum` で追加インストールした際に、依存関係の更新で Oracle が前提としているライブラリや OS 設定に影響すると、初回 DB 作成で落ちることがあります。見た目は「PDB backup piece restore failed」でも、実際の真因はその直下の ORA-/RMAN- エラーです。Oracle のコンテナ文書でも、DB 作成は内部の初期化フローに依存しています。([Oracle Docs][1])

**3. `/opt/oracle/oradata` の中身またはマウント状態が期待と違う**
Oracle コンテナは、DB の再利用判定を `/opt/oracle/oradata` のチェックポイントファイルやディレクトリ有無で行います。つまり、Kubernetes 側で PVC をそこにマウントしていて、

* 空ではない
* 途中作成状態の残骸がある
* 権限が違う
* 別 SID の痕跡がある
  と、初期化ロジックが誤判定し、PDB 復元工程で落ちやすいです。([GitHub][3])

**4. setup/startup の使い分けを誤っている**
初回 DB 作成の前後で何を実行するかは重要です。DB 作成前にやるべき処理と、作成後に SQL でやるべき処理が混ざると壊れます。Oracle は `/opt/oracle/scripts/setup` で「セットアップ後」、`startup` で「起動時」のスクリプト実行を案内しています。([Oracle Docs][1])

なので、あなたの説明から最有力なのはこれです。

**最有力原因**
**Dockerfile に入れたスクリプトのどれかが、Oracle の初回 DB 作成前提を壊した。**
とくに危ないのは次です。

* `useradd`, `groupadd` の後に `chown -R` を広くかけた
* `/opt/oracle`, `/opt/oracle/oradata`, `$ORACLE_HOME` 配下を触った
* `oracle` ユーザ以外で実行されるようにした
* `USER root` のまま終わっている
* パッケージ追加時に OS ライブラリが更新された
* setup スクリプト内で DB 作成前に DB 関連ファイルや環境変数を触った

特に **`USER` の最終状態** は重要です。公式イメージの想定実行ユーザから外れると、Kubernetes で Pod は起動しても DBCA が内部で失敗します。Oracle のコンテナはスクリプト実行とデータ領域の扱いに明確な前提があります。([Oracle Docs][1])

確認は次の順でやるのが早いです。

1. **Pod ログの数行下の ORA-/RMAN- を見る**
   `error while restoring pdb backup piece` 自体は総称です。真因はその少し下です。

2. **DBCA ログを見る**
   典型的には `cfgtoollogs/dbca/...` 配下です。ここに実際の失敗理由が出ます。PDB restore 失敗の中身が分かります。([Oracle Docs][4])

3. **Dockerfile を確認する**
   特に

   * `USER`
   * `RUN chown/chmod`
   * 追加パッケージ
   * `/opt/oracle` 配下に触れているか
     を見ます。

4. **Kubernetes の volumeMounts を確認する**
   `/opt/oracle/oradata` に何をどうマウントしているか。空 PVC か、既存データか、subPath かで意味が変わります。Oracle はそこを見て既存 DB を判定します。([GitHub][3])

切り分けとしては、まずこの比較が有効です。

* **公式イメージをそのまま K8s で起動** → 成功するか
* **同じ YAML でカスタムイメージだけ差し替え** → 失敗するか

これで失敗するなら、ほぼ Dockerfile 側です。
さらに、カスタムイメージで **追加スクリプトを全部外して、1つずつ戻す** と犯人を特定できます。

実務的には、Oracle コンテナのカスタマイズは次の方針が安全です。

* OS パッケージ追加は最小限
* Oracle の既存ユーザ/グループ/権限をむやみに変更しない
* `/opt/oracle/oradata` は DB データ専用として扱う
* DB ユーザ作成や tablespace 作成は、できるだけ **setup の SQL** でやる
* root 作業後は最終的に想定ユーザへ戻す

Dockerfile と StatefulSet の該当部分を貼れば、どこが危険かをかなり具体的に指摘できます。特に見たいのは **`FROM` 以降の `RUN`、`USER`、`ENTRYPOINT/CMD`、そして `volumeMounts`** です。

[1]: https://docs.oracle.com/en/database/oracle/oracle-database/21/deeck/index.html?utm_source=chatgpt.com "How to Install Oracle Database 21c on Docker"
[2]: https://technology.amis.nl/database/databases/oracle-database-21c-xe-setting-up-a-demo-environment-using-vagrant-and-oracle-virtualbox-part-3/?utm_source=chatgpt.com "Oracle Database 21c XE, setting up a demo environment ..."
[3]: https://github.com/oracle/docker-images/issues/2599?utm_source=chatgpt.com "Setup script ignored if using image container-registry. ..."
[4]: https://docs.oracle.com/cd/F39414_01/deeck/oracle-database-enterprise-edition-installation-guide-docker-containers-oracle-linux.pdf?utm_source=chatgpt.com "Oracle Database Enterprise Edition Installation Guide for ..."
