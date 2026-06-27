# AI質問ログ：第3章 カメラの利用

## 使用した生成AIツール

Gemini

## 質問と回答の記録

### Q1

**質問：**
`PhotosPicker`、`PhotosPickerItem` はそれぞれ何をしている？

**AIの回答の要点：**
`PhotosPicker` はフォトライブラリから写真を選択するための SwiftUI の部品である。選択された写真は `PhotosPickerItem` として `selectedItem` に入る。

**自分の理解：**
写真を選ぶだけではすぐに画面に出せるわけではなく、`PhotosPickerItem` から画像データを読み込んで、`UIImage`、`Image` の順に変換していると理解した。

### Q2

**質問：**
`Coordinator` の役割は？

**AIの回答の要点：**
`makeUIViewController` でカメラ用の `UIImagePickerController` を作り、`Coordinator` が撮影完了やキャンセルなどのイベントを受け取る。

**自分の理解：**
`Coordinator` は UIKit 側の delegate の処理を SwiftUI 側の状態更新につなげる役割だと理解した。

### Q3

**質問：**
写真にフィルターをかけるコードでは、`CoreImage`、`CIImage`、`CIFilter`、`CIContext` はどんな役割を持っている？

**AIの回答の要点：**
CoreImage` は画像加工を行うためのフレームワークである。`CIImage` は CoreImage で処理するための画像データ、`CIFilter` はセピア・モノクロ・ブルームなどの効果、`CIContext` は処理結果を実際の画像として書き出すために使う。

**自分の理解：**

画像加工では、SwiftUI の `Image` を直接加工するのではなく、`UIImage` から `CIImage` に変換して CoreImage のフィルターを通すと理解した。

## 今日の質問を振り返って

写真アプリでは、写真を選ぶ処理、カメラを開く処理、フィルターをかけて保存する処理を分けて質問できたのがよかった。
