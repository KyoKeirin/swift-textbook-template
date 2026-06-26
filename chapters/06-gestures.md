# 第6章：ジェスチャー操作

> 執筆者：許 敬林
> 最終更新：2026-06-26

## この章で学ぶこと

この章では、SwiftUIでユーザーの指の動きを受け取る「ジェスチャー操作」を学んだ。

## 模範コードの全体像

```swift
// ============================================
// 第6章（基本）：ジェスチャーで操作するカードアプリ
// ============================================
// タップ、ロングプレス、ドラッグ、ピンチ、回転の
// 各ジェスチャーを実際に体験しながら学びます。
// ============================================

import SwiftUI

// MARK: - メインビュー

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("タップ & ロングプレス") {
                    TapDemoView()
                }
                NavigationLink("ドラッグ") {
                    DragDemoView()
                }
                NavigationLink("ピンチ（拡大縮小）") {
                    MagnifyDemoView()
                }
                NavigationLink("回転") {
                    RotateDemoView()
                }
                NavigationLink("組み合わせ") {
                    CombinedDemoView()
                }
            }
            .navigationTitle("ジェスチャー体験")
        }
    }
}

// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}

// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}

// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}

// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、SwiftUIのジェスチャーを体験するための練習アプリである。最初の画面にはリストがあり、「タップ & ロングプレス」「ドラッグ」「ピンチ（拡大縮小）」「回転」「組み合わせ」の5つのページへ移動できる。

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
@State private var tapCount = 0
@State private var backgroundColor: Color = .blue
@State private var isPressed = false

RoundedRectangle(cornerRadius: 16)
    .fill(backgroundColor)
    .frame(width: 200, height: 200)
    .overlay {
        Text("タップしてね")
            .foregroundStyle(.white)
            .font(.headline)
    }
    .onTapGesture {
        tapCount += 1
        backgroundColor = Color(
            hue: Double.random(in: 0...1),
            saturation: 0.7,
            brightness: 0.9
        )
    }

Circle()
    .fill(isPressed ? .green : .orange)
    .frame(width: 120, height: 120)
    .scaleEffect(isPressed ? 1.3 : 1.0)
    .overlay {
        Text(isPressed ? "成功!" : "長押し")
            .foregroundStyle(.white)
            .font(.headline)
    }
    .animation(.spring(duration: 0.3), value: isPressed)
    .onLongPressGesture(minimumDuration: 1.0) {
        isPressed = true
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            isPressed = false
        }
    }
```

**何をしているか：**

この部分では、タップと長押しの2つの基本ジェスチャーを使っている。`tapCount`はタップした回数を保存する変数で、`backgroundColor`は四角形の色を保存する変数である。四角形に`.onTapGesture`を付けることで、ユーザーがタップした時に`tapCount += 1`で回数を増やし、`Color(hue:saturation:brightness:)`でランダムな色に変えている。

**なぜこう書くのか：**

SwiftUIでは、画面の状態を変えたい時に`@State`を使う。`@State`を付けると、値が変わった時に画面も自動で更新される。例えば、`tapCount`が増えたら、`Text("タップ回数: \(tapCount)")`の表示もすぐに変わる。`isPressed`も同じで、`true`か`false`によって丸の色、大きさ、文字が変わる。

**もしこう書かなかったら：**

`@State`を付けなかったら、タップして変数を変えても画面が正しく更新されない。`.onTapGesture`を消すと、四角形をタップしても何も起きない。`.onLongPressGesture`の`minimumDuration`を短くすると、少し触っただけでも長押し成功になりやすい。反対に長くしすぎると、ユーザーが反応しないと感じる可能性がある。

---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero
@State private var lastOffset: CGSize = .zero

RoundedRectangle(cornerRadius: 20)
    .frame(width: 200, height: 280)
    .offset(offset)
    .gesture(
        DragGesture()
            .onChanged { value in
                offset = CGSize(
                    width: lastOffset.width + value.translation.width,
                    height: lastOffset.height + value.translation.height
                )
            }
            .onEnded { _ in
                lastOffset = offset
            }
    )

Button("リセット") {
    withAnimation(.spring) {
        offset = .zero
        lastOffset = .zero
    }
}
```

**何をしているか：**

この部分では、カードのような四角形をドラッグできるようにしている。`offset`は現在の表示位置のずれを表す。`lastOffset`は、前回ドラッグが終わった時の位置を保存するための変数である。

**なぜこう書くのか：**

ドラッグでは「今のドラッグ中の移動量」と「前回までに動かした位置」を分ける必要がある。`value.translation`だけを使うと、毎回ドラッグ開始位置からの移動量だけになる。そのため、2回目以降のドラッグで位置が不自然に戻ることがある。

**もしこう書かなかったら：**

`lastOffset`を使わずに`offset = value.translation`だけにすると、ドラッグをやめてもう一度触った時、カードの位置が急に変わることがある。`.offset(offset)`を書かなかったら、`offset`の値が変わっても画面上のカードは動かない。`.onEnded`を書かなかったら、前回の位置を保存できないため、連続して自然に動かすことが難しくなる。

---

### ジェスチャーの組み合わせとアニメーション

```swift
@State private var offset: CGSize = .zero
@State private var lastOffset: CGSize = .zero
@State private var scale: CGFloat = 1.0
@State private var lastScale: CGFloat = 1.0
@State private var angle: Angle = .zero
@State private var lastAngle: Angle = .zero

