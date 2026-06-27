# AI質問ログ：第4章 データの永続化

## 使用した生成AIツール

Gemini

## 質問と回答の記録

### Q1

**質問：**
`@AppStorage` と SwiftData は、何が違うの？

**AIの回答の要点：**
`@AppStorage` は、ユーザー名や表示設定のような小さな設定値を保存するのに向いている。内部的には `UserDefaults` を使い、アプリを閉じても値が残る。一方、SwiftData はメモのタイトル・本文・作成日・お気に入り状態のように、構造を持った複数のデータを保存・検索・削除するのに向いている。このアプリでは `userName` と `sortByFavorite` を `@AppStorage` で保存し、`Memo` の一覧は SwiftData で保存している。

**自分の理解：**
簡単な設定は `@AppStorage`、アプリの中心になるデータは SwiftData で保存すると考えると分かりやすいと思いました。

### Q2

**質問：**
SwiftData の `@Model`、`@Query`、`modelContext.insert`、`modelContext.delete` はそれぞれ何をしている？

**AIの回答の要点：**
`@Model` は SwiftData に保存できるデータモデルを定義するためのマクロである。このコードでは `Memo` クラスが保存対象になっている。`@Query` は SwiftData に保存されている `Memo` を自動で取得し、変更があれば画面にも反映する。`modelContext.insert(memo)` は新しいメモを保存し、`modelContext.delete(memo)` は指定したメモを削除する。`@main` の App ファイルに `.modelContainer(for: Memo.self)` を設定することで、アプリ全体で `Memo` を保存できるようになる。

**自分の理解：**
SwiftData では、モデルを作る、一覧を取得する、追加する、削除する、という流れがかなり少ないコードで書けると分かりました。

### Q3

**質問：**
`MemoEditView` で `@Bindable var memo: Memo` を使っている理由は？

**AIの回答の要点：**
`@Bindable` を使うと、SwiftData の `Memo` オブジェクトのプロパティを `TextField` や `Toggle` に直接バインディングできる。そのため、`TextField("タイトル", text: $memo.title)` のように書くと、入力内容が `memo.title` に反映される。

**自分の理解：**

編集画面では `@Bindable` のおかげで、フォームの入力と SwiftData のデータが直接つながっていると理解しました。

## 今日の質問を振り返って

設定の保存とメモデータの保存を分けて質問できたのがよかった。
