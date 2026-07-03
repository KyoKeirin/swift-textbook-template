# 第8章：ウィジェット

> 執筆者：許 敬林
> 最終更新：2026-07-02

## この章で学ぶこと

この章では、WidgetKitを使ってホーム画面に表示できる「今日の名言」ウィジェットを作る方法を学ぶ。メインアプリで使う名言データをウィジェット側でも使い、`TimelineProvider`で更新タイミングを決め、`widgetFamily`によって小サイズ・中サイズのレイアウトを切り替える流れを理解する。

## 模範コードの全体像

```swift
// ============================================
// 第8章：ウィジェットを作る
// ============================================
// 今日の名言をホーム画面に表示するウィジェットです。
// メインアプリとウィジェットの両方のコードを含みます。
//
// 【セットアップ手順】
// 1. Xcodeで File → New → Target → Widget Extension を選択
// 2. 「Include Configuration App Intent」のチェックを外す
// 3. Widget Extensionの名前を「QuoteWidget」にする
// 4. メインアプリとウィジェットで App Group を設定する
//    （Signing & Capabilities → App Groups）
// ============================================

// ============================================
// ■ メインアプリ側のコード（ContentView.swift）
// ============================================

import SwiftUI

// MARK: - 名言データ（アプリとウィジェットで共有）

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}

// MARK: - メインアプリのContentView

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}


// ============================================
// ■ ウィジェット側のコード（QuoteWidget.swift）
// ============================================
// ※ Widget Extension ターゲット内のファイルに記述します。
// ※ QuoteStore は共有ファイルとして両ターゲットに追加するか、
//    同じコードをウィジェット側にもコピーしてください。
// ============================================

/*
import WidgetKit
import SwiftUI

// MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
*/
```

**このアプリは何をするものか：**

このアプリは、名言の一覧を表示するメインアプリと、今日の名言をホーム画面に表示するウィジェットで構成されている。

## コードの詳細解説

### TimelineProviderの仕組み

```swift
struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}
```

**何をしているか：**
`QuoteProvider` は、ウィジェットに表示するデータと更新タイミングを決める役割を持っている。

**なぜこう書くのか：**
WidgetKitのウィジェットは、アプリの画面のように常に動き続けるものではない。そのため、あらかじめ「この時刻にはこの内容を表示する」という `Timeline` を渡す必要がある。

**もしこう書かなかったら：**
`getTimeline` を正しく実装しないと、ウィジェットがいつ更新されるべきか分からなくなる。`.after(tomorrow)` を指定しなければ、毎日名言を切り替える意図が伝わりにくくなる。

---

### TimelineEntryとウィジェットビュー

```swift
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }
}
```

**何をしているか：**
`QuoteEntry` は、ウィジェットがある時点で表示するデータをまとめた構造体である。`TimelineEntry` に必要な `date` と、今回表示したい `quote` を持っている。

**なぜこう書くのか：**
ウィジェットは時間ごとに表示内容が変わるため、「いつ」「何を表示するか」を1つの `TimelineEntry` にまとめる必要がある。

**もしこう書かなかったら：**
`TimelineEntry` がなければ、WidgetKitが表示すべきデータを時間軸で管理できない。

---

### ウィジェットサイズごとのレイアウト

```swift
@Environment(\.widgetFamily) var family

var body: some View {
    switch family {
    case .systemSmall:
        smallWidget
    case .systemMedium:
        mediumWidget
    default:
        mediumWidget
    }
}

var smallWidget: some View {
    VStack(spacing: 4) {
        Image(systemName: "quote.opening")
            .font(.caption)
            .foregroundStyle(.blue)

        Text(entry.quote.text)
            .font(.caption)
            .bold()
            .multilineTextAlignment(.center)
            .lineLimit(3)

        Text(entry.quote.author)
            .font(.caption2)
            .foregroundStyle(.secondary)
    }
    .padding(12)
}

var mediumWidget: some View {
    HStack(spacing: 16) {
        Image(systemName: "quote.opening")
            .font(.title)
            .foregroundStyle(.blue)

        VStack(alignment: .leading, spacing: 4) {
            Text("今日の名言")
                .font(.caption2)
                .foregroundStyle(.secondary)

            Text(entry.quote.text)
                .font(.subheadline)
                .bold()

            Text("— \(entry.quote.author)")
                .font(.caption)
                .foregroundStyle(.secondary)
        }

        Spacer()
    }
    .padding()
}
```