Image(systemName: "photo.artframe")
    .font(.system(size: 120))
    .foregroundStyle(.indigo)
    .scaleEffect(scale)
    .rotationEffect(angle)
    .offset(offset)
    .gesture(
        DragGesture()
            .onChanged { value in
                offset = CGSize(
                    width: lastOffset.width + value.translation.width,
                    height: lastOffset.height + value.translation.height
                )
            }
            .onEnded { _ in
                lastOffset = offset
            }
    )
    .gesture(
        MagnifyGesture()
            .onChanged { value in
                scale = lastScale * value.magnification
            }
            .onEnded { _ in
                lastScale = scale
            }
    )
    .gesture(
        RotateGesture()
            .onChanged { value in
                angle = lastAngle + value.rotation
            }
            .onEnded { _ in
                lastAngle = angle
            }
    )

Button("リセット") {
    withAnimation(.spring) {
        offset = .zero
        lastOffset = .zero
        scale = 1.0
        lastScale = 1.0
        angle = .zero
        lastAngle = .zero
    }
}
```

**何をしているか：**

この部分では、1つの`Image(systemName: "photo.artframe")`に対して、ドラッグ、ピンチ、回転の3つのジェスチャーを付けている。表示には`.scaleEffect(scale)`、`.rotationEffect(angle)`、`.offset(offset)`を全部使っているため、画像アイコンを移動しながら大きさを変えたり、回転したりできる。

**なぜこう書くのか：**

実際のアプリでは、1つの部品に複数の操作を付けたいことがある。例えば写真編集アプリでは、写真を動かしながら拡大し、角度も調整する。このサンプルはその基本形である。

**もしこう書かなかったら：**

ジェスチャーを1つだけにすると、移動・拡大・回転のうち1種類しかできない。リセットボタンで`lastOffset`、`lastScale`、`lastAngle`を戻し忘れると、見た目は初期状態に戻っても、次の操作で古い値が残っていて変な動きになる可能性がある。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`DragGesture` | ドラッグジェスチャーを認識するジェスチャーレコグナイザー | `.gesture(DragGesture().onChanged { ... })` |
| 例：`MagnificationGesture` | ピンチジェスチャーで拡大・縮小を認識 | `.gesture(MagnificationGesture().onChanged { scale in ... })` |
| `.onLongPressGesture` | 長押しされた時の処理を書く。 | `.onLongPressGesture(minimumDuration: 1.0) { ... }` |
| `DragGesture` | 指でドラッグした移動量を受け取る。 | `.gesture(DragGesture().onChanged { value in ... })` |
| `withAnimation` | 状態が変わる時にアニメーションを付ける。 | `withAnimation(.spring) { scale = 1.0 }` |

## 自分の実験メモ

**実験1：タップした時の色の変化を止める**
- やったこと：`.onTapGesture`の中で、`backgroundColor = Color(...)`の部分をコメントアウトしてみる想定で考えた。
- 結果：タップ回数は増えるが、四角形の色は青のままになる。
- わかったこと：1つのジェスチャーの中で、複数の状態を同時に変えられる。今回は`tapCount`と`backgroundColor`を別々に変えている。

**実験2：ドラッグの`lastOffset`を使わない場合を考える**
- やったこと：`offset = CGSize(width: value.translation.width, height: value.translation.height)`のように、`lastOffset`を足さない形に変える想定で考えた。
- 結果：1回目は動くが、2回目以降に前の位置から自然に動かすことが難しくなる。
- わかったこと：ドラッグでは「前回までの位置」と「今の操作量」を分けて保存することが大切である。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**  
   `offset`と`lastOffset`はなぜ2つ必要か。1つだけではだめなの？  
   **得られた理解：**  
   `offset`は今画面に表示する位置で、`lastOffset`は前回の操作が終わった位置である。2つを分けることで、次のドラッグを前の場所から続けられる。

2. **質問：**  
   `MagnifyGesture`の`value.magnification`は何を表している？
   **得られた理解：**  
   ピンチ中の拡大率を表している。`1.0`なら元の大きさ、`2.0`なら2倍、`0.5`なら半分という考え方で理解できる。

3. **質問：**  
   `withAnimation(.spring)`はなぜリセットボタンで使う？  
   **得られた理解：**  
   値を戻すだけならアニメーションはなくてもよい。しかし、アニメーションを付けると見た目が自然になり、ユーザーが変化を理解しやすくなる。

## この章のまとめ

この章で一番大切だと思ったことは、ジェスチャーは「ユーザーの指の動き」を受け取り、その結果を`@State`の値に保存して、画面表示に反映する仕組みだということである。タップや長押しは比較的簡単に書けるが、ドラッグ、ピンチ、回転では、今の操作量だけでなく前回までの状態も保存する必要がある。
