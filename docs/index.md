# INDEX（データベースの索引）

## 1分サマリ

**辞書の巻末索引と同じ発想**。テーブル全部をフルスキャンせずに、目的の行をピンポイントで見つけるための補助データ構造（多くは B-Tree）。

- **SELECT は速くなる**（O(N) → O(log N)）
- **INSERT / UPDATE / DELETE は遅くなる**（索引も更新するため）
- **容量を食う**

面接で 15 秒で答えるなら：「**カラムに対する B-Tree などの索引で、検索を O(N) から O(log N) にする。代わりに書き込みコストとディスク容量が増えるトレードオフ**」。

---

## 本質の理解

### なぜあるのか

```sql
SELECT * FROM users WHERE name = 'taro';
```

INDEX なしだと：**全 100 万行を 1 行ずつ見る**（フルテーブルスキャン、O(N)）。

INDEX ありだと：name 列を**ソートした B-Tree** から二分探索的にたどる（O(log N)）。

### B-Tree の中身（ざっくり）

```
          [m]
         /   \
      [d,h]   [r,v]
      / | \    / | \
    ... ... ... 行ポインタ
```

- 各ノードに複数のキーを持つ「太い二分木」
- 全ての葉ノードが同じ深さ（バランスツリー）
- 深さが log N に抑えられるので、100 万行でも数回の読み込みで葉に到達

**葉ノードには**「行の実体（クラスタード INDEX の場合）」もしくは「行への参照（プライマリキーや行ID）」が入っている。

### トレードオフ

| 操作 | INDEX あり | INDEX なし |
|---|---|---|
| `SELECT WHERE` | 速い | 遅い |
| `INSERT` | やや遅い（索引更新） | 速い |
| `UPDATE`（索引列） | 遅い | 普通 |
| `DELETE` | やや遅い（索引更新） | 普通 |
| ディスク容量 | 増える | 増えない |

→ **読み多め / 書き少なめのワークロードに効く**。書き込み主体のテーブルへの安易な INDEX は逆効果。

---

## ハンズオン

### 0. 環境

SQLite（Mac には標準で入っている）を使う。

```bash
sqlite3 /tmp/index-demo.db
```

### 1. 10 万行のテーブルを作る

```sql
CREATE TABLE users (
  id   INTEGER PRIMARY KEY,
  name TEXT,
  age  INTEGER
);

-- 10 万件を一気に挿入
WITH RECURSIVE seq(n) AS (
  SELECT 1 UNION ALL SELECT n+1 FROM seq WHERE n < 100000
)
INSERT INTO users(name, age)
SELECT 'user_' || n, (n % 80) + 1 FROM seq;
```

### 2. INDEX なしの状態を `EXPLAIN QUERY PLAN` で見る

```sql
EXPLAIN QUERY PLAN
SELECT * FROM users WHERE name = 'user_77777';
```

出力（だいたいこんな感じ）:
```
SCAN users
```

`SCAN` = フルテーブルスキャン。

実時間も計っておく:
```sql
.timer on
SELECT * FROM users WHERE name = 'user_77777';
```

### 3. INDEX を貼る

```sql
CREATE INDEX idx_users_name ON users(name);
```

### 4. もう一度 `EXPLAIN QUERY PLAN`

```sql
EXPLAIN QUERY PLAN
SELECT * FROM users WHERE name = 'user_77777';
```

出力:
```
SEARCH users USING INDEX idx_users_name (name=?)
```

`SEARCH ... USING INDEX` に変わった = 索引を使っている。実行時間も体感できるレベルで速くなる。

### 5. INDEX が使われないケースを作る

```sql
-- 関数を噛ませる → INDEX 効かない
EXPLAIN QUERY PLAN
SELECT * FROM users WHERE LOWER(name) = 'user_77777';
-- SCAN users（INDEX を使えない）

-- 前方一致は使えるが、後方一致は使えない
EXPLAIN QUERY PLAN SELECT * FROM users WHERE name LIKE 'user_77%';   -- INDEX 使える
EXPLAIN QUERY PLAN SELECT * FROM users WHERE name LIKE '%77777';     -- INDEX 使えない
```

### 6. 複合インデックスの「左端から使う」原則

```sql
CREATE INDEX idx_users_age_name ON users(age, name);

-- (age) のみ条件 → 使える
EXPLAIN QUERY PLAN SELECT * FROM users WHERE age = 30;

-- (age, name) 両方 → 使える
EXPLAIN QUERY PLAN SELECT * FROM users WHERE age = 30 AND name = 'user_5';

-- (name) のみ → idx_users_age_name は使われない（idx_users_name の方が使われる）
EXPLAIN QUERY PLAN SELECT * FROM users WHERE name = 'user_5';
```

`(age, name)` の複合 INDEX は **age で絞ってから name で絞る** という構造になっている。なので `name` だけで検索しても役立たない。これが「左端プレフィックスの原則」。

---

## よくある誤解と落とし穴

### ❌ 「INDEX を貼れば必ず速くなる」
✅ 以下のケースでは使われないか、むしろ遅くなる：

