# open-kagaku

open-cg-cadの姉妹プロジェクト(プラグイン的位置づけ)。OTC(市販)医薬品
有効成分・漢方処方の参考データベースと、風邪薬(漢方+化学薬品ハイブリッド)
配合の一次スクリーニングツール、およびそれらをJSON APIと日英併記のWeb UI
として提供するHTTPサーバー。教育・研究用途。

**公開URL**: https://easy-web.tokyo/open-kagaku/ (systemd常駐、自動起動有効)

## ⚠️ 最重要制約

**本リポジトリは医療機器・医薬品ソフトウェアではなく、教育・研究・
設計検討用の参考ツールに過ぎない。実際の服薬判断・処方・製剤設計には
薬剤師・医師等の有資格者によるレビューが必須であり、本ツールの出力を
根拠に服薬量を決定してはならない。**

## スコープ(安全性への配慮)

開発着手時、「猛毒から毒性を取り除く機能」を含む提案があったが、化学
兵器・違法薬物の無毒化/精製手順の生成に転用され得るデュアルユースリスク
があるため**実装しないことをユーザーと合意の上で確認済み**(2026-08-25)。
この機能は今後も機能・将来課題(ストレッチゴール)のいずれとしても扱わず、
**安全上の理由により検討の上で見送った**ことを明記する。
本リポジトリが扱うのは以下の安全な範囲のみ:

1. **`molecular_formula`**: 市販薬有効成分名(英語)→分子式の参考データベース
   (例: acetaminophen → C8H9NO2)。**正直な開示**: 「あらゆる薬品名」を
   解決できる汎用パーサーではなく、限定的な収録リストのみ。
2. **`otc_ingredient`**: OTC医薬品有効成分DB。薬効分類(解熱鎮痛薬/
   抗ヒスタミン薬/鎮咳薬/去痰薬/鼻閉改善薬/中枢神経刺激薬)+成人1日
   最大量の参考値を収録。
3. **`kampo_formula`**: 漢方処方の参考DB。現在5処方
   (葛根湯・麻黄湯・小青竜湯・麦門冬湯・麻杏甘石湯)に加え、実在する
   ハイブリッド処方2件(**柴朴湯**=小柴胡湯+半夏厚朴湯の合方、
   **清肺湯**=16種の生薬から成る処方。いずれも喉・気管支の両方に
   伝統的に用いられ、Google検索による事実確認済み)を収録。
   - `glycyrrhiza_overlap_warning`: 2処方以上を併用した際、いずれも
     甘草(グリチルリチン)を含む場合に警告。**正直な開示**: リスクは
     肝臓障害ではなく**偽アルドステロン症**(高血圧・低カリウム血症等)。
   - `recommend_combined_formula`: 複数の適応分類(例: 喉+気管支)を
     要求された際、それらを単独でカバーする既存の単一処方を推奨する
     機能。存在しない組み合わせ処方を新規に創作することはせず、
     データベースに無ければ正直に「見つからなかった」旨を返す。
4. **`cold_medicine_formulation`**: 化学成分(OTC)+任意の漢方処方を
   組み合わせた「配合案」を構成し、同一薬効分類の重複を検出する
   一次スクリーニング機能。
5. **`vitamin_supplement`**: ビタミン・ミネラルの過剰摂取参考DB
   (ナイアシン/B6/葉酸/プレフォームドビタミンA/サプリメントβカロテン/
   カフェイン)。耐容上限量(UL)+成分ごとの文献的に裏付けられる具体的
   リスクを収録。**正直な開示**: 「ビタミンB全般ががんの原因」という
   一般化は文献的支持が無く不正確。実際に文献(CARET/ATBC試験)で
   裏付けられるのは**サプリメントの高用量βカロテン摂取(特に喫煙者)と
   肺がんリスク**の関連。
