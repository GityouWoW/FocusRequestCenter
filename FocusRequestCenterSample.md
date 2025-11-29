# FocusController / FocusRequestCenter パターン導入用プロンプト

以下のサンプルコードを参考に、既存プロジェクトのフォーカス制御を **FocusController + FocusRequestCenter パターン** にリファクタリングしてください。  
このファイル全体を 1 回でコピペして使える形にしています。

---

## サンプルコード

    import SwiftUI
    import Combine

    // このファイルはサンプルであり変更しないでください。
    // ナビゲーションの仕組みの例であり画面遷移方法をこのサンプルに寄せる必要はありません。
    // 本番コードの画面遷移方法を優先しそちらへ合わせて下さい。
    /*
重要: シート内で TextField / TextEditor に初期フォーカスを当てたい場合、
必ずフォーカス状態 (@FocusState) は「シート内の子ビュー」に内包させてください。
親ビューに置いた @FocusState を sheet の中で使うのは禁止です。
フォーカスと表示階層は同一ルートで管理され、sheet は別階層のため親の FocusState は無効になります。
必ずシート専用の子ビューを作り、そこで @FocusState を保持してください。

    */

    // 1) 共通のフォーカスルート（画面＋フィールド）を定義
    enum FocusRouteSample: Equatable {
        case none
        case memo(field: MemoFieldSample)
        
        // 必要なら他の画面も
        // case profile(field: ProfileField)
    }

    // App.environmentObject(focusCenter)
    struct FocusContentSampleView: View {
        @EnvironmentObject var focusCenter: FocusRequestCenterSample
        
        var body: some View {
            NavigationStack {
                VStack {
                    NavigationLink("メモ画面へ") {
                        MemoScreenSample()
                    }
                    Button("メモ画面の Title にフォーカスして遷移") {
                        focusCenter.request(.memo(field: .title))
                    }
                    
                    Button("メモ画面の Body にフォーカスして遷移") {
                        focusCenter.request(.memo(field: .body))
                    }
                }
            }
        }
    }

    // 2) フィールド定義
    enum MemoFieldSample: Hashable {
        case title
        case body
    }

    // 3) フォーカス制御用の共通コントローラ
    struct FocusControllerSample<Field: Hashable> {
        let focusedField: FocusState<Field?>.Binding
        
        /// フォーカスを指定フィールドに移動（または nil で解除）
        func set(_ field: Field?) {
            // レイアウト更新とぶつからないように、非同期で実行
            DispatchQueue.main.async {
                self.focusedField.wrappedValue = field
            }
        }
    }

    // 4) フォーカス要求を管理するステートオブジェクト
    /// これはどの画面のどのフィールドにフォーカスしてほしいかを表すだけ
    /// 実際の @FocusState や TextField には触らない
    @MainActor
    final class FocusRequestCenterSample: ObservableObject {
        @Published var route: FocusRouteSample = .none
        
        func request(_ route: FocusRouteSample) {
            self.route = route
        }
        
        func clear() {
            self.route = .none
        }
    }

    // 5) メモ画面
    struct MemoScreenSample: View {
        @EnvironmentObject var focusCenter: FocusRequestCenterSample
        
        @FocusState private var focusedField: MemoFieldSample?
        
        @State private var title: String = ""
        @State private var content: String = ""
            
        private var focusController: FocusControllerSample<MemoFieldSample> {
            FocusControllerSample(focusedField: $focusedField)
        }
        
        var body: some View {
            VStack(spacing: 16) {
                TitleSampleView(
                    title: $title,
                    focusController: focusController
                )
                
                ContentBodySampleView(
                    content: $content,
                    focusController: focusController
                )
                
                HStack {
                    Button("この画面からタイトルにフォーカス") {
                        focusController.set(.title)
                    }
                    Button("この画面からボディーにフォーカス") {
                        focusController.set(.body)
                    }
                }
                .buttonStyle(.borderedProminent)
                
                Spacer()
            }
            .padding()
            .onAppear {
                // .onAppear：画面表示時に「既に要求が来ている」場合に処理
                applyFocusRoute(focusCenter.route)
            }
            .onChange(of: focusCenter.route) { _, newRoute in
                // .onChange：画面表示後に要求が来た場合に処理
                applyFocusRoute(newRoute)
            }
        }
        
        private func applyFocusRoute(_ route: FocusRouteSample) {
            switch route {
            case .memo(let field):
                focusController.set(field)
                // 一度処理したらクリアする
                focusCenter.clear()
            default:
                break
            }
        }
    }

    // 6) タイトル入力部分 View
    struct TitleSampleView: View {
        @Binding var title: String
        let focusController: FocusControllerSample<MemoFieldSample>
        
        var body: some View {
            VStack(alignment: .leading, spacing: 8) {
                Text("Title")
                    .font(.title)
                
                TextField("title", text: $title)
                    .textFieldStyle(.roundedBorder)
                    .focused(
                        focusController.focusedField,
                        equals: .title
                    )
                
                Button("Title にフォーカス") {
                    focusController.set(.title)
                }
                .font(.caption)
            }
        }
    }

    // 7) コンテンツ入力部分 View
    struct ContentBodySampleView: View {
        @Binding var content: String
        let focusController: FocusControllerSample<MemoFieldSample>
        
        var body: some View {
            VStack(alignment: .leading, spacing: 8) {
                Text("Content")
                    .font(.headline)
                
                TextEditor(text: $content)
                    .frame(minHeight: 120)
                    .overlay(
                        RoundedRectangle(cornerRadius: 8)
                            .stroke(Color.gray.opacity(0.3), lineWidth: 1)
                    )
                    .focused(
                        focusController.focusedField,
                        equals: .body
                    )
                
                Button("Body にフォーカス") {
                    focusController.set(.body)
                }
                .font(.caption)
            }
        }
    }

