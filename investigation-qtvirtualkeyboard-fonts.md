# QtVirtualKeyboard フォント仕様・構造 調査ドキュメント

## 目次

1. [アーキテクチャ概要](#1-アーキテクチャ概要)
2. [フォントの読み込みメカニズム](#2-フォントの読み込みメカニズム)
3. [ソースコード上のフォント関連構造](#3-ソースコード上のフォント関連構造)
4. [レイアウトとスタイルの分離原則](#4-レイアウトとスタイルの分離原則)
5. [KeyboardStyle のフォント関連プロパティ](#5-keyboardstyle-のフォント関連プロパティ)
6. [scaleHint によるフォントサイズスケーリング](#6-scalehint-によるフォントサイズスケーリング)
7. [システムフォントとの関係](#7-システムフォントとの関係)
8. [入力モード別のフォント挙動](#8-入力モード別のフォント挙動)
9. [カスタムフォントの設定方法](#9-カスタムフォントの設定方法)
10. [関連する環境変数・API 一覧](#10-関連する環境変数api-一覧)
11. [既知の問題と注意点](#11-既知の問題と注意点)
12. [参考資料](#12-参考資料)

---

## 1. アーキテクチャ概要

QtVirtualKeyboard はフォントの扱いにおいて、**レイアウト（コンテンツ）とスタイル（ビジュアル）を厳格に分離する** 設計を採用している。

```
┌─────────────────────────────────────────────────────┐
│                   アプリケーション                      │
│   ┌─────────────────────────────────────────────┐   │
│   │          InputPanel (QML)                    │   │
│   │   ┌─────────────┐   ┌──────────────────┐   │   │
│   │   │   Layout     │   │     Style        │   │   │
│   │   │ (キー配列)    │   │ (見た目・フォント) │   │   │
│   │   │              │   │                  │   │   │
│   │   │  Key {       │   │  KeyboardStyle { │   │   │
│   │   │   text: "q"  │──▶│   fontFamily     │   │   │
│   │   │  }           │   │   keyPanel       │   │   │
│   │   │              │   │   scaleHint      │   │   │
│   │   └─────────────┘   └──────────────────┘   │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**重要な原則**: キーボードレイアウトはキーの論理的な配列（どの文字をどの位置に配置するか）のみを定義する。フォントファミリー、サイズ、色などの視覚的要素はすべて `KeyboardStyle` が担当する。

### フォントレンダリングのパイプライン

1. キーボードレイアウトがキーデータを定義（`text`, `displayText`, `smallText`）
2. `KeyboardStyle` がビジュアルデリゲートを提供（`keyPanel`, `enterKeyPanel` 等）
3. 各デリゲート内の `Text` 要素が `control.displayText` を参照し、`font` プロパティを指定
4. Qt Quick のテキストレンダリングエンジンがグリフをラスタライズ

---

## 2. フォントの読み込みメカニズム

QtVirtualKeyboard でフォントを読み込む方法は大きく2つある。

### 2.1 C++ 側での読み込み（QFontDatabase）

```cpp
// main.cpp（QMLエンジン生成前に実行）
int fontId = QFontDatabase::addApplicationFont(":/fonts/MyFont.ttf");
QStringList families = QFontDatabase::applicationFontFamilies(fontId);
qDebug() << "Loaded font families:" << families;
```

**ポイント**: `QFontDatabase::addApplicationFont()` はグローバルにフォントを登録する。QMLエンジン生成**前**に呼ぶことで、全ての QML コンポーネントからそのフォントが利用可能になる。

### 2.2 QML 側での読み込み（FontLoader）

```qml
import QtQuick 2.0

FontLoader {
    id: customFont
    source: "qrc:/fonts/MyCustomFont.ttf"
}
```

`FontLoader` の主要プロパティ:

| プロパティ | 型 | 説明 |
|---|---|---|
| `source` | `url` | フォントファイルの URL（`qrc:`, `file:`, リモート URL に対応） |
| `name` | `string` | 読み込み完了後に自動設定されるフォントファミリー名 |
| `font` | `font` | 読み込まれたフォントのデフォルトクエリ（Qt 6.0〜） |
| `status` | `enumeration` | `Null`, `Ready`, `Loading`, `Error` のいずれか |

### 2.3 Qt リソースシステム（QRC）での埋め込み

フォントファイルは通常 Qt リソースシステムに埋め込む:

```xml
<!-- resources.qrc -->
<RCC>
    <qresource prefix="/fonts">
        <file>NotoSansJP-Regular.ttf</file>
        <file>NotoSansJP-Bold.ttf</file>
    </qresource>
</RCC>
```

QML から `source: "qrc:/fonts/NotoSansJP-Regular.ttf"` として参照可能。

---

## 3. ソースコード上のフォント関連構造

### 3.1 Qt 5 のディレクトリ構造

```
src/virtualkeyboard/
├── content/
│   ├── styles/
│   │   ├── default/
│   │   │   ├── style.qml          ← デフォルトスタイル（fontFamily: "Sans"）
│   │   │   └── images/
│   │   └── retro/
│   │       ├── style.qml          ← レトロスタイル
│   │       └── images/
│   └── layouts/
│       ├── fallback/              ← フォールバックレイアウト（必須）
│       │   ├── main.qml
│       │   ├── symbols.qml
│       │   ├── digits.qml
│       │   ├── numbers.qml
│       │   ├── handwriting.qml
│       │   └── dialpad.qml
│       ├── en_GB/                 ← ロケール別レイアウト
│       ├── ja_JP/
│       └── zh_CN/
└── 3rdparty/
    └── hunspell/                  ← スペルチェック辞書
```

### 3.2 Qt 6 のディレクトリ構造

```
src/
├── styles/
│   └── builtin/
│       ├── default/
│       │   ├── style.qml
│       │   └── images/
│       └── retro/
│           ├── style.qml
│           └── images/
├── layouts/
│   └── builtin/
│       └── <language_COUNTRY>/
├── plugins/                       ← 入力メソッドプラグイン（QMLモジュール）
└── settings/
    └── qquickvirtualkeyboardsettings.cpp
```

### 3.3 重要な観察事項

**QtVirtualKeyboard はフォントファイル（`.ttf`, `.otf`）をソースツリーに同梱しない**。フォントは以下のいずれかの方法で供給される:

- ターゲットプラットフォームのシステムフォント
- `QFontDatabase::addApplicationFont()` でアプリケーション起動時に登録
- QML の `FontLoader` で動的に読み込み

デフォルトスタイルの `fontFamily` は `"Sans"` に設定されており、これはシステムのサンセリフフォントに解決されることが期待されている。

---

## 4. レイアウトとスタイルの分離原則

### 4.1 レイアウト側（フォント指定なし）

レイアウトはキーの論理情報のみを定義する。**フォントに関する指定は一切含まない**。

```qml
// layouts/en_GB/main.qml
import QtQuick 2.0
import QtQuick.VirtualKeyboard 2.1

KeyboardLayout {
    keyWeight: 160
    KeyboardRow {
        Key { key: Qt.Key_Q; text: "q" }
        Key { key: Qt.Key_W; text: "w" }
        Key { key: Qt.Key_E; text: "e" }
        // ...
    }
}
```

`Key`（`BaseKey` を継承）のテキスト関連プロパティ:

| プロパティ | 型 | 説明 |
|---|---|---|
| `text` | `string` | 入力メソッド処理用のキーテキスト（Unicode） |
| `displayText` | `string` | キーボード上に表示される文字列（デフォルトは `text`） |
| `smallText` | `string` | キーの隅に小さく表示される文字列 |
| `smallTextVisible` | `bool` | `smallText` の表示制御 |
| `alternativeKeys` | `var` | 長押し時の代替キー一覧 |

**`Key` 型に `font` プロパティは存在しない。** フォントレンダリングはスタイルの `keyPanel` デリゲートが全面的に担当する。

### 4.2 スタイル側（フォント定義）

```qml
// styles/default/style.qml
KeyboardStyle {
    readonly property string fontFamily: "Sans"

    keyPanel: KeyPanel {
        // ここでフォントを指定
        Text {
            text: control.displayText
            font.family: fontFamily
            font.pixelSize: 52 * scaleHint
            font.weight: Font.Normal
            font.capitalization: control.uppercased
                ? Font.AllUppercase : Font.MixedCase
        }
    }
}
```

### 4.3 keyPanelDelegate の役割

各 `BaseKey` は `keyPanelDelegate` エイリアスプロパティを持つ。このデリゲートなしではキーは**不可視**になる。デリゲートはアクティブな `KeyboardStyle` から供給され、フォントを含む全ての視覚情報を定義する。

---

## 5. KeyboardStyle のフォント関連プロパティ

### 5.1 スタイルレベルのフォントプロパティ

| プロパティ | 型 | 説明 | バージョン |
|---|---|---|---|
| `fullScreenInputFont` | `font` | 全画面入力フィールドのフォント | Styles 2.2 |
| `fullScreenInputColor` | `color` | 全画面入力のテキスト色（デフォルト: 黒） | Styles 2.2 |
| `fullScreenInputSelectedTextColor` | `color` | 選択テキストの色 | Styles 2.2 |
| `scaleHint` | `real` | `keyboardHeight / keyboardDesignHeight` | Styles 1.0 |
| `keyboardDesignWidth` | `real` | キーボードのデザイン時幅 | Styles 1.0 |
| `keyboardDesignHeight` | `real` | キーボードのデザイン時高さ | Styles 1.0 |
| `keyboardHeight` | `real` | 実行時のキーボード高さ | Styles 1.0 |

### 5.2 コンポーネントデリゲート一覧（フォントが設定される場所）

| デリゲートプロパティ | 説明 | フォント適用箇所 |
|---|---|---|
| `keyPanel` | 通常キーのテンプレート | キーラベル、小文字テキスト |
| `backspaceKeyPanel` | バックスペースキー | （通常アイコン） |
| `enterKeyPanel` | Enterキー | ラベル文字列 |
| `shiftKeyPanel` | Shiftキー | （通常アイコン） |
| `spaceKeyPanel` | スペースキー | 言語名表示 |
| `symbolKeyPanel` | 記号キー | モード切替ラベル |
| `modeKeyPanel` | モードキー | モードラベル |
| `handwritingKeyPanel` | 手書きモードキー | （通常アイコン） |
| `hideKeyPanel` | 非表示キー | （通常アイコン） |
| `languageKeyPanel` | 言語キー | 言語名表示 |
| `characterPreviewDelegate` | 文字プレビューポップアップ | プレビュー文字 |
| `alternateKeysListDelegate` | 代替キーリスト項目 | 代替文字ラベル |
| `selectionListDelegate` | 単語候補リスト項目 | 候補テキスト |
| `popupListDelegate` | ポップアップリスト項目 | リスト項目テキスト |
| `languageListDelegate` | 言語リスト項目 | 言語名テキスト |

### 5.3 keyPanel 内で利用可能な control プロパティ

各デリゲート内では `control` オブジェクトを通じてキーの情報にアクセスする:

| プロパティ | 型 | 説明 |
|---|---|---|
| `control.displayText` | `string` | キーの表示テキスト |
| `control.smallText` | `string` | 隅の小文字テキスト |
| `control.smallTextVisible` | `bool` | 小文字テキストの可視性 |
| `control.text` | `string` | キーの Unicode テキスト |
| `control.key` | `int` | Unicode キーコード |
| `control.pressed` | `bool` | 押下状態 |
| `control.enabled` | `bool` | 有効/無効状態 |
| `control.uppercased` | `bool` | 大文字状態（Shift/CapsLock） |
| `control.alternativeKeys` | `var` | 代替キーの一覧 |

---

## 6. scaleHint によるフォントサイズスケーリング

### 6.1 scaleHint の仕組み

`scaleHint` は QtVirtualKeyboard のスケーリングシステムの中核を成すプロパティである。

```
scaleHint = keyboardHeight / keyboardDesignHeight
```

**全てのピクセル値は `scaleHint` に比例して指定しなければならない。** これにより、キーボードがリサイズされても均一にスケーリングされる。

### 6.2 デザイン寸法

```qml
KeyboardStyle {
    keyboardDesignWidth: 2560    // デザイン時の幅
    keyboardDesignHeight: 800    // デザイン時の高さ
}
```

実行時のキーボード高さはランタイムで決定される。キーボードは `keyboardDesignWidth / keyboardDesignHeight` のアスペクト比を維持する。

### 6.3 フォントサイズの階層（一般的な設計値の例）

| 要素 | ピクセルサイズの計算式 | デザイン時の値 | 用途 |
|---|---|---|---|
| 通常キーテキスト | `52 * scaleHint` | 52px | メインのキーラベル |
| 小文字テキスト（隅） | `38 * scaleHint` | 38px | キー隅の補助テキスト |
| 記号/モードキー | `44 * scaleHint` | 44px | 記号切替ラベル |
| Enter キーテキスト | `44 * scaleHint` | 44px | Enter ラベル |
| 文字プレビューポップアップ | `82 * scaleHint` | 82px | キー押下時のプレビュー |
| 候補リスト項目 | `44 * scaleHint` | 44px | 変換候補 |
| スペースキー言語名 | `48 * scaleHint` | 48px | 言語名表示 |

### 6.4 pixelSize と pointSize の使い分け

QtVirtualKeyboard は**必ず `font.pixelSize` を使用**する（`font.pointSize` は使わない）。

| 特性 | `font.pixelSize` | `font.pointSize` |
|---|---|---|
| 単位 | ピクセル | ポイント（1/72インチ） |
| DPI依存 | なし | あり |
| scaleHint との相性 | 最適 | 不適切 |
| 使用場面 | **常にこちらを使用** | 使用しない |

理由: `pointSize` は DPI に依存するため、`scaleHint` のスケーリングモデルと相容れない。`pixelSize` であれば正確なピクセル制御が可能で、`scaleHint` との乗算で適切にスケーリングされる。

### 6.5 スケーリングの計算例

```
デザイン時: keyboardDesignHeight = 800, フォントサイズ = 52px
実行時:     keyboardHeight = 400 の場合

scaleHint = 400 / 800 = 0.5
実際のフォントサイズ = 52 * 0.5 = 26px
```

---

## 7. システムフォントとの関係

### 7.1 フォント解決チェーン

QtVirtualKeyboard のフォントは以下の順序で解決される:

```
1. スタイルの fontFamily プロパティ
   （例: "Sans", "Roboto", "Noto Sans JP"）
       │
       ▼
2. Qt のフォントデータベースを検索
   （QFontDatabase に登録済みのフォントを照合）
       │
       ▼
3. フォントフォールバック
   （名前が見つからない場合、Qt のフォントマッチングアルゴリズムが代替を選択）
       │
       ▼
4. プラットフォーム固有のフォント置換
   （例: Windows レジストリの FontSubstitutes）
```

### 7.2 プラットフォーム別の挙動

| プラットフォーム | デフォルト "Sans" の解決先 | CJK フォント | 注意事項 |
|---|---|---|---|
| Linux (X11/Wayland) | fontconfig で解決 | Noto Sans CJK 等 | `QT_QPA_FONTDIR` で指定可能 |
| Windows | **解決失敗の可能性あり** | MS Gothic 等 | レジストリ置換が必要な場合あり |
| macOS | Helvetica Neue 等 | ヒラギノ等 | 通常問題なし |
| 組込み Linux (EGLFS) | 明示的なデプロイが必要 | 手動配置 | 必ずフォントを含める |
| Android | Roboto | Noto Sans CJK | システムフォント利用 |

### 7.3 組込みシステムでのフォント配置

組込み Linux など最小構成の環境では、フォントファイルを手動で配置する必要がある:

```bash
# フォントディレクトリの指定
export QT_QPA_FONTDIR=/usr/share/fonts/truetype/

# ディレクトリ構造の例
/usr/share/fonts/truetype/
├── NotoSansJP-Regular.ttf
├── NotoSansJP-Bold.ttf
├── NotoSansCJK-Regular.ttc
└── DejaVuSans.ttf
```

ランタイムでのフォント確認:

```cpp
QFontDatabase db;
qDebug() << "Available families:" << db.families();
```

---

## 8. 入力モード別のフォント挙動

### 8.1 標準キーボードモード

キーラベルは `keyPanel` デリゲートの `Text` 要素を通じてレンダリングされる。

```qml
Text {
    text: control.displayText
    font.family: fontFamily          // スタイルで定義
    font.pixelSize: 52 * scaleHint   // scaleHint でスケーリング
    font.capitalization: control.uppercased
        ? Font.AllUppercase
        : Font.MixedCase             // Shift 状態に連動
}
```

**大文字/小文字の制御**: `font.capitalization` が `control.uppercased` に連動して動的に変化する。Shift キーや CapsLock の状態が反映される。

### 8.2 手書き入力モード

手書きモードはフォントとは根本的に異なるアプローチを取る:

```
入力: QVirtualKeyboardTrace（タッチ/ストロークデータ）
      │
      ▼
描画: TraceCanvas（Canvas 2D レンダリング）
      │ renderSmoothedLine() による描画
      │ ※フォントは使用しない
      ▼
認識: 認識エンジン（Cerence, MyScript 等）
      │
      ▼
出力: 認識されたテキスト → アプリケーションのテキストフィールド
      │ アプリケーション側のフォントで表示
```

- **ストローク描画**: `traceCanvasDelegate`（`TraceCanvas` 型）がCanvas 2D でストロークを描画。フォントは関与しない
- **認識結果**: 認識されたテキストはアプリケーションのテキストフィールドに出力され、**アプリケーション側のフォント**でレンダリングされる
- **ガイドライン**: `TraceInputKey` の `horizontalRulers` プロパティで手書きガイド線を定義

### 8.3 Pinyin / CJK 入力モード

```
キーラベル: 通常の Latin キーラベル（keyPanel のフォントで描画）
      │
      ▼
候補リスト: selectionListDelegate でレンダリング
      │ CJK 対応フォントが必要
      ▼
確定文字: アプリケーション側のフォントで表示
```

- **キーラベル**: 通常の Latin 文字キーとして `keyPanel` のフォントで描画
- **候補リスト**: `selectionListDelegate` を通じて表示。**CJK グリフを含むフォントがシステムに必要**
- **フォント最適化**: アプリケーションのテキストフィールドと `selectionListDelegate` で同じフォントを使用することで、CJK グリフテーブルの重複読み込みを回避できる

### 8.4 全画面入力モード

全画面入力モードには `KeyboardStyle` に専用のフォントプロパティがある:

```qml
KeyboardStyle {
    fullScreenInputFont.family: "Noto Sans JP"
    fullScreenInputFont.pixelSize: 52 * scaleHint
    fullScreenInputColor: "#000000"
    fullScreenInputSelectedTextColor: "#0000ff"
}
```

---

## 9. カスタムフォントの設定方法

### 方法 1: カスタムスタイルの作成（推奨）

最も包括的で推奨される方法。スタイル全体をカスタマイズしてフォントを変更する。

**ステップ 1: スタイルディレクトリの作成**

```
<QML_IMPORT_PATH>/QtQuick/VirtualKeyboard/Styles/MyStyle/
├── style.qml
└── images/
    ├── backspace.png
    ├── enter.png
    └── ...
```

**ステップ 2: style.qml の作成**

```qml
import QtQuick 2.0
import QtQuick.VirtualKeyboard 2.1
import QtQuick.VirtualKeyboard.Styles 2.1

KeyboardStyle {
    id: currentStyle

    // --- カスタムフォントの読み込み ---
    FontLoader {
        id: mainFont
        source: "qrc:/fonts/NotoSansJP-Regular.ttf"
    }

    readonly property string fontFamily: mainFont.name

    // --- デザイン寸法 ---
    keyboardDesignWidth: 2560
    keyboardDesignHeight: 800

    // --- 全画面入力フォント ---
    fullScreenInputFont.family: fontFamily
    fullScreenInputFont.pixelSize: 52 * scaleHint
    fullScreenInputColor: "#000000"

    // --- 通常キー ---
    keyPanel: KeyPanel {
        Rectangle {
            id: keyBackground
            anchors.fill: parent
            anchors.margins: 13 * scaleHint
            color: "#35322f"
            radius: 5 * scaleHint

            // 小文字テキスト（隅）
            Text {
                visible: control.smallTextVisible
                text: control.smallText
                font.family: fontFamily
                font.weight: Font.Normal
                font.pixelSize: 38 * scaleHint
                color: "#aaaaaa"
                anchors {
                    right: parent.right
                    top: parent.top
                    margins: 8 * scaleHint
                }
            }

            // メインキーラベル
            Text {
                anchors.fill: parent
                anchors.margins: 45 * scaleHint
                text: control.displayText
                font.family: fontFamily
                font.weight: Font.Normal
                font.pixelSize: 52 * scaleHint
                font.capitalization: control.uppercased
                    ? Font.AllUppercase : Font.MixedCase
                color: "#ffffff"
                horizontalAlignment: Text.AlignHCenter
                verticalAlignment: Text.AlignVCenter
            }
        }

        states: [
            State {
                name: "pressed"
                PropertyChanges { target: keyBackground; opacity: 0.75 }
            },
            State {
                name: "disabled"
                PropertyChanges { target: keyBackground; opacity: 0.20 }
            }
        ]
    }

    // --- スペースキー ---
    spaceKeyPanel: KeyPanel {
        Rectangle {
            anchors.fill: parent
            color: "#35322f"
            Text {
                text: Qt.locale(InputContext.locale).nativeLanguageName
                font.family: fontFamily
                font.pixelSize: 48 * scaleHint
                color: "#ffffff"
                anchors.centerIn: parent
            }
        }
    }

    // --- 文字プレビューポップアップ ---
    characterPreviewDelegate: Item {
        property string previewText
        Text {
            text: previewText
            font.family: fontFamily
            font.pixelSize: 82 * scaleHint
            font.weight: Font.Bold
            color: "#ffffff"
            anchors.centerIn: parent
        }
    }

    // --- 候補リスト ---
    selectionListDelegate: SelectionListItem {
        property string decorationText
        Text {
            text: decorationText
            font.family: fontFamily
            font.pixelSize: 44 * scaleHint
            color: "#ffffff"
            anchors.centerIn: parent
        }
    }
}
```

**ステップ 3: スタイルの有効化**

```cpp
// C++ 側（QML エンジン生成前）
qputenv("QT_VIRTUALKEYBOARD_STYLE", "MyStyle");
```

または QML 側:

```qml
import QtQuick.VirtualKeyboard.Settings 2.0

Component.onCompleted: {
    VirtualKeyboardSettings.styleName = "MyStyle"
}
```

### 方法 2: アプリケーションレベルでのフォント登録

スタイルを変更せず、デフォルトスタイルの `fontFamily` が参照する名前のフォントをシステムに登録する:

```cpp
// main.cpp
#include <QFontDatabase>
#include <QGuiApplication>

int main(int argc, char *argv[]) {
    QGuiApplication app(argc, argv);

    // フォントを登録
    int id = QFontDatabase::addApplicationFont(":/fonts/Sans.ttf");
    if (id == -1) {
        qWarning() << "Failed to load font";
    }

    // QML エンジン生成...
}
```

### 方法 3: プラットフォームレベルのフォント置換

Windows の場合、レジストリでフォント置換を設定:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\FontSubstitutes
"Sans" = "Meiryo"
```

Linux の場合、fontconfig で設定:

```xml
<!-- /etc/fonts/local.conf -->
<fontconfig>
    <alias>
        <family>Sans</family>
        <prefer>
            <family>Noto Sans JP</family>
        </prefer>
    </alias>
</fontconfig>
```

---

## 10. 関連する環境変数・API 一覧

### 10.1 環境変数

| 環境変数 | 説明 | 例 |
|---|---|---|
| `QT_VIRTUALKEYBOARD_STYLE` | キーボードスタイルを選択 | `"default"`, `"retro"`, `"MyStyle"` |
| `QT_VIRTUALKEYBOARD_LAYOUT_PATH` | カスタムレイアウトディレクトリ | `/opt/keyboard/layouts/` |
| `QT_QPA_FONTDIR` | プラットフォームフォントディレクトリ | `/usr/share/fonts/truetype/` |
| `QT_IM_MODULE` | 入力メソッド選択 | `"qtvirtualkeyboard"` |

### 10.2 VirtualKeyboardSettings プロパティ

| プロパティ | 説明 |
|---|---|
| `styleName` | アクティブなキーボードスタイルの取得/設定 |
| `locale` | ロケールの取得/設定 |
| `activeLocales` | アクティブな入力ロケールの一覧 |
| `fullScreenMode` | 全画面入力モードの有効/無効 |
| `hwrTimeoutForAlphabetic` | アルファベット手書き認識のタイムアウト |
| `hwrTimeoutForCJK` | CJK 手書き認識のタイムアウト |

### 10.3 フォント関連 QML API

| API | 説明 |
|---|---|
| `FontLoader` | QML でフォントファイルを読み込む |
| `Qt.font()` | JavaScript でフォントオブジェクトを生成 |
| `Qt.fontFamilies()` | 利用可能なフォントファミリー一覧を取得 |
| `Text.font` | テキスト要素のフォント設定 |

### 10.4 フォント関連 C++ API

| API | 説明 |
|---|---|
| `QFontDatabase::addApplicationFont(path)` | アプリケーションフォントを登録 |
| `QFontDatabase::applicationFontFamilies(id)` | 登録フォントのファミリー名を取得 |
| `QFontDatabase::families()` | 利用可能な全フォントファミリーを列挙 |
| `QFontDatabase::removeApplicationFont(id)` | 登録フォントを削除 |
| `QFont::setFamily(family)` | フォントファミリーを設定 |
| `QFont::setPixelSize(size)` | ピクセルサイズを設定 |

---

## 11. 既知の問題と注意点

### 11.1 "Sans" フォントが Windows で見つからない問題

**症状**: デフォルトスタイルの `fontFamily: "Sans"` が Windows 上でフォントマッチングに失敗し、`dwrite` 例外やフォントが正しく表示されない。

**原因**: Windows には "Sans" という名前のフォントが標準で存在しない。Linux では fontconfig が "Sans" をサンセリフフォントに解決するが、Windows にはこの仕組みがない。

**対策**:
- カスタムスタイルで明示的なフォントファミリーを指定（例: `"Segoe UI"`, `"Meiryo"`）
- Windows レジストリでフォント置換を設定
- `QFontDatabase::addApplicationFont()` で "Sans" という名前のフォントを登録

### 11.2 組込みシステムでの CJK フォント不足

**症状**: 中国語・日本語・韓国語の変換候補が表示されない、または豆腐文字（□）になる。

**対策**:
- CJK フォント（Noto Sans CJK 等）をデプロイ
- `QT_QPA_FONTDIR` 環境変数で正しいディレクトリを指定
- ランタイムで `QFontDatabase::families()` を使って利用可能なフォントを確認

### 11.3 FontLoader の相対パスの問題

**症状**: `FontLoader` で相対パスを指定するとフォントが読み込めない。

**原因**: 相対パスは QML ファイルではなく実行ファイルの位置を基準に解決される場合がある。

**対策**: 常に `qrc:` プロトコルまたは絶対パスを使用する。

```qml
// 正しい
FontLoader { source: "qrc:/fonts/MyFont.ttf" }

// 問題が起きうる
FontLoader { source: "fonts/MyFont.ttf" }
```

### 11.4 QML エンジン生成後のフォント登録

**症状**: `QFontDatabase::addApplicationFont()` を QML エンジン生成後に呼ぶと、フォントが QML コンポーネントから参照できないことがある。

**対策**: 必ず QML エンジン生成**前**にフォントを登録する。

```cpp
int main(int argc, char *argv[]) {
    QGuiApplication app(argc, argv);

    // ✓ QML エンジン生成前に登録
    QFontDatabase::addApplicationFont(":/fonts/MyFont.ttf");

    QQmlApplicationEngine engine;
    engine.load(QUrl("qrc:/main.qml"));
    // ...
}
```

### 11.5 CJK グリフテーブルの重複読み込み

**症状**: キーボードの候補リストとアプリケーションのテキストフィールドで異なるフォントを使用すると、大きな CJK グリフテーブルが二重に読み込まれ、メモリ消費が増大する。

**対策**: キーボードの `selectionListDelegate` とアプリケーション側で同じフォントを使用する。

---

## 12. 参考資料

- [Qt Virtual Keyboard Overview (Qt 6)](https://doc.qt.io/qt-6/qtvirtualkeyboard-overview.html)
- [KeyboardStyle QML Type (Qt 6)](https://doc.qt.io/qt-6/qml-qtquick-virtualkeyboard-styles-keyboardstyle.html)
- [KeyboardStyle QML Type (Qt 5.15)](https://doc.qt.io/qt-5/qml-qtquick-virtualkeyboard-styles-keyboardstyle.html)
- [VirtualKeyboardSettings QML Type (Qt 6)](https://doc.qt.io/qt-6/qml-qtquick-virtualkeyboard-settings-virtualkeyboardsettings.html)
- [FontLoader QML Type (Qt 6)](https://doc.qt.io/qt-6/qml-qtquick-fontloader.html)
- [KeyPanel QML Type (Qt 6)](https://doc.qt.io/qt-6/qml-qtquick-virtualkeyboard-styles-keypanel.html)
- [BaseKey QML Type (Qt 6)](https://doc.qt.io/qt-6/qml-qtquick-virtualkeyboard-components-basekey.html)
- [Handwriting Recognition (Qt 6)](https://doc.qt.io/qt-6/handwriting.html)
- [TraceCanvas QML Type (Qt 5.15)](https://doc.qt.io/qt-5/qml-qtquick-virtualkeyboard-styles-tracecanvas.html)
- [Qt Virtual Keyboard Technical Guide (Qt 5.15)](https://doc.qt.io/qt-5/technical-guide.html)
- [Qt Virtual Keyboard GitHub Repository](https://github.com/qt/qtvirtualkeyboard)
- [Qt for MCUs Virtual Keyboard Overview](https://doc.qt.io/QtForMCUs/qul-virtual-keyboard-overview.html)
