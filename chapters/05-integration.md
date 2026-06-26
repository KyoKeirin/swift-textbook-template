# 第5章：機能統合の実践

> 執筆者：許 敬林
> 最終更新：2026-06-26

## この章で学ぶこと

この章では、第2〜4章で学んだ機能を一つのアプリに統合する方法を学ぶ。  
具体的には、写真を選択し、現在地の緯度・経度と一緒に保存して、地図と一覧の両方で確認できる「フォトマップ」アプリを作る。

## 模範コードの全体像

```swift
// ============================================
// 第5章：カメラ + 地図 + データ保存の統合アプリ
// ============================================
// 写真を撮影し、撮影場所を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
//   - NSCameraUsageDescription（実機の場合）
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}

```

**このアプリは何をするものか：**

このアプリは、ユーザーが写真を選び、タイトルとメモを入力して保存する。保存すると、その記録はSwiftDataに保存され、地図上には写真付きのピンとして表示される。また、一覧タブでは保存した記録を新しい順に見ることができる。

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**  
`PhotoRecord`は、1つの写真記録を表すデータモデルである。タイトル、メモ、緯度、経度、画像データ、作成日時を保存する。`@Model`を付けることで、このクラスのデータをSwiftDataで保存できるようになる。

**なぜこう書くのか：**  
地図に表示するためには緯度と経度が必要で、一覧に表示するためにはタイトルや日付が必要である。また、写真を表示するためには画像データも必要である。

**もしこう書かなかったら：**  
`@Model`がないとSwiftDataで保存できない。緯度と経度を保存しないと、地図上に場所を表示できない。`imageData`を保存しないと、アプリを閉じたあとに写真を再表示できない。

---

### タブ構成の設計

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}
```

**何をしているか：**  
`TabView`を使って、「マップ」と「一覧」の2つの画面を切り替えられるようにしている。`MapTab`では地図表示を行い、`ListTab`では保存した記録をリストで表示する。

**なぜこう書くのか：**  
地図で見たいときと、リストで整理して見たいときでは、画面の目的が違う。そのため、タブで画面を分けると使いやすい。`ContentView`はアプリ全体の入口として、細かい処理は`MapTab`や`ListTab`に分けている。

**もしこう書かなかったら：**  
1つの画面に地図と一覧を全部入れることになり、画面が見づらくなる。また、コードも長くなり、初心者には理解しにくくなる。

---

### カメラと位置情報の連携

```swift
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}
```

**何をしているか：**
写真を`PhotosPicker`で選び、現在地を`LocationManager`で取得する。さらに、保存すると、写真・タイトル・位置情報を１つの`PhotoRecord`にまとめる。

**なぜこう書くのか：**
位置情報と画面の処理を分けることで、コードが見やすくなる。また、写真と位置情報を同時に保存できるようにしている。

**もしこう書かなかったら：**
位置情報が保存されず、写真を地図に表示できない。写真データを保存しないと、一覧や詳細画面に写真を表示できない。

---

### SwiftDataでの画像保存

```swift
func saveRecord() {
    guard let location = locationManager.currentLocation else { return }

    let record = PhotoRecord(
        title: title,
        memo: memo,
        latitude: location.latitude,
        longitude: location.longitude,
        imageData: selectedImageData
    )
    modelContext.insert(record)
    dismiss()
}
```

**何をしているか：**  
保存ボタンを押したときに、現在地があるか確認し、新しい`PhotoRecord`を作ってSwiftDataに追加している。`modelContext.insert(record)`が、データを保存対象に追加する処理である。保存後は`dismiss()`で追加画面を閉じる。

**なぜこう書くのか：**  
タイトル、メモ、位置情報、画像データを1つの`PhotoRecord`にまとめて保存することで、あとから地図・一覧・詳細画面で同じデータを使える。`guard`を使うことで、位置情報がない場合は安全に処理を止められる。

**もしこう書かなかったら：**  
保存処理が実行されず、地図や一覧に新しい記録が表示されない。`guard`がないと、位置情報がない状態で保存しようとしてエラーや不完全なデータの原因になる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TabView` | 複数のビューをタブで切り替えるコンポーネント | `TabView { ... }.tabViewStyle(.page)` |
| 例：`CLLocationManager` | GPS位置情報を取得するAPIManager | `let location = manager.location?.coordinate` |
| `Annotation` | 地図上に独自の表示を置く | `Annotation(record.title,  coordinate:record.coordinate) { ... }` |
| `UserAnnotation` | 地図上にユーザーの現在地を表示する | `UserAnnotation()` |
| `sheet` | 別画面をモーダル表示する | `.sheet(isPresented: $isShowingAddSheet {...}` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：写真を選ばずに保存する**

- やったこと：タイトルだけ入力し、写真を選ばずに保存できるか確認した。
- 結果：写真がなくても保存できる。地図上では写真アイコンが表示される。
- わかったこと：`imageData`は`Data?`なので、写真がない場合は`nil`でもよい。オプショナル型にすると、データがない場合にも対応できる。

**実験2：タイトルを空のまま保存しようとする**

- やったこと：タイトルを入力しないで保存ボタンを見た。
- 結果：保存ボタンが無効になっていた。
- わかったこと：`.disabled(title.isEmpty || locationManager.currentLocation == nil)`によって、必要な情報がないと保存できないようにしている。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**  
   `@Model`と`@Query`は何が違う？  
   **得られた理解：**  
   `@Model`は保存するデータの形を決めるもの、`@Query`は保存されたデータを画面で読み込むものだと理解した。

2. **質問：**  
   写真を`Image`ではなく`Data`で保存する理由は何？  
   **得られた理解：**  
   `Image`は画面表示用で、保存には向いていない。写真をあとから復元するためには、元のバイナリデータである`Data`として保存する必要がある。

3. **質問：**  
   なぜ位置情報は`LocationManager`という別のクラスに分けるの？  
   **得られた理解：**  
   位置情報の取得は許可や更新処理があり複雑なので、ビューから分けるとコードが読みやすくなる。ビューは画面表示に集中できる。

## この章のまとめ

この章で一番大事だと思ったことは、アプリは1つの機能だけではなく、複数の機能を組み合わせて作られるということである。