- **テーブルが小さい**（フルスキャンの方が速い）
- **選択性が低い**（boolean のように同じ値だらけ→ INDEX 経由でほぼ全行読むことになる）
- **取得する行数が多い**（テーブル全体の 20-30% を超えるならフルスキャンの方が速いことが多い）
- **関数や式を噛ませている**（`WHERE LOWER(name) = ...`）

### ❌ 「複合インデックスは順番関係ない」
✅ **超関係ある**。`(a, b)` と `(b, a)` は別物。

- `(a, b)` は「a で絞ってから b」だから、`WHERE b = ?` 単独では使えない
- 並び替えにも影響する: `ORDER BY a, b` は使えるが `ORDER BY b, a` には使えない

### ❌ 「INDEX を増やせば増やすほどいい」
✅ INDEX が 10 個あれば INSERT のたびに 10 個更新する必要がある。容量も食う。**「使われていない INDEX」は害**。`pg_stat_user_indexes` などで使用頻度を見て不要なものは消す。

### ❌ 「主キーは INDEX とは別物」
✅ 主キーには**自動で INDEX が作られる**（多くの RDBMS でクラスタード INDEX）。だから `WHERE id = ?` は最初から速い。

### ❌ 「カバリングインデックス？聞いたことない」
✅ **SELECT する全カラムが INDEX に含まれている**状態。テーブル本体を見に行かなくて済むので超速い：
```sql
CREATE INDEX idx_cover ON users(name, age);
SELECT name, age FROM users WHERE name = 'taro';  -- INDEX だけで完結
```

---

## 章末 Q&A

**Q1. なぜ B-Tree は O(log N) で検索できるのか？**

<details>
<summary>解答</summary>

B-Tree は**バランス木**で、全ての葉が同じ深さに保たれる。各ノードに枝が m 本あれば、N 個の要素を保持するための深さは log_m(N) になる。1 ノードはディスク 1 ブロックに収まるサイズで設計されるため、ディスク I/O 回数も log_m(N)。これにより 100 万行でも 3〜4 回程度の I/O で目的の葉にたどり着ける。
</details>

**Q2. INDEX を貼っているのに SELECT が遅い。考えられる原因は？**

<details>
<summary>解答</summary>

- **WHERE 句で関数を使っている**（`WHERE LOWER(col) = ...`）→ INDEX が無効化
- **暗黙の型変換**（文字列カラムに数値を渡すなど）
- **後方一致 LIKE**（`LIKE '%xxx'`）
- **取得行数が多すぎる**（テーブルの大部分を返す）→ オプティマイザがフルスキャンを選ぶ
- **統計情報が古い**（`ANALYZE` で更新）
- **複合 INDEX の左端プレフィックスを満たしていない**
- **OR 条件**で別々のカラムを使っている

`EXPLAIN` でプランを確認するのが第一歩。
</details>

**Q3. ユニークインデックスと通常のインデックスの違いは？**

<details>
<summary>解答</summary>

- **データ構造は基本同じ**（B-Tree）
- **ユニーク INDEX は重複を許さない**（INSERT 時に既存値があればエラー）
- 検索速度は同等だが、「一意性が保証されている」ことを**オプティマイザが利用**できるため、「1 件見つけたら終わり」と判断して早く打ち切れる
- 制約として機能する（メールアドレスの一意性など）
</details>

**Q4. カーディナリティとは？ INDEX 設計とどう関係する？**

<details>
<summary>解答</summary>

カーディナリティ = **そのカラムの値のバリエーション数**（一意な値の数）。

- 高いカーディナリティ（user_id、メールアドレス等）→ INDEX が**よく効く**（絞り込みが強い）
- 低いカーディナリティ（性別、is_active 等）→ INDEX が**効きにくい**（B-Tree でたどっても結局たくさん読む）

ただし「low cardinality + 偏った分布」（例: is_deleted=true は 1% しかない）の場合は**部分インデックス**で有効化できる:
```sql
CREATE INDEX ON orders(is_deleted) WHERE is_deleted = true;
```
</details>

**Q5. INSERT が遅くなったテーブルがある。INDEX を見直す方針は？**

<details>
<summary>解答</summary>

1. **使われていない INDEX を特定して削除**
   - PostgreSQL: `pg_stat_user_indexes` で `idx_scan = 0` のもの
2. **重複・冗長な INDEX を統合**
   - `(a)` と `(a, b)` があれば、前者は後者で代用できることが多い
3. **書き込みが多いテーブルは INDEX 数を絞る**
4. **バルクインサート時は INDEX を一旦 DROP → INSERT → 再作成**
5. **不要な UNIQUE 制約を見直す**
</details>

---

## 模擬面接質問

1. **INDEX の仕組みを 1 分で説明してください。**
2. **あるテーブルで SELECT が遅い時、まず何を確認しますか？**
3. **複合インデックス `(a, b)` と `(b, a)` の違いは？**
4. **INDEX が多すぎることで起きる問題は？**
5. **B-Tree インデックスと Hash インデックスの違いと使い分けは？**
