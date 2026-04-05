---
title: "Power Automate でよくあるエラー5選とその対処法"
emoji: "⚡"
type: "tech"
topics: ["PowerAutomate", "PowerPlatform", "Microsoft", "ローコード", "トラブルシューティング"]
published: false
---

## はじめに

こんにちは、竹内（[@htake1126](https://twitter.com/htake1126)）です。

普段は Microsoft で Power Platform（Power Apps / Power Automate / Copilot Studio）のサポートエンジニアとして、法人のお客様からの技術的なお問い合わせに対応しています。

本記事では、Power Automate のサポート対応を通じてよく目にするエラーと、その対処法をまとめました。フロー開発中や運用中に「あれ？」と思ったときの参考にしていただければ幸いです。

> **注意:** 本記事は個人の見解・学習メモであり、所属組織の公式見解ではありません。正確な情報は公式ドキュメントをご参照ください。サポートが必要な場合は公式チャネルからお問い合わせください。

---

## 1. 「The provided condition '' for field 'xxx' is not supported.」

### どんなエラー？

条件（Condition）アクションやフィルタークエリで、サポートされていない書き方をした場合に発生します。

### よくある原因

- OData フィルター構文が正しくない（例: `eq` の前後のスペース不足、文字列をシングルクォートで囲んでいない）
- Dataverse のフィルター可能でない列をフィルター条件に指定している
- 動的なコンテンツを式の中で正しく参照できていない

### 対処法

1. **フィルタークエリの構文を確認する**
   ```
   ❌ statuscode eq 1
   ✅ statuscode eq 1     （スペースの全半角に注意）
   ```
   文字列の場合はシングルクォートで囲みます。
   ```
   name eq 'テスト'
   ```

2. **フィルター可能な列か確認する**
   Dataverse のテーブル定義で、対象の列が「フィルター可能」になっているか確認してください。

3. **式エディタで動的コンテンツが正しく展開されているか確認する**
   実行履歴から、実際に送信されたフィルター文字列を確認するのが最も確実です。

### 参考

- [OData フィルター構文 - Microsoft Learn](https://learn.microsoft.com/ja-jp/power-automate/dataverse/filtering)

---

## 2. 「ActionFailed / The workflow action 'xxx' failed with error code 'Forbidden'.」

### どんなエラー？

フロー内のアクションがアクセス権限不足で失敗した場合に発生します。特に Dataverse、SharePoint、Exchange Online 関連で頻出します。

### よくある原因

- **接続（Connection）のユーザーに必要なアクセス権がない**
  - 例: Dataverse の行を更新しようとしたが、対象テーブルへの書き込み権限がない
- **共有メールボックスへのアクセスが許可されていない**
- **SharePoint サイトへのアクセス権が不足している**
- **接続が期限切れ・無効になっている**

### 対処法

1. **フローの接続を確認する**
   Power Automate ポータル → 対象フロー → 「編集」→ 各アクションの接続が正しいユーザーで認証されているか確認

2. **実行ユーザーの権限を確認する**
   - Dataverse: セキュリティロールで対象テーブルへの適切な権限があるか
   - SharePoint: サイトメンバー以上の権限があるか
   - Exchange: メールボックスへのアクセスが許可されているか

3. **接続を再認証する**
   接続が期限切れの場合、接続を削除して再作成することで解消する場合があります。

### 参考

- [接続の管理 - Microsoft Learn](https://learn.microsoft.com/ja-jp/power-automate/add-manage-connections)

---

## 3. 「ExpressionEvaluationFailed / The template language expression 'xxx' cannot be evaluated.」

### どんなエラー？

式（Expression）の構文エラーや、式が評価できない場合に発生します。Power Automate を使い始めた方が最もつまずきやすいエラーです。

### よくある原因

- **関数名のスペルミス**（`toLower` を `tolower` と書くなど ── ただしこれは動く場合もあり紛らわしい）
- **引数の型が間違っている**（文字列を渡すべきところに整数を渡しているなど）
- **null 値を処理していない**（前のアクションの出力が null の場合に式が失敗する）
- **動的コンテンツの参照パスが間違っている**

### 対処法

1. **式を小さく分解してテストする**
   「作成」（Compose）アクションに式を入れてテスト実行し、期待通りの結果が返るか確認。

2. **null チェックを入れる**
   ```
   if(empty(triggerOutputs()?['body/value']), '既定値', triggerOutputs()?['body/value'])
   ```
   `coalesce()` 関数も便利です。
   ```
   coalesce(triggerOutputs()?['body/value'], '既定値')
   ```

3. **実行履歴で式の入出力を確認する**
   実行履歴の各アクションを展開すると、式に渡された実際の値と評価結果が確認できます。

### 参考

- [式関数のリファレンス - Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/logic-apps/workflow-definition-language-functions-reference)

---

## 4. 「Throttling / Rate limit exceeded. Retry after 'xxx' seconds.」

### どんなエラー？

API の呼び出し回数が制限を超えた場合に発生します。大量のデータを処理するフローや、短時間に多数のフローが同時実行される環境で頻発します。

### よくある原因

- **Apply to each 内で大量のAPI呼び出しを行っている**
- **短時間に同じコネクタへのリクエストが集中している**
- **Power Platform のリクエスト制限（API Request Limits）に達している**

### 対処法

1. **バッチ処理を活用する**
   1件ずつ処理するのではなく、可能であればバッチ操作（例: Dataverse の一括操作）を使用。

2. **Apply to each の並列度を調整する**
   Apply to each の設定 → 「コンカレンシー制御」をオンにし、並列度を `1` に下げることでスロットリングを回避。

3. **リトライポリシーを設定する**
   アクションの設定 → 「リトライポリシー」で、指数バックオフ（Exponential）を設定すると、自動的にリトライ間隔を広げてくれます。

4. **フローの実行タイミングを分散する**
   スケジュールトリガーの場合、毎時0分に集中しないようオフセットを設ける。

### 参考

- [Power Platform の要求の制限と割り当て - Microsoft Learn](https://learn.microsoft.com/ja-jp/power-platform/admin/api-request-limits-allocations)

---

## 5. 「ConnectionNotConfigured / The connection 'xxx' is not configured for this flow.」

### どんなエラー？

フローに設定されている接続が見つからない、または無効になっている場合に発生します。ソリューションでフローを別の環境にインポートした直後に頻発します。

### よくある原因

- **フローを別の環境にインポートした後、接続の再設定を行っていない**
- **接続を作成したユーザーが組織を離れた**
- **接続の認証トークンが期限切れになった**
- **異なるテナントの接続を参照している**

### 対処法

1. **フローの接続参照を再設定する**
   フローの編集画面を開き、エラーが出ているアクションの接続を選び直す。

2. **ソリューションインポート時に接続参照を正しく設定する**
   ソリューションのインポート時に「接続参照」の設定画面が表示されるので、ターゲット環境の有効な接続を選択。

3. **サービスプリンシパル接続の利用を検討する**
   個人ユーザーの接続ではなく、アプリケーションユーザー（サービスプリンシパル）の接続を使うことで、人の異動による接続切れを防げます。

### 参考

- [ソリューションでの接続参照の使用 - Microsoft Learn](https://learn.microsoft.com/ja-jp/power-apps/maker/data-platform/create-connection-reference)

---

## まとめ

| # | エラー | 原因の要約 | まず確認すべきこと |
|---|--------|-----------|-------------------|
| 1 | Condition not supported | フィルター構文の誤り | 実行履歴でクエリ文字列を確認 |
| 2 | Forbidden | 権限不足 | 接続ユーザーの権限を確認 |
| 3 | Expression cannot be evaluated | 式の構文エラー / null | Compose でテスト + null チェック |
| 4 | Rate limit exceeded | API 呼び出し過多 | 並列度を下げる + リトライ設定 |
| 5 | Connection not configured | 接続の未設定 / 期限切れ | 接続を再認証 / 再設定 |

Power Automate のエラーは、**実行履歴を確認する**ことで大半は原因を特定できます。エラーが出たら、まず対象アクションの実行履歴を展開して、入力・出力・エラーメッセージを確認する習慣をつけましょう。

---

*この記事が参考になったら「いいね」していただけると励みになります！*