**何をしているか：**
`@Environment(\.widgetFamily)` によって、現在のウィジェットサイズを取得している。

**なぜこう書くのか：**
ウィジェットはサイズによって使える面積が大きく違う。小サイズでは情報を詰め込みすぎると読みにくいため、フォントを小さめにして `lineLimit(3)` で最大行数を制限している。

**もしこう書かなかったら：**
サイズを考慮せず同じレイアウトだけにすると、小サイズで文字が切れたり、中サイズで余白が多すぎたりする。

---

### メインアプリとの連携

```swift
struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}
```

**何をしているか：**
メインアプリでは、`QuoteStore.todaysQuote()` で今日の名言を取得し、上部にカードのような見た目で表示している。

**なぜこう書くのか：**
メインアプリとウィジェットで同じデータを使うと、内容の一貫性を保てる。

**もしこう書かなかったら：**
メインアプリとウィジェットで別々の名言データを持つと、片方だけ更新されて内容がずれる可能性がある。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TimelineProvider` | ウィジェットを更新するタイミングとコンテンツを定義 | `struct QuoteProvider: TimelineProvider { ... }` |
| 例：`@main` + `WidgetConfiguration` | ウィジェットのエントリーポイント | `@main struct QuoteWidget: Widget { ... }` |
| `@Environment(\.widgetFamily)` | 現在のウィジェットサイズを取得するプロパティラッパー | `@Environment(\.widgetFamily) var family` |
| `TimelineEntry` | ウィジェットが特定の時刻に表示するデータを表すプロトコル | `struct QuoteEntry: TimelineEntry { let date: Date ... }` |
| `StaticConfiguration` | ユーザー設定なしのウィジェットを定義する | `StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in ... }` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：`smallWidget` の `Text(entry.quote.text)` にある `.lineLimit(3)` を外した場合を考えた。
- 結果：短い名言なら問題ないが、長い名言では作者名まで表示するスペースが足りなくなりそうだと分かった。
- わかったこと：ウィジェットは画面が小さいため、通常のアプリ画面よりも文字量の制限が重要になる。

**実験2：**
- やったこと：`.supportedFamilies([.systemSmall, .systemMedium])` に `.systemLarge` を追加した場合を考えた。
- 結果：`switch family` の `default` で `mediumWidget` が表示されるため、大サイズでも中サイズ用レイアウトが使われる。
- わかったこと：対応サイズを増やすだけでは不十分で、大サイズ専用の `largeWidget` を作った方が見た目が自然になる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   `TimelineProvider` の `placeholder`、`getSnapshot`、`getTimeline` はそれぞれ何が違うの？
   
   **得られた理解：**
   `placeholder` は仮表示、`getSnapshot` はギャラリーやプレビュー、`getTimeline` は実際のホーム画面での表示と更新スケジュールに使われる。
   

2. **質問：**
   なぜ日付を使って名言を選ぶの？
   
   **得られた理解：**
   日付を使うと、同じ日なら同じ名言になり、日替わりの動きとして自然になる。

3. **質問：**
   メインアプリとウィジェットで同じ `QuoteStore` を使うには何に注意する必要なの？
   
   **得られた理解：**
   Widget Extensionはメインアプリとは別ターゲットなので、共有したいファイルを両方のターゲットに含める必要がある。共有データを保存する場合はApp Groupの設定も重要になる。

## この章のまとめ

この章で一番重要だと感じたのは、ウィジェットは普通のSwiftUI画面とは違い、「ビューを常に動かす」のではなく、「時間ごとの表示内容を `Timeline` として `WidgetKit` に渡す」仕組みで動くという点である。
