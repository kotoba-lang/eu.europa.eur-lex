# eu.europa.eur-lex

`publications.europa.eu`（Cellar）から取得した、**EU 法の依存グラフ全体**と **現行 EU 指令の全文** を保全する DataLad dataset です。統合用 datom は `etzhayyim/global-legislation-datoms` が生成します。

この dataset には **2 つの層**があり、カバレッジが違います。混同しないように分けて書きます。

| 層 | 対象 | カバレッジ |
|---|---|---|
| **メタデータ + 依存辺** | 現行効力を持つ EU 法令 **全種別** | 全件 |
| **全文** | 現行効力を持つ **指令 (Directive)** | 1,114 / 1,114 全件 |

規則 (Regulation) は辺とメタデータは持っていますが**本文はまだ持っていません**（wave-2、`raw/source-catalog.edn` の `:catalog/known-gaps` に記録）。

## レイヤ

| パス | 中身 | git での扱い |
|---|---|---|
| `raw/sparql/metadata-*.json` | 現行法令の CELEX・種別・日付・ELI・英語題名 | git-annex → B2 |
| `raw/sparql/relations-<述語>-*.json` | **構造的な法令間の辺**（下記） | git-annex → B2 |
| `raw/text/<CELEX>.xhtml` (+ `.meta`) | Cellar 本文。`.meta` に実際の言語とメディア型 | git-annex → B2 |
| `raw/source-catalog.edn` | 全ファイルの sha256（custody 記録） | **git 本体** |
| `index/laws.edn` / `index/relations.edn` | 消費者が読む索引（数 MB） | **git 本体** |

## 依存辺 — EU がいちばん強い

EUR-Lex は法令間の関係を **RDF の述語として明示的に持っている**ので、本文解析なしで辺が取れます。抽出しているのは「その法令がまだ適用されるか・どう適用されるかを変える」構造的述語だけ:

| 述語 (cdm:) | 辺 |
|---|---|
| `resource_legal_amends_resource_legal` | `:law.rel/amends` |
| `resource_legal_repeals_resource_legal` | `:law.rel/repeals` |
| `resource_legal_implicitly_repeals_resource_legal` | `:law.rel/implicitly-repeals` |
| `resource_legal_based_on_resource_legal` | `:law.rel/based-on` — 法的根拠。実装規則→基本規則がこれ |
| `resource_legal_completes_resource_legal` | `:law.rel/completes` |
| `resource_legal_corrects_resource_legal` | `:law.rel/corrects` |
| `resource_legal_codified_version` | `:law.rel/codified-as` |

**`cdm:work_cites_work` は wave-1 では取っていません。** 上の全述語を合わせたより 1 桁大きく、しかも「引用」であって「依存」ではないためです。取らなかったことを記録として残しています——不在から推測させないために。

## 実装上わかったこと（同じ罠を踏まないために）

- **OFFSET ページングは使えない。** Cellar の Virtuoso は OFFSET 10000 以降で HTTP 500 を返します（実測 2026-08-04: page 0,1 は成功し page 2 で落ちる）。キーセットページング（`ORDER BY ?k` + `FILTER(str(?k) > "直前の値")`）に切り替えてあります。辺のクエリは `CONCAT(?from," ",?to)` を複合ソートキーにしています——`?from` だけで区切ると、ある法令の辺リストがページ境界をまたいだ時に取りこぼします。
- **`Accept-Language` は必須。** 付けないと Cellar は全 CELEX に 400 を返します（実測: 1114/1114 失敗）。
- **`Accept` も 1 つでは足りない。** 古い指令は XHTML 版を持たず `text/html` しか無く、その時 Cellar は 406 ではなく **404** を返します。`(メディア型 × 言語)` の連鎖にして初めて 1,114 件全部が取れました（XHTML のみでは 780 件で頭打ち）。取れた本文が実際に何語の何形式かは `.meta` に記録してあり、推測させません。

## ライセンス

**Commission Decision 2011/833/EU** — 欧州委員会文書の再利用は出典表示のもとで許可。**Tier-A**。

## 再取得 / 再生成

```bash
nbb --classpath bin bin/fetch.cljk --pool 5   # SPARQL ページ + 指令本文（ページ単位で再開可）
nbb --classpath bin bin/index.cljk            # index/ と raw/source-catalog.edn
```