6. **HTTPサーバー(`server`クレート、RPoemベース、既定ポート4702)**:
   上記全機能をJSON APIとして公開し、日英併記のWeb UI(`index.html`)
   から呼び出せる。入力欄の区切り文字を柔軟化しており、半角/全角の
   カンマ・ピリオド・ハイフン・アンダーライン・スペース等を区切り文字
   として受け付ける(ただし適応分類欄は`cold_flu_early_stage`のように
   アンダーラインを値の一部として使うため対象外)。
7. **open-cg-cadとの相互リンク**: open-english⇄open-cg-cadと同型の
   localStorageベースの双方向ナビゲーションリンクを実装。
8. **実デプロイ**: `https://easy-web.tokyo/open-kagaku/`へsystemd
   サービス(`installer/open-kagaku.service`、ポート8106)として公開済み。
   `domains.toml`に`path_prefix = "/open-kagaku"`を設定。

詳細は[`CLAUDE.md`](CLAUDE.md)を参照。

## ビルド・テスト

```
cargo build --workspace --release
cargo test --workspace --release
cargo clippy --workspace --all-targets --release -- -D warnings
```

## サーバー起動

```
cargo run --release -p server
# 既定で 127.0.0.1:4702 で待受(環境変数 OPEN_KAGAKU_SERVER_BIND で変更可)
```

## アーキテクチャ

Rustワークスペース。`crates/kagaku-core`にプラットフォーム非依存の
純粋ロジック、`server`にRPoemベースのHTTPサーバー+日英併記Web UIを実装。

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, noch nicht von Muttersprachlern geprüft)
- **REST wird abgeschafft. Neue APIs werden nicht als REST gebaut.** APIs, die aus dem Browser oder mit `curl` aufgerufen werden, nutzen **Connect-Unary-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body); zwischen Diensten gilt **gRPC / gRPC-web** (wie bei der Admin-API von RPoem, `open_runo.admin.v1.Admin`). Streaming erst, wenn es wirklich nötig ist.
- **Bestehende REST-Endpunkte werden einzeln migriert, sobald man sie anfasst.** Nach der Migration wird die alte REST-Route entfernt. Bis dahin bleiben sie bestehen, es werden aber **keine neuen hinzugefügt**.
- **JSON wird auf RS-JSON vereinheitlicht**: Lesen mit `from_str_secure` / `parse_secure` (Grenzen für Größe, Tiefe und doppelte Schlüssel), Schreiben mit `rust_json::fast`. `prost` kommt nicht in Browser-WASM (die Datei würde größer).
- **Ausnahmen (Formate, die ein fremder Standard vorgibt, nicht unsere eigene API)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Dritten (z. B. GitHub), Health-Checks, statische Dateien, OTLP usw. Diese nehmen schlichtes HTTP entgegen, wie es der Standard verlangt.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines Repositorys diese Richtlinie schon befolgt, hängt vom jeweiligen Repository ab. Dieser Abschnitt hält die Richtlinie fest; er bedeutet nicht, dass der Code bereits geändert wurde.

### Русский (машинный перевод, носителями языка ещё не проверен)
- **REST полностью отменяется. Новые API не создаются как REST.** API, вызываемые из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело — JSON); между сервисами — **gRPC / gRPC-web** (так же, как административный API RPoem, `open_runo.admin.v1.Admin`). Потоковая передача — только когда она действительно нужна.
- **Существующие REST-эндпоинты переносятся по одному, когда до них доходят руки.** После переноса старый REST-маршрут удаляется. До завершения переноса они остаются, но **новые не добавляются**.
- **JSON унифицируется на RS-JSON**: чтение через `from_str_secure` / `parse_secure` (ограничения на размер, глубину и повторяющиеся ключи), запись через `rust_json::fast`. `prost` не добавляется в браузерный WASM (размер вырос бы).
- **Исключения (форматы, заданные чужим стандартом, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и т. п. Они принимают обычный HTTP, как требует стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Этот раздел фиксирует политику и не означает, что код уже изменён.
