# 第7章：センサーの活用

> 執筆者：許 敬林
> 最終更新：2026-06-26

## この章で学ぶこと

この章では、iPhoneやiPadに入っているセンサーを、SwiftUIアプリから使う方法を学ぶ。今回の模範コードでは、`CoreMotion`を使って端末の傾きデータを取得し、水平器（水準器）のように画面へ表示する。

## 模範コードの全体像

```swift
// ============================================
// 第7章（基本）：加速度センサーで動く水平器アプリ
// ============================================
// CoreMotionを使って端末の傾きをリアルタイムで取得し、
// 水平器（水準器）として表示するアプリです。
//
// 【注意】シミュレータではセンサーが使えません。
//         実機（iPhone / iPad）でテストしてください。
// ============================================

import SwiftUI
import CoreMotion

// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool = false

    func startUpdates() {
        guard motionManager.isDeviceMotionAvailable else {
            isAvailable = false
            return
        }

        isAvailable = true
        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var motionManager = MotionManager()

    var body: some View {
        NavigationStack {
            if motionManager.isAvailable {
                VStack(spacing: 30) {
                    // 水平器の円
                    LevelIndicator(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll
                    )

                    // 数値表示
                    DataDisplay(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll,
                        yaw: motionManager.yaw
                    )
                }
                .padding()
                .navigationTitle("水平器")
            } else {
                ContentUnavailableView(
                    "センサーが利用できません",
                    systemImage: "iphone.slash",
                    description: Text("このアプリは実機（iPhone）で動作します。\nシミュレータではセンサーが使えません。")
                )
            }
        }
        .onAppear {
            motionManager.startUpdates()
        }
        .onDisappear {
            motionManager.stopUpdates()
        }
    }
}

// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、iPhoneの傾きを使って動く「水平器」アプリである。アプリを開くと、画面に大きな円と小さいバブルが表示される。端末を前後・左右に傾けると、バブルの位置が変わる。端末がほぼ水平になると、バブルが緑色になり、「水平!」という文字が表示される。

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
import CoreMotion

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool = false

    func startUpdates() {
        guard motionManager.isDeviceMotionAvailable else {
            isAvailable = false
            return
        }

        isAvailable = true
        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}
```

**何をしているか：**  
`MotionManager`は、端末の動きや傾きを管理するためのクラスである。中で`CMMotionManager`を作り、`startDeviceMotionUpdates`を使ってセンサーデータをリアルタイムで受け取っている。受け取ったデータから、`pitch`、`roll`、`yaw`を取り出して、画面で使える変数に保存している。

**なぜこう書くのか：**  
センサーの処理を`ContentView`に直接たくさん書くと、画面のコードとセンサーのコードが混ざって読みにくくなる。`MotionManager`という別のクラスに分けることで、「センサーを読む役割」と「画面を表示する役割」を分けられる。

**もしこう書かなかったら：**  
`isDeviceMotionAvailable`を確認しないと、センサーが使えない環境でも処理を始めようとしてしまう。特にシミュレータではセンサーが使えないので、画面が期待通りに動かない可能性がある。`stopUpdates()`を書かないと、画面を閉じた後もセンサー更新が続き、バッテリー消費が増える原因になる。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
self.pitch = motion.attitude.pitch
self.roll = motion.attitude.roll
self.yaw = motion.attitude.yaw
```

**何をしているか：**  
`motion.attitude`は、端末の姿勢を表すデータである。その中から、前後の傾きである`pitch`、左右の傾きである`roll`、水平方向の回転である`yaw`を取り出している。

**なぜこう書くのか：**  
水平器では、特に前後と左右の傾きが重要である。そのため、`LevelIndicator`では`pitch`と`roll`を使ってバブルの位置を変えている。`yaw`は水平器のバブル移動には直接使っていないが、データ表示として画面に出しているので、端末の向きの変化を確認できる。

**もしこう書かなかったら：**  
`pitch`や`roll`を保存しなければ、画面側で端末の傾きを使えない。つまり、端末を傾けてもバブルが動かないアプリになってしまう。`yaw`を表示しなければ、水平回転の変化を確認できない。

---

### 歩数計（CMPedometer）

```swift
import CoreMotion

@Observable
class PedometerManager {
    private let pedometer = CMPedometer()

    var steps: Int = 0
    var distance: Double = 0
    var isAvailable: Bool = false

