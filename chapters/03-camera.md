# 第3章：カメラの利用

> 執筆者：許 敬林
> 最終更新：2026-06-21

## この章で学ぶこと

写真ライブラリから画像を選択し、SwiftUIで表示する方法を学ぶ。また、フィルターで加工して実装することで、画像の扱い方や、写真編集など学ぶことができる。

## 模範コードの全体像
```swift
import SwiftUI
import PhotosUI
import CoreImage
import CoreImage.CIFilterBuiltins

// MARK: - フィルター定義

enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
    case chrome = "クローム"
    case fade = "フェード"
    case bloom = "ブルーム"

    var id: String { rawValue }

    func apply(to inputImage: CIImage, context: CIContext) -> CIImage? {
        switch self {
        case .original:
            return inputImage
        case .sepia:
            let filter = CIFilter.sepiaTone()
            filter.inputImage = inputImage
            filter.intensity = 0.8
            return filter.outputImage
        case .mono:
            let filter = CIFilter.photoEffectMono()
            filter.inputImage = inputImage
            return filter.outputImage
        case .chrome:
            let filter = CIFilter.photoEffectChrome()
            filter.inputImage = inputImage
            return filter.outputImage
        case .fade:
            let filter = CIFilter.photoEffectFade()
            filter.inputImage = inputImage
            return filter.outputImage
        case .bloom:
            let filter = CIFilter.bloom()
            filter.inputImage = inputImage
            filter.radius = 10
            filter.intensity = 0.8
            return filter.outputImage
        }
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var originalUIImage: UIImage?
    @State private var displayImage: Image?
    @State private var currentFilter: PhotoFilter = .original
    @State private var isSaving = false
    @State private var showSaveAlert = false
    @State private var saveMessage = ""

    private let context = CIContext()

    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                // 画像表示
                if let image = displayImage {
                    image
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .frame(maxHeight: 350)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                        .padding(.horizontal)
                } else {
                    placeholderView
                }

                // フィルター選択
                if originalUIImage != nil {
                    filterSelector
                }

                // ボタン群
                HStack(spacing: 16) {
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選ぶ", systemImage: "photo")
                    }
                    .buttonStyle(.bordered)

                    if displayImage != nil {
                        Button {
                            saveFilteredImage()
                        } label: {
                            Label("保存", systemImage: "square.and.arrow.down")
                        }
                        .buttonStyle(.borderedProminent)
                        .disabled(isSaving)
                    }
                }
                .padding()

                Spacer()
            }
            .navigationTitle("フォトフィルター")
            .onChange(of: selectedItem) { _, newItem in
                Task { await loadOriginalImage(from: newItem) }
            }
            .onChange(of: currentFilter) { _, _ in
                applyFilter()
            }
            .alert("保存結果", isPresented: $showSaveAlert) {
                Button("OK") {}
            } message: {
                Text(saveMessage)
            }
        }
    }

    // MARK: - プレースホルダー

    private var placeholderView: some View {
        RoundedRectangle(cornerRadius: 12)
            .fill(.gray.opacity(0.1))
            .frame(height: 300)
            .overlay {
                VStack(spacing: 8) {
                    Image(systemName: "camera.filters")
                        .font(.system(size: 48))
                        .foregroundStyle(.gray)
                    Text("写真を選んでフィルターを試そう")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .padding(.horizontal)
    }

    // MARK: - フィルター選択UI

    private var filterSelector: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 12) {
                ForEach(PhotoFilter.allCases) { filter in
                    VStack(spacing: 4) {
                        // フィルタープレビュー（サムネイル）
                        if let thumbnail = createThumbnail(filter: filter) {
                            Image(uiImage: thumbnail)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 60, height: 60)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                                .overlay(
                                    RoundedRectangle(cornerRadius: 8)
                                        .stroke(
                                            currentFilter == filter ? Color.blue : Color.clear,
                                            lineWidth: 3
                                        )
                                )
                        }

                        Text(filter.rawValue)
                            .font(.caption2)
                            .foregroundStyle(
                                currentFilter == filter ? .blue : .secondary
                            )
                    }
                    .onTapGesture {
                        currentFilter = filter
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 画像処理

    func loadOriginalImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                originalUIImage = uiImage
                currentFilter = .original
                displayImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像読み込みエラー: \(error)")
        }
    }

    func applyFilter() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return }

        guard let outputImage = currentFilter.apply(to: ciImage, context: context) else { return }

        if let cgImage = context.createCGImage(outputImage, from: ciImage.extent) {
            displayImage = Image(uiImage: UIImage(cgImage: cgImage))
        }
    }

    func createThumbnail(filter: PhotoFilter) -> UIImage? {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return nil }

        guard let output = filter.apply(to: ciImage, context: context) else { return nil }

        if let cgImage = context.createCGImage(output, from: ciImage.extent) {
            return UIImage(cgImage: cgImage)
        }
        return nil
    }

    func saveFilteredImage() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage),
              let output = currentFilter.apply(to: ciImage, context: context),
              let cgImage = context.createCGImage(output, from: ciImage.extent) else { return }

        let finalImage = UIImage(cgImage: cgImage)
        UIImageWriteToSavedPhotosAlbum(finalImage, nil, nil, nil)

        saveMessage = "写真を保存しました"
        showSaveAlert = true
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

ユーザーが写真を選び、フィルターを適用し、加工後の画像を保存できるようにするアプリである。

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
    PhotosPicker(selection: $selectedItem, matching: .images) {
        Label("写真を選ぶ", systemImage: "photo")
    }
    .buttonStyle(.bordered)
```

