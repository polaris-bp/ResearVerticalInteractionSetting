# 標準インタラクション Writing-Mode 継承メカニズム調査レポート

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

## ソースコード詳細

### Widget.js（イベントリスナー登録）

```javascript
// taoQtiItem/views/js/qtiCreator/widgets/interactions/choiceInteraction/Widget.js

initCreator: function() {
    this.registerStates(states);
    Widget.initCreator.call(this);

    var self = this;
    var $itemBody = this.$container.closest('.qti-itemBody');

    // アイテムのwriting-mode変更を監視
    $itemBody.on('item-writing-mode-changed', function() {
        // クラスを削除して継承状態にする
        self.element.removeClass(verticalWriting.WRITING_MODE_VERTICAL_RL_CLASS);
        self.element.removeClass(verticalWriting.WRITING_MODE_HORIZONTAL_TB_CLASS);

        // スクロールコンテンツの再ラップ
        itemScrollingMethods.wrapContent(self, false, 'interaction');
    });
}
```

### Question.js（フォーム処理）

```javascript
// taoQtiItem/views/js/qtiCreator/widgets/interactions/choiceInteraction/states/Question.js

// フォーム初期化時
verticalWritingEditing.checkItemWritingMode(widget)
    .then(function(result) {
        var isItemVertical = result.isItemVertical;
        $form.data('isItemVertical', isItemVertical);

        // インタラクションの設定を判定
        var isVertical;
        if (interaction.hasClass(writingModeVerticalRlClass)) {
            isVertical = true;  // 明示的に縦書き
        } else if (interaction.hasClass(writingModeHorizontalTbClass)) {
            isVertical = false;  // 明示的に横書き
        } else {
            isVertical = isItemVertical;  // アイテムから継承
        }

        // ラジオボタンを設定
        $form.find('input[name="writingMode"][value="vertical"]')
            .prop('checked', isVertical);
        $form.find('input[name="writingMode"][value="horizontal"]')
            .prop('checked', !isVertical);
    });

// コールバック（ユーザー操作時）
var callbacks = {
    writingMode: function(interaction, mode) {
        var isItemVertical = $form.data('isItemVertical');

        // まず既存のクラスを削除
        interaction.removeClass(writingModeVerticalRlClass);
        interaction.removeClass(writingModeHorizontalTbClass);

        // アイテムと異なる設定の場合のみクラスを追加
        if (mode === 'vertical' && !isItemVertical) {
            interaction.addClass(writingModeVerticalRlClass);
        } else if (mode === 'horizontal' && isItemVertical) {
            interaction.addClass(writingModeHorizontalTbClass);
        }
        // 同じ設定の場合はクラスを追加しない（継承させる）
    }
};
```

### Active.js（イベント発火）

```javascript
// taoQtiItem/views/js/qtiCreator/widgets/item/states/Active.js

// アイテムのwriting-mode変更時
var callbacks = {
    writingMode: function(item, mode) {
        // ... アイテムの設定処理 ...

        // インタラクションに変更を通知
        $itemBody.trigger('item-writing-mode-changed');
    }
};
```

---

## まとめ

標準インタラクションのwriting-mode継承は以下の仕組みで機能します：

1. **クラス属性による状態管理**: クラスあり=明示的設定、クラスなし=継承
2. **CSS継承**: `writing-mode`プロパティは自然に親から子へ継承
3. **フォームロジック**: クラスがなければアイテムの設定を参照してUI表示
4. **イベント連携**: アイテム変更時は`item-writing-mode-changed`イベントでクラスを削除
5. **保存の最適化**: アイテムと同じ設定は保存しない（継承に任せる）