---

## 必須条件

### 設計方針

- サンプルコードの `FocusController<Field>` / `FocusRequestCenter` パターンを採用してください
- **画面内のフォーカス制御**：各画面で `@FocusState` + `FocusController` を使用
- **画面を跨ぐフォーカス制御**：`FocusRequestCenter` + `FocusRoute` enum を使用
- **既存の画面遷移方法**（`NavigationStack` / `NavigationView` / `sheet` / `fullScreenCover` / 独自 Router 等）は **変更せずそのまま活かす**
  - サンプルのナビゲーション実装に寄せる必要はありません
  - 本番コードの画面遷移ロジックに合わせて組み込んでください

### Swift 6 / strict concurrency 対応

- `FocusRequestCenter` には `@MainActor` を付けてください
- `@FocusState` は View レイヤーに閉じる（ViewModel に持たせない）
- actor の init 中に actor-isolated なインスタンスメソッドを呼ばないでください
- `ObservableObject` の中で別の `ObservableObject` を `@ObservedObject` で持たないでください

### VSA_Architecture 準拠

- **View**：UI 表示とフォーカス制御（`@FocusState` + `FocusController`）
- **State**：画面を跨ぐフォーカス要求の管理（`FocusRequestCenter` / `FocusRoute`）
- **Action**：`focusController.set()` / `focusCenter.request()` を通して行う
- 不必要な中間 ViewModel や過度な抽象化は避け、シンプルで実践的なコードにしてください

### コード品質・その他

- `Text("\(Int(minutes))分")` が `Text("\\(Int(minutes))分")` にならないよう注意してください
- `scrollPosition` を使う場合は **必ず `ScrollView` / `LazyVStack` と組み合わせる**（`List` とは組み合わせない）
- **コンパイルが必ず通る** 完成形のフルコード を提示してください
- 私は基本的にコピペで使うので、**そのまま貼り替えて動くコード** にしてください

---

## リファクタリング手順

### 1. 共通ファイルの作成

以下のファイルを新規作成してください（名称はプロジェクトに合わせて変更可）。

#### FocusController.swift

    import SwiftUI

    struct FocusController<Field: Hashable> {
        let focusedField: FocusState<Field?>.Binding
        
        /// フォーカスを指定フィールドに移動（または nil で解除）
        func set(_ field: Field?) {
            // レイアウト更新と競合しないよう非同期で実行
            DispatchQueue.main.async {
                self.focusedField.wrappedValue = field
            }
        }
    }

#### FocusRoute.swift

    import Foundation

    /// アプリ全体で使う共通のフォーカスルート
    enum FocusRoute: Equatable {
        case none
        // 既存の画面に合わせて case を追加してください
        // 例:
        // case memo(field: MemoField)
        // case profile(field: ProfileField)
    }

#### FocusRequestCenter.swift

    import SwiftUI

    /// フォーカス要求を管理するステートオブジェクト
    /// どの画面のどのフィールドにフォーカスしてほしいかを表すだけで、
    /// 実際の @FocusState や TextField には触らない
    @MainActor
    final class FocusRequestCenter: ObservableObject {
        @Published var route: FocusRoute = .none
        
        func request(_ route: FocusRoute) {
            self.route = route
        }
        
        func clear() {
            self.route = .none
        }
    }

