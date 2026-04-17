# 第1章：WebAPIの基本

> 執筆者：25CM0110
> 最終更新：2026-04-17

## この章で学ぶこと　（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

この章では、インターネット上のサービス（API）からデータを取得して、アプリ内に表示する方法を学。具体的にはiTunes Search APIを使って曲を検索し、その結果をリスト表示するアプリである。ただ、iTunesに登録されている曲しか表示できないです。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る

// ============================================
// 第1章（基本）：iTunes Search APIで音楽を検索するアプリ
// ============================================
// このアプリは、iTunes Search APIを使って
// 音楽（曲）を検索し、結果をリスト表示します。
// APIキーは不要で、すぐに動かすことができます。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**
iTunes Search APIを使って、曲を検索するアプリである。
<img width="150" height="326" alt="Simulator Screenshot - iPhone 17 Pro - 2026-04-15 at 15 02 23" src="https://github.com/user-attachments/assets/cc52507b-2d96-4559-a28f-c8d00bcc287a" />


## コードの詳細解説

### データモデル（Codable構造体）

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
（この部分が果たしている役割を説明する）

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

---

### API通信の処理

```swift
// 該当部分のコードを抜粋して貼る
"https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
```

**何をしているか：**
term=\(encodedText): キーワードの入力
media=music: 音楽しか現れない
country=jp: 日本の音楽のみ
limit=25: 表示されるアイテムは25本まで

**なぜこう書くのか：**
エンジニアにとって、使いやすくなる

**もしこう書かなかったら：**
こう制限しなかったら、ほかの不要なアイテムが現れるわげである

---

### ビューの構成

```swift
　　　　　　　VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)
                    
                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)
                
                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
```

**何をしているか：**
レイアウトしている

**なぜこう書くのか：**
「VStack」と「HStack」を使うと、キレイなレイアウトができれる

**もしこう書かなかったら：**
メチャクチャになってしまう

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| `Itentifiable` | 安定した同一性を持つ実体の値をインスタンスに持つ型のクラス | `let trackId: Int    var id: Int { trackId }` |
| `JSONDecoder` | JSONオブジェクトからデータタイプに変換する | `let response = try JSONDecoder().decode(SearchResponse.self, from: data)` |
| `AsyncImage` | 外部からの画像データを非同期で取得表示する方法を知りたい | AsyncImage(url: URL(string: song.artworkUrl100)) |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：「country=ja_jp」を「en_us」に変換しにみた
- 結果：英語版になった
- わかったこと：APIをどうやって使うかわかった

**実験2：**
- やったこと：他のAPIを導入しようとしてみた
- 結果：全部失敗しちゃった
- わかったこと：「API」というのは、ほぼ「APIキー」というものが必要するべきだ

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ
APIの使い方が勉強になった。しかも、Codable, Identifiableの役割も深く理解して、APIに合わせて、変数名を作るのが重要だと感じた。

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