    func startUpdates() {
        guard CMPedometer.isStepCountingAvailable() else {
            isAvailable = false
            return
        }

        isAvailable = true
        let startDate = Calendar.current.startOfDay(for: Date())

        pedometer.startUpdates(from: startDate) { [weak self] data, error in
            guard let self = self, let data = data else { return }

            Task { @MainActor in
                self.steps = data.numberOfSteps.intValue
                self.distance = data.distance?.doubleValue ?? 0
            }
        }
    }

    func stopUpdates() {
        pedometer.stopUpdates()
    }
}
```

**何をしているか：**

この部分は、iPhoneの歩数計機能を使って、ユーザーが今日何歩歩いたかを取得するためのコードである。`CMPedometer`はCoreMotionの中にあるクラスで、歩数、歩いた距離、上った階数などの活動データを取得できる。

**なぜこう書くのか：**

歩数計は、すべての端末や環境で使えるとは限らない。特にシミュレータでは正しい歩数データを取得できないことが多い。

**もしこう書かなかったら：**

こう書かないと、歩数計が使えない端末でも処理を始めてしまい、正しいデータが表示されない可能性がある。 

---

### CoreLocationとの連携

```swift
import CoreLocation

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()

    var currentLocation: CLLocation?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func startUpdates() {
        manager.startUpdatingLocation()
    }

    func stopUpdates() {
        manager.stopUpdatingLocation()
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus

        if authorizationStatus == .authorizedWhenInUse ||
            authorizationStatus == .authorizedAlways {
            startUpdates()
        }
    }

    func locationManager(
        _ manager: CLLocationManager,
        didUpdateLocations locations: [CLLocation]
    ) {
        currentLocation = locations.last
    }
}
```

**何をしているか：**

この部分は、CoreLocationを使って現在地を取得するためのコードである。

**なぜこう書くのか：**

位置情報はプライバシーに関わるデータなので、勝手に使うことはできない。そのため、必ずユーザーに許可を求める必要がある。許可の状態は`authorizationStatus`で確認できる。

**もしこう書かなかったら：**

位置情報の許可を求めなければ、現在地を取得できない。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| 例：`CMPedometer` | 歩数や歩行距離をカウント | `pedometer.queryPedometerData(from: startDate, to: Date())` |
| `CoreMotion` | iPhoneやiPadの動き・傾きなどを扱うフレームワーク | `import CoreMotion` |
| `CMMotionManager` | デバイスモーションなどのセンサーデータを取得する管理クラス | `private let motionManager = CMMotionManager()` |
| `isDeviceMotionAvailable` | デバイスモーションが使えるか確認するプロパティ | `guard motionManager.isDeviceMotionAvailable else { return }` |

## 自分の実験メモ

**実験1：水平判定の条件を変えた**
- やったこと：`abs(pitch) < 0.03 && abs(roll) < 0.03`の`0.03`を`0.10`に変えてみた。
- 結果：端末が少し傾いていても、緑色になりやすくなった。
- わかったこと：水平判定の数値を大きくすると判定がゆるくなり、小さくすると判定が厳しくなる。

**実験2：バブルの移動量を変えた**
- やったこと：`maxOffset`を`100`から`60`に変えてみた。
- 結果：端末を傾けても、バブルの動きが小さくなった。
- わかったこと：センサーの値を画面上の動きに変える時は、倍率の調整が大切である。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**  
   `CMMotionManager`は何をするためのクラスですか。  
   **得られた理解：**  
   端末の加速度、回転、姿勢などのセンサー情報を取得するための管理クラスだと分かった。今回のコードでは、`deviceMotion`を使って`pitch`、`roll`、`yaw`を取得している。

2. **質問：**  
   `pitch`、`roll`、`yaw`の違いは何ですか。  
   **得られた理解：**  
   `pitch`は前後の傾き、`roll`は左右の傾き、`yaw`は水平回転を表す。水平器では主に`pitch`と`roll`が重要で、`yaw`は回転方向の確認に使える。

3. **質問：**  
   なぜ`onDisappear`で`stopUpdates()`を呼ぶ必要がありますか。  
   **得られた理解：**  
   センサーはリアルタイムで動き続けるため、使わない時は止めた方がよい。止めないと、画面を閉じた後も処理が続き、バッテリー消費の原因になると理解した。

## この章のまとめ

この章では、`CoreMotion`を使ってiPhoneの傾きを取得し、SwiftUIの画面に反映する方法を学んだ。重要なのは、センサーの値を直接見るだけでなく、その値をアプリの状態として保存し、ビューに渡して表示を変えることである。
