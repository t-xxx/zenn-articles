---
title: "ハッシュジョインにおけるマイクロパーティションプルーニングの注意"
emoji: ""
type: "snowflake"
topics: []
published: false
---

# ハッシュジョインとマイクロパーティションプルーニングの仕組み【Snowflake事例で解説】

Snowflakeにおけるクエリ最適化の中でも重要な要素の1つが「ハッシュジョイン」です。本記事では、ハッシュジョインの仕組みと、それがどのようにマイクロパーティションのプルーニングと連携してパフォーマンス向上を図っているのかについて、図とコードを交えて解説します。


## ハッシュジョインとは？

ハッシュジョインは、**結合条件の片側のテーブルをメモリ上にハッシュテーブルとして保持し、もう片方のテーブルの行と照合して結合する手法**です。特に大規模なデータに対して高速な結合を可能にするアルゴリズムです。

### 💡 処理の流れ

以下のようなクエリを例に考えます：

```sql
SELECT *
FROM orders o
JOIN customers c
  ON o.customer_id = c.customer_id;
```

この場合、通常は小さい方のテーブル（customers）がハッシュ化され、大きなテーブル（orders）と照合されます。

![ハッシュジョインの図解](https://raw.githubusercontent.com/t-xxx/zenn-articles/main/images/hash_join.png)

- Build phase：小さいテーブル (customers) をハッシュテーブルにする
- Probe phase：大きいテーブル (orders) の行を1つずつ走査して一致するキーを探す


## Snowflakeのマイクロパーティションとプルーニング
Snowflakeでは、テーブルはマイクロパーティション（約16MB〜）という単位で物理的に分割されて保存されます。

各マイクロパーティションには、**統計情報（メタデータ）**として以下が保持されます：

- カラムの最小値・最大値
- NULLの有無
- ディストリビューション情報（ヒストグラム）

これにより、クエリ実行時に**不要なマイクロパーティションをスキャン対象から除外（プルーニング）**することで、高速化を実現しています。

## ハッシュジョインとプルーニングの連携
以下のようなクエリを例に考えます：

```sql
SELECT *
FROM orders o
JOIN customers c
  ON o.customer_id = c.customer_id
WHERE c.country = 'Japan';
```

この場合、customers テーブルに country = 'Japan' のフィルタが効いていれば、Snowflakeは以下のように処理します：

![プルーニングとハッシュジョインの図解](https://raw.gihubusercontent.com/t-xxx/zenn-articles/main/images/pruning_hash_join_diagram.drawio.png)

これにより、不要なデータのスキャンを抑制しつつ、メモリ効率も向上します。

## プルーニングが効かないパターン
以下のようなケースでは、プルーニングが効かない可能性があります：

```sql
SELECT *
FROM orders o
JOIN customers c
  ON o.customer_id = c.customer_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '7 days';
```

このようにフィルター条件がジョインキーとは無関係で、かつ動的な値を使っている場合、プルーニングは十分に効かないことがあります。

さらに、サブクエリやビュー、関数を挟んだ複雑な条件では、Snowflakeのオプティマイザが条件を正しく評価できず、プルーニングの効果が減少します。

## 最適化のためのポイント


- JOIN前にWHERE句でフィルタを入れる：フィルタ→JOINの順序が重要
- フィルタ条件にリテラル値を使う（または定数に近い式）
- ビューやCTE内で複雑な計算をしない：オプティマイザが正しくプルーニングできるようにする

##まとめ

Snowflakeにおけるパフォーマンスチューニングは、単体の技術というよりもハッシュジョイン × プルーニングなど、複数の技術の連携で成立しています。クエリパフォーマンスが思わしくない時は、Explain PlanやQuery Profileを確認し、これらの技術が期待通りに使われているかをチェックしましょう。