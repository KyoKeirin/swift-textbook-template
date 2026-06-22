# 第4章：データの永続化

> 執筆者：許 敬林
> 最終更新：2026-06-22

## この章で学ぶこと
この章では、@Modelによるデータの定義方法が勉強できる。さらに、@Query を使って取得したり、modelContext.insert()・modelContext.delete()を追加って追加・削除したりすることもできる。特に、@Query で取得したデータが変更されると、UI も自動更新ができる。

## 模範コードの全体像

```swift
// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

```

**このアプリは何をするものか：**

このアプリは SwiftData で作られた「メモアプリ」である。SwiftData を利用して、追加・編集・削除・保存することができる。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**
Memo というデータの仕組みを定めます。

**なぜこう書くのか：**
SwiftData に保存できるデータになれるため。

**もしこう書かなかったら：**
作り方が不明で、エラーになりやすい。

---

### データの追加・削除（modelContext）

```swift
@Environment(\.modelContext) private var modelContext
```

**何をしているか：**
SwiftData のデータを追加・削除・保存・管理するための道具である。

**なぜこう書くのか：**
SwiftData のデータ保存場所は、アプリ全体で用意されている。

**もしこう書かなかったら：**
エラーになる。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
```

**何をしているか：**
SwiftData に保存されている「Memo」というデータを取得する。

**なぜこう書くのか：**
SwiftUI の画面で、SwiftData のデータを自動で表示・更新される。

**もしこう書かなかったら：**
SwiftData に保存されたデータの操作ができない。

---

### @AppStorageによる設定保存

```swift
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
```

**何をしているか：**
アプリの設定を保存する。さらに値が変わったときに自動で保存される。

**なぜこう書くのか：**
ユーザーネームを保存するため。

**もしこう書かなかったら：**
画面を開いている間は値を持てるのですが、アプリを終了すると設定が消えてしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model final class Memo { ... }` |
| 例：`@Query` | データベースからデータを取得し、変更を自動で反映するプロパティラッパー | `@Query var memos: [Memo]` |
| `@Environment(\.modelContext)` | SwiftData の保存・削除に使う | `@Environment(\.modelContext) private var modelContext` |
| `@AppStorage("userName")` | アプリの設定を保存して、次回起動時も残る | `@AppStorage("userName") private var userName: String = ""` |
| `IndexSet` | リストでアイテムを削除した時、どの行が削除されたかを IndexSet が教えてくれる | `func deleteMemos(at offsets: IndexSet) {...}` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：@AppStorage を使って、ユーザー名を保存してみた。`@AppStorage("userName") private var userName: String = "電子太郎"`。
- 結果：「電子太郎のメモ帳」と表示された。
- わかったこと：`@AppStorage` に保存されたデータ、次回起動時も残る。

**実験2：**
- やったこと：`@AppStorage("userName") private var showDate: Bool = true` を追加し、SettingView に `Toggle("日付を表示する", isOn: $showDate)` を追加した。
- 結果：メモが追加された日付は表示される。
- わかったこと：`@AppStorage` を使えば、自分好きな設定項目も追加できる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   Swift 言語の @Query って何？
   
   **得られた理解：**
   SwiftData の データを監視し、変更があれば SwiftUI の UI を自動更新する。

2. **質問：**
   Swift 言語の @Environment って何？
   
   **得られた理解：**
   SwiftUI が用意している共有情報を View の中で受け取るための仕組みである。

3. **質問：**
   Swift 言語の @Bindable って何？
   
   
   **得られた理解：**
   SwiftData のモデルを編集画面で直接編集できるようにするために @Bindable が使われる。
   

## この章のまとめ

SwiftData と様々な APIなど を使って、データをどう管理するか学んだ。特に、@AppStorage を使うと、保存されたアプリの設定、次回起動時も残るのは重要である。