**何をしているか：**
画面には「写真を選ぶ」というボタンを追加して、そのボタンを押すと、iPhoneの写真ライブラリが開く。

**なぜこう書くのか：**
「PhotosPicker」は公式のライブラリで、セキュリティを確保しながら簡単に写真を選択することができる。

**もしこう書かなかったら：**
UIKit の PHPickerViewController とか UIImagePickerController も使えるが、PhotosPicker ほど便利ではない。

---

### 画像の非同期読み込み

```swift
    func loadOriginalImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                originalUIImage = uiImage
                currentFilter = .original
                displayImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像読み込みエラー: \(error)")
        }
    }
```

**何をしているか：**
PhotosPicker で選ばれた写真を読み込んで、画面に表示する処理である。

**なぜこう書くのか：**
写真の読み込みは時間がかかる可能性があるため。

**もしこう書かなかったら：**
「guard let」がないと、写真が選ばれていないまま処理されると、エラーになる。
「try await」がないと、写真の読み込み処理を正しく待てない。

---

### UIViewControllerRepresentableによるカメラ連携

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

```

**何をしているか：**
撮影した写真をSwiftUIに渡す。

**なぜこう書くのか：**
SwiftUI と UIKitをスムーズに連携させるため。

**もしこう書かなかったら：**
UIImagePickerControllerはUIKit の機能なので、こう書かないと SwiftUI で直接使いにくい。

---

### Coordinatorパターン

```swift
class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
```

**何をしているか：**
画像が選択されたときやキャンセルされた時の処理を担当している。

**なぜこう書くのか：**
UIImagePickerControllerDelegate は UIKit の機能であり、画像選択やキャンセルの結果を Delegate パターンで通知する仕組みになっている。

**もしこう書かなかったら：**
Coordinator がない場合、画像が選択されたことを受け取れない。

---


## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`PhotosPicker` | フォトライブラリから画像を選択するコンポーネント | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| 例：`UIImagePickerController` | カメラまたはフォトライブラリにアクセスするUIKitコンポーネント | `picker.sourceType = .camera` |
| `UIImagePickerControllerDelegate` | 写真選択や撮影結果を受け取るためのデリゲート | ` class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {...}` |
| `CIFilter` | 画像にフィルター効果を適用する Core Image の API | ` let filter = CIFilter.sepiaTone()`|
| `dissmiss()` | 表示中のカメラ画面や選択画面を閉じる処理 | `parent.dismiss()` |

## 自分の実験メモ

```
    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        print("キャンセルしました")
        parent.dismiss()
    }
```

**実験1：**
- やったこと：「imagePickerControllerDidCancel」という関数の「parent.dismiss()」の上に「print("キャンセルしました")」を追加した。
- 結果：閉じる時お知らせが出てくる
- わかったこと：Coordinatorに自分の好きな関数が入れる。

**実験2：**
- やったこと：「filter.intensity = 0.8」の値を「1.0」を変えた。
- 結果：かなり古い写真っぽくなった。
- わかったこと：値が高いほど、昔の写真みたいになる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   Swift の　「UIViewControllerRepresentable」って何？
   
   **得られた理解：**
   UIKit の 「ViewController」を SwiftUI の中で使えるようにする橋渡しみたいなものである。
   
3. **質問：**
   Swift の　「CaseIterable」って何？
   
   **得られた理解：**
   enum に書いた選択しを Swift が自動で一覧化してくれる。

4. **質問：**
   Swift の　「CIImage」って何？
   
   **得られた理解：**
   「CIImage」はフィルター処理用の画像で、「UIImage」は表示用の画像である。

## この章のまとめ
SwiftUI で写真を選び、Core Image でフィルターをかけ、プレビュー表示し、加工後の写真を保存する方法を学んだ。
