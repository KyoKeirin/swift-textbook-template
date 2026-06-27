# AI質問ログ：第2章 地図アプリの基本

## 使用した生成AIツール

Gemini

## 質問と回答の記録

### Q1

**質問：**
`@AppStorage` と SwiftData は、何が違うの？

**AIの回答の要点：**
`@AppStorage` は、ユーザー名や表示設定のような小さな設定値を保存するのに向いている。内部的には `UserDefaults` を使い、アプリを閉じても値が残る。一方、SwiftData はメモのタイトル・本文・作成日・お気に入り状態のように、構造を持った複数のデータを保存・検索・削除するのに向いている。このアプリでは `userName` と `sortByFavorite` を `@AppStorage` で保存し、`Memo` の一覧は SwiftData で保存している。

**自分の理解：**
地図を表示するだけでなく、「どこを中心に見せるか」「どれくらい拡大するか」を決めるために `MapCameraPosition` と `MKCoordinateRegion` を使うと理解した。

### Q2

**質問：**
`ForEach(filteredLandmarks)` の中で `Marker` を作っている理由は？

**AIの回答の要点：**
`Identifiable` はデータを一意に区別するための仕組みである。`Landmark` に `id = UUID()` があるので、SwiftUI は各ランドマークを別々のデータとして認識できる。`ForEach(filteredLandmarks)` を使うことで、配列に入っている観光スポットの数だけ `Marker` を地図上に表示できる。

**自分の理解：**
地図上のマーカーを一つずつ手で書くのではなく、配列のデータをもとに自動で表示していると理解した。

### Q3

**質問：**
`LocationManager`、`UserAnnotation()`、`MKLocalSearch` はそれぞれどんな役割なの？

**AIの回答の要点：**
`LocationManager` は `CLLocationManager` を使ってユーザーの現在地と位置情報の許可状態を管理する。`UserAnnotation()` は地図上に現在地を表示するための SwiftUI MapKit の部品である。

**自分の理解：**

現在地アプリでは、まず位置情報の許可を取り、現在地が取得できたら地図の中心をその場所に移動し、さらにその周辺を検索する流れになっていると理解した。

## 今日の質問を振り返って

地図アプリでは、表示する場所を決める仕組み、データからマーカーを作る仕組み、現在地と周辺検索の仕組みを分けて質問できたのがよかった。
