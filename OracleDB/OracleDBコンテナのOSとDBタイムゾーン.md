Oracleコンテナでは、**OSタイムゾーン（Linux）とDBタイムゾーン（Oracle）は完全に別レイヤー**です。両方を明示的に制御する必要があります。

---

# ■ 全体像（重要）

* OSタイムゾーン → コンテナの `date` / ログ時刻
* DBタイムゾーン → `TIMESTAMP WITH TIME ZONE` などの内部基準

👉 **一致させないと時刻ズレの原因になります**

---

# ■ 1. OSタイムゾーンの設定（コンテナ側）

## 方法①：環境変数 `TZ`（最も簡単）

```yaml
env:
  - name: TZ
    value: "Asia/Tokyo"
```

または Docker:

```dockerfile
ENV TZ=Asia/Tokyo
```

![Image](https://www.cyberciti.biz/media/new/faq/2024/04/How-to-Check-Timezone-in-Linux-using-the-CLI.png)

![Image](https://oneuptime.com/blog/post/2026-01-16-docker-container-timezone/social-media.png)

![Image](https://cdn-ak.f.st-hatena.com/images/fotolife/s/sbc-web/20170501/20170501221349.png)

![Image](https://image.itmedia.co.jp/ait/articles/1906/13/yh201906command_312-02.png)

---

## 方法②：/etc/localtime を直接設定（確実）

```dockerfile
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
```

または：

```bash
cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
```

---

## 確認

```bash
date
```

---

# ■ 2. Oracle DBタイムゾーンの設定

## 現在値確認

```sql
SELECT DBTIMEZONE, SESSIONTIMEZONE FROM DUAL;
```

---

## 方法①：DBタイムゾーン変更（DB全体）

```sql
ALTER DATABASE SET TIME_ZONE = 'Asia/Tokyo';
```

⚠ 注意:

* **即時反映されない**
* **DB再起動が必要**

```sql
SHUTDOWN IMMEDIATE;
STARTUP;
```

---

## 方法②：セッション単位（安全）

```sql
ALTER SESSION SET TIME_ZONE = 'Asia/Tokyo';
```

👉 アプリ単位で制御する場合はこちらが一般的

---

## 方法③：環境変数（Oracleクライアント）

```bash
export ORA_SDTZ=Asia/Tokyo
```

* セッション開始時のデフォルトタイムゾーンになる

---

# ■ 3. Kubernetesでの実用構成例

```yaml
env:
  - name: TZ
    value: "Asia/Tokyo"
  - name: ORA_SDTZ
    value: "Asia/Tokyo"
```

👉 これで

* OS
* セッション

は揃う

---

# ■ 4. 推奨構成（実務）

### パターンA（安定・推奨）

* OS → UTC
* DB → UTC
* アプリ → JST変換

```bash
TZ=UTC
ORA_SDTZ=UTC
```

👉 分散システムではこれが標準

---

### パターンB（日本ローカル用途）

* OS → Asia/Tokyo
* DB → Asia/Tokyo

```bash
TZ=Asia/Tokyo
ORA_SDTZ=Asia/Tokyo
ALTER DATABASE SET TIME_ZONE = 'Asia/Tokyo';
```

---

# ■ 5. よくある落とし穴

### ❌ OSだけ変える

→ DB内部はUTCのまま → 時刻ズレ

---

### ❌ DBTIMEZONEだけ変える

→ セッションは変わらない

---

### ❌ コンテナ再作成でリセット

→ DB設定は永続化されるが、OS設定は毎回必要

---

# ■ 結論

最低限これをやればよい：

```yaml
env:
  - name: TZ
    value: "Asia/Tokyo"
  - name: ORA_SDTZ
    value: "Asia/Tokyo"
```

＋必要なら：

```sql
ALTER DATABASE SET TIME_ZONE = 'Asia/Tokyo';
```

---

必要であれば、
「TIMESTAMP / DATE の挙動の違い」や
「JDBCでのタイムゾーン事故パターン」まで掘り下げて説明できます。
