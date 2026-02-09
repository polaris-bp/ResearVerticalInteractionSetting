# Writing-Mode 継承メカニズム調査レポート

## 概要

標準インタラクション（choiceInteraction等）における文字列方向（writing-mode）設定の継承メカニズムを調査しました。

---

## 設定値の保存形式

インタラクションのwriting-mode設定は**クラス属性**として保存されます。

| クラス属性 | 意味 |
|-----------|------|
| `class="writing-mode-vertical-rl"` | 明示的に縦書き |
| `class="writing-mode-horizontal-tb"` | 明示的に横書き |
| クラスなし | アイテムの設定を継承 |

---

## 継承の仕組み

### 基本原則

**「クラスなし」=「親（アイテム）に従う」**

### 継承が機能する場所

| 場所 | 機能 |
|------|------|
| CSS | `writing-mode`プロパティが親要素から子要素に継承される |
| フォーム表示 | Question.jsがクラスを確認し、なければアイテムの設定を使用 |

---

## インタラクションの設定保存ルール

### いつ保存されるか

| アイテムの設定 | インタラクションの選択 | 保存される？ | 理由 |
|---------------|---------------------|-------------|------|
| 横書き | 横書き | ❌ しない | 継承で同じ結果になる |
| 横書き | 縦書き | ✅ する | アイテムと異なる |
| 縦書き | 横書き | ✅ する | アイテムと異なる |
| 縦書き | 縦書き | ❌ しない | 継承で同じ結果になる |

### コードでの実装

```javascript
// Question.js: callbacks.writingMode

// 1. まず既存のクラスを削除
interaction.removeClass('writing-mode-vertical-rl');
interaction.removeClass('writing-mode-horizontal-tb');

// 2. アイテムと異なる場合のみクラスを追加
if (mode === 'vertical' && !isItemVertical) {
    // アイテムが横書き、選択が縦書き → 保存する
    interaction.addClass('writing-mode-vertical-rl');
}
else if (mode === 'horizontal' && isItemVertical) {
    // アイテムが縦書き、選択が横書き → 保存する
    interaction.addClass('writing-mode-horizontal-tb');
}
// それ以外（アイテムと同じ設定）→ 何も追加しない（継承させる）
```

---

## 処理フロー

### 1. アイテムのwriting-modeが変更されたとき

```
ユーザーがアイテムの設定を変更
    ↓
Active.js: イベント発火
    $itemBody.trigger('item-writing-mode-changed')
    ↓
Widget.js: イベントを受信してクラスを削除
    this.element.removeClass('writing-mode-vertical-rl');
    this.element.removeClass('writing-mode-horizontal-tb');
    ↓
インタラクションは「クラスなし」状態になる
    ↓
CSS継承でアイテムの設定に従う
```

**ポイント:** 新しいクラスは追加しない。クラスを削除するだけで継承が機能する。

### 2. インタラクションのフォームを開いたとき

```
ユーザーがインタラクションを選択
    ↓
Question.js: クラスをチェック

if (interaction.hasClass('writing-mode-vertical-rl')) {
    → ラジオボタン「縦書き」を選択
}
else if (interaction.hasClass('writing-mode-horizontal-tb')) {
    → ラジオボタン「横書き」を選択
}
else {
    → アイテムの設定に基づいてラジオボタンを選択
}
```

**ポイント:** フォーム表示時にはクラスを追加しない。ラジオボタンの表示のみ。

### 3. ユーザーがインタラクションの設定を変更したとき

```
ユーザーがラジオボタンを変更
    ↓
Question.js: callbacks.writingMode が実行
    ↓
1. 既存のクラスを削除
2. アイテムと異なる設定の場合のみクラスを追加
```

---

## 具体例

### 例1: インタラクションが継承状態でアイテムを変更

```
【初期状態】
アイテム: 横書き
インタラクション: クラスなし → 横書き（継承）

【操作】アイテムを縦書きに変更

【処理】
Widget.js: removeClass() → 変化なし（すでにクラスなし）

【結果】
アイテム: 縦書き
インタラクション: クラスなし → 縦書き（継承）
```

### 例2: インタラクションに明示的設定がある状態でアイテムを変更

```
【初期状態】
アイテム: 横書き
インタラクション: class="writing-mode-vertical-rl" → 縦書き（明示的）

【操作】アイテムを縦書きに変更

【処理】
Widget.js: removeClass('writing-mode-vertical-rl')
→ クラスが削除される

【結果】
アイテム: 縦書き
インタラクション: クラスなし → 縦書き（継承）
```

### 例3: インタラクションで親と異なる設定を選択

```
【初期状態】
アイテム: 縦書き
インタラクション: クラスなし → 縦書き（継承）

【操作】インタラクションのフォームで「横書き」を選択

【処理】
Question.js: アイテム（縦書き）と選択（横書き）が異なる
→ class="writing-mode-horizontal-tb" を追加

【結果】
アイテム: 縦書き
インタラクション: class="writing-mode-horizontal-tb" → 横書き（明示的）
```

### 例4: インタラクションで親と同じ設定を選択

```
【初期状態】
アイテム: 縦書き
インタラクション: class="writing-mode-horizontal-tb" → 横書き（明示的）

【操作】インタラクションのフォームで「縦書き」を選択

【処理】
Question.js: アイテム（縦書き）と選択（縦書き）が同じ
→ クラスを追加しない

【結果】
アイテム: 縦書き
インタラクション: クラスなし → 縦書き（継承）
```

---

## Widgetの初期化タイミング

```
アイテムを開く
    ↓
全インタラクションのWidgetが初期化される
    ↓
initCreator() で 'item-writing-mode-changed' イベントリスナーが登録される
```

**重要:** Widgetはアイテムを開いた時点で初期化されます。インタラクションを「選択」するかどうかに関係なく、イベントリスナーは常にアクティブです。

---

## 設定値の永続化

```
removeClass() / addClass() 呼び出し
    ↓
内部で this.attr('class', value) を呼び出し
    ↓
editable mixin が attributeChange.qti-widget イベントを発火
    ↓
QTI XMLに自動保存
```

---

## QTI XMLでの保存形式

```xml
<!-- アイテムの設定 -->
<assessmentItem class="writing-mode-vertical-rl">
    <itemBody>

        <!-- 継承の場合（クラスなし） -->
        <choiceInteraction responseIdentifier="RESPONSE">
            ...
        </choiceInteraction>

        <!-- 明示的設定の場合 -->
        <choiceInteraction class="writing-mode-horizontal-tb" responseIdentifier="RESPONSE">
            ...
        </choiceInteraction>

    </itemBody>
</assessmentItem>
```

---

## 関連ソースコード

| ファイル | 役割 |
|----------|------|
| `choiceInteraction/Widget.js` | アイテム変更時にクラスを削除 |
| `choiceInteraction/states/Question.js` | フォーム表示とユーザー操作の処理 |
| `item/states/Active.js` | アイテム設定変更時にイベント発火 |
| `qtiCreator/model/mixin/editable.js` | attr()でattributeChangeイベント発火 |

---

## PCIへの適用

### 実装に必要な要素

1. **Widget.js**: `item-writing-mode-changed`イベントリスナーを登録し、クラスを削除
2. **Question.js**: クラスを確認してフォーム表示、アイテムと異なる場合のみクラスを追加

### 検証が必要な項目

- [ ] PCIの`this.element.addClass/removeClass`が動作するか
- [ ] PCIの`<customInteraction>`要素にclass属性が保存されるか
- [ ] CSS `writing-mode`がPCI内部に継承されるか