### 2. App.swift の修正

アプリのエントリポイントで `FocusRequestCenter` を生成し、`environmentObject` で全画面に提供してください。

    import SwiftUI

    @main
    struct YourApp: App {
        @StateObject private var focusCenter = FocusRequestCenter()
        
        var body: some Scene {
            WindowGroup {
                YourRootView() // 既存のルート View 名に合わせてください
                    .environmentObject(focusCenter)
            }
        }
    }

### 3. 各画面の修正方針

#### 3-1. 画面内フォーカス制御がある View

1. **フォーカス対象フィールドの enum を定義**

        enum XxxField: Hashable {
            case title
            case body
            // 必要に応じて追加
        }

2. **View に `@FocusState` を追加**

        @FocusState private var focusedField: XxxField?

3. **`FocusController` を用意**

        private var focusController: FocusController<XxxField> {
            FocusController(focusedField: $focusedField)
        }

4. **`TextField` / `TextEditor` に `.focused(...)` を付与**

        TextField("title", text: $title)
            .textFieldStyle(.roundedBorder)
            .focused(focusController.focusedField, equals: .title)

5. **フォーカス変更は `focusController.set(.title)` のように呼び出す**

#### 3-2. 画面を跨ぐフォーカス制御がある View

1. **`@EnvironmentObject var focusCenter: FocusRequestCenter` を追加**

2. **`.onAppear` と `.onChange` でルートを監視**

        .onAppear {
            applyFocusRoute(focusCenter.route)
        }
        .onChange(of: focusCenter.route) { _, newRoute in
            applyFocusRoute(newRoute)
        }

3. **`applyFocusRoute` で自分に関係する case だけを処理し、処理後に `focusCenter.clear()` を呼ぶ**

        private func applyFocusRoute(_ route: FocusRoute) {
            switch route {
            case .memo(let field):
                focusController.set(field)
                focusCenter.clear()
            default:
                break
            }
        }

4. **他画面からのフォーカス要求は、既存の遷移処理の直前 or 直後に**

        focusCenter.request(.memo(field: .title))

のように呼び出してください（遷移方法は既存ロジックに合わせてください）

#### 3-3. 子 View（部分 View）

すでに `@Binding var text: String` などで受けている場合、そこに

    let focusController: FocusController<XxxField>

を追加で受け取り、

    .focused(focusController.focusedField, equals: .xxx)

を付けてください。  
フォーカス制御が不要な子 View には渡す必要はありません。

### 4. 既存の @FocusState の置き換え

- **既存で `@FocusState` を直接子 View に渡している場合**：
  - `@Binding var focusedField: XxxField?` → `let focusController: FocusController<XxxField>` に変更
  - `.focused($focusedField, equals: .xxx)` → `.focused(focusController.focusedField, equals: .xxx)` に変更
- **既存の ViewModel に `@FocusState` 相当の状態を持たせている場合**：
  - その部分は View 側に移し、ViewModel からは削除してください（ViewModel はフォーカスを知らない設計に）

---

## 出力フォーマット

あなた（AI）からの回答は、以下の形式でお願いします。

1. **現状コードの問題点・改善点（箇条書き）**
2. **リファクタリング後のフルコード**
   - FocusController.swift
   - FocusRoute.swift
   - FocusRequestCenter.swift
   - App.swift（修正箇所がある場合）
   - 各画面の View（修正後の完全版）
   - 必要であれば、関連する ViewModel の修正版
   - すべて **コンパイルが通る形で** 提示してください
3. **変更点のまとめ**
   - 何をどう変更したかを箇条書きで説明してください
   - 特にフォーカス制御の流れ（どこで request して、どこで set しているか）を明確にしてください
4. **動作確認ポイント**
   - どの画面で、どのボタンを押すと、どのフィールドにフォーカスが当たるべきか
   - Navigation / sheet / fullScreenCover など、各パターンごとの確認方法

---

## 補足

- **既存コードに ViewModel がある場合でも**、
  - フォーカス制御は View レイヤーに閉じる
  - ViewModel にはフォーカス状態や FocusController を持たせない
- **Sheet / fullScreenCover の場合も**、
  - シート側の View が自分の `@FocusState` と `FocusController` を持つようにしてください
- **将来的にフォーカス対象画面が増えることを考慮し**、
  - `FocusRoute` の命名や case の分け方も分かりやすく設計してください
