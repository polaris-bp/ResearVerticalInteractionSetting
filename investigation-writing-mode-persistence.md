# Writing-Mode 親子継承・上書き構造の調査レポート

## 問題の背景

以下の構造を実現したいという要件があります：
- **親（アイテム）の文字列方向設定が子（インタラクション）にも反映される**
- **子の設定をしたときは子の設定のみが更新される**

この「継承と上書き」の仕組みが標準インタラクションでどのように実現されているかを調査しました。

---

## 調査結果の要約

### 核心的な発見

**設定値の保存方法:**
- インタラクションのwriting-mode設定は「クラス属性」として保存される
- `writing-mode-vertical-rl` → 縦書き（明示的設定）
- `writing-mode-horizontal-tb` → 横書き（明示的設定）
- **クラスなし** → アイテムの設定を継承（デフォルト）

**アイテム変更時の動作:**
- インタラクションに「アイテムの設定値をコピー」するのではない
- インタラクションの「明示的設定を削除して継承させる」という設計

---

## 設定値の継承メカニズム詳細

### 1. 設定値の保存形式

```
インタラクションの設定状態:
├─ class="writing-mode-vertical-rl"  → 明示的に縦書き
├─ class="writing-mode-horizontal-tb" → 明示的に横書き
└─ class=""（またはclass属性なし）    → アイテムの設定を継承
```

### 2. アイテム変更時の処理フロー

```
ユーザーがアイテムのwriting-modeを変更
    ↓
Active.js: $itemBody.trigger('item-writing-mode-changed')
    ↓
Widget.js: イベントを受信
    ↓
インタラクションのクラスを削除（addClass ではなく removeClass のみ）
    this.element.removeClass('writing-mode-vertical-rl');
    this.element.removeClass('writing-mode-horizontal-tb');
    ↓
インタラクションは「クラスなし」状態になる
    ↓
★ 新しいクラスは追加しない ★
    ↓
以降、継承メカニズムが機能する
```

### 3. 継承が機能する2つの場所

#### 3.1 CSS継承（表示・レンダリング）

```css
/* CSSのwriting-modeプロパティは親要素から継承される */
.qti-itemBody.writing-mode-vertical-rl {
    writing-mode: vertical-rl;
}

/* インタラクションにクラスがなければ、親の設定を継承 */
.qti-choiceInteraction {
    /* writing-mode: inherit; (デフォルト) */
}
```

#### 3.2 フォーム表示（UI）

```javascript
// Question.js: toggleVerticalWritingModeByLang
if (interaction.hasClass(writingModeVerticalRlClass)) {
    isVertical = true;  // 明示的に縦書き
} else if (interaction.hasClass(writingModeHorizontalTbClass)) {
    isVertical = false; // 明示的に横書き
} else {
    // ★ クラスなし = アイテムの設定を継承
    isVertical = isItemVertical;
}

// フォームのラジオボタンを設定
$form.find('input[name="writingMode"][value="vertical"]').prop('checked', isVertical);
$form.find('input[name="writingMode"][value="horizontal"]').prop('checked', !isVertical);
```

---

## ソースコード詳細

### Widget.js（アイテム変更時のイベントハンドラ）

**ファイル:** `extension-tao-itemqti/views/js/qtiCreator/widgets/interactions/choiceInteraction/Widget.js`

```javascript
define([
    'taoQtiItem/qtiCreator/widgets/interactions/Widget',
    'taoQtiItem/qtiCreator/widgets/interactions/choiceInteraction/states/states',
    'taoQtiItem/qtiCommonRenderer/helpers/sizeAdapter',
    'taoQtiItem/qtiCreator/widgets/static/helpers/itemScrollingMethods',
    'taoQtiItem/qtiCommonRenderer/helpers/verticalWriting'
], function (Widget, states, sizeAdapter, itemScrollingMethods, verticalWriting) {
    'use strict';

    var ChoiceInteractionWidget = Widget.clone();

    ChoiceInteractionWidget.initCreator = function () {
        this.registerStates(states);
        Widget.initCreator.call(this);

        // ...

        const $itemBody = this.$container.closest('.qti-itemBody');
        $itemBody.on('item-writing-mode-changed', () => {
            // ★ クラスを削除するだけ（追加はしない）
            this.element.removeClass(verticalWriting.WRITING_MODE_VERTICAL_RL_CLASS);
            this.element.removeClass(verticalWriting.WRITING_MODE_HORIZONTAL_TB_CLASS);
            itemScrollingMethods.wrapContent(this, false, 'interaction');
        });
    };

    return ChoiceInteractionWidget;
});
```

### Question.js（フォーム初期化とコールバック）

**ファイル:** `extension-tao-itemqti/views/js/qtiCreator/widgets/interactions/choiceInteraction/states/Question.js`

#### フォーム初期化時の継承ロジック

```javascript
const toggleVerticalWritingModeByLang = (widget, $form, interaction) =>
    verticalWritingEditing
        .checkItemWritingMode(widget)
        .then(({ isVerticalSupported, isItemVertical }) => {
            // アイテムの設定を保存
            $form.data('isItemVertical', isItemVertical);

            let isVertical = null;

            // 優先順位:
            // 1. インタラクションの明示的設定（クラスあり）
            // 2. アイテムの設定を継承（クラスなし）
            if (interaction.hasClass(writingModeVerticalRlClass)) {
                isVertical = true;
            } else if (interaction.hasClass(writingModeHorizontalTbClass)) {
                isVertical = false;
            } else {
                // ★ クラスなし = アイテムの設定を継承
                isVertical = isItemVertical;
            }

            // フォームのラジオボタンを更新
            $form.find('input[name="writingMode"][value="vertical"]').prop('checked', isVertical);
            $form.find('input[name="writingMode"][value="horizontal"]').prop('checked', !isVertical);
        });
```

#### ユーザーが設定を変更したときのコールバック

```javascript
callbacks.writingMode = function (i, mode) {
    let isScrolling = false;

    // 既存のクラスを削除
    interaction.removeClass(writingModeVerticalRlClass);
    interaction.removeClass(writingModeHorizontalTbClass);

    // ★ アイテムと異なる設定の場合のみクラスを追加
    if (mode === 'vertical' && !$form.data('isItemVertical')) {
        // アイテムが横書き、インタラクションを縦書きに → クラス追加
        interaction.addClass(writingModeVerticalRlClass);
        isScrolling = true;
    } else if (mode === 'horizontal' && $form.data('isItemVertical')) {
        // アイテムが縦書き、インタラクションを横書きに → クラス追加
        interaction.addClass(writingModeHorizontalTbClass);
        isScrolling = true;
    }
    // ★ アイテムと同じ設定の場合はクラスを追加しない（継承させる）

    itemScrollingMethods.initSelect($form, isScrolling);
    itemScrollingMethods.wrapContent(widget, isScrolling, 'interaction');
};
```

---

## Widgetライフサイクル

### 初期化フロー

```
アイテムを開く
    ↓
Renderer が全要素をレンダリング
    ↓
各インタラクションに対して:
    CreatorChoiceInteraction.render()
        ↓
    ChoiceInteractionWidget.build(interaction, container, ...)
        ↓
    Widget.init() が呼ばれる
        ↓
    initCreator() が呼ばれる（非同期）
        ↓
    'item-writing-mode-changed' イベントリスナーが登録される
```

### 重要なポイント

**アイテムを開いた時点で、全てのインタラクションのWidgetは即座に初期化されます。**

インタラクションを「選択」する/しないに関係なく、イベントリスナーは**アイテムが開いている間は常にアクティブ**です。

---

## 設定値の永続化

### removeClass() がどのように保存されるか

```
removeClass() 呼び出し
    ↓
内部で this.attr('class', newValue) を呼び出し
    ↓
editable mixin の attr() がイベントを発火
    $(document).trigger('attributeChange.qti-widget', {
        element: this,
        key: 'class',
        value: newValue
    });
    ↓
qtiXmlRenderer がこのイベントを検知
    ↓
QTI XMLに自動保存
```

---

## 動作シナリオ

### シナリオ1: インタラクションが継承状態 → アイテム変更

```
初期状態:
  アイテム: 横書き
  インタラクション: クラスなし（継承 → 横書き）

操作: アイテムを縦書きに変更

処理:
  1. Active.js: item-writing-mode-changed イベント発火
  2. Widget.js: removeClass() 呼び出し（すでにクラスなしなので変化なし）

結果:
  アイテム: 縦書き
  インタラクション: クラスなし（継承 → 縦書き）
```

### シナリオ2: インタラクションが明示的設定あり → アイテム変更

```
初期状態:
  アイテム: 横書き
  インタラクション: class="writing-mode-vertical-rl"（明示的に縦書き）

操作: アイテムを縦書きに変更

処理:
  1. Active.js: item-writing-mode-changed イベント発火
  2. Widget.js: removeClass('writing-mode-vertical-rl') 呼び出し
  3. インタラクションのクラスが削除される
  4. attributeChange.qti-widget イベント発火
  5. QTI XMLに保存

結果:
  アイテム: 縦書き
  インタラクション: クラスなし（継承 → 縦書き）
```

### シナリオ3: インタラクションのフォームを開く

```
状態:
  アイテム: 縦書き
  インタラクション: クラスなし

操作: インタラクションを選択してフォームを開く

処理:
  1. Question.js: toggleVerticalWritingModeByLang() 呼び出し
  2. interaction.hasClass() チェック → 両方 false
  3. isItemVertical = true を使用
  4. フォームのラジオボタン: 縦書きが選択状態

結果:
  フォームには「縦書き」が表示される
  ★ この時点ではクラスは追加されない
```

---

## QTI XMLでの保存形式

### アイテムレベルの設定

```xml
<assessmentItem class="writing-mode-vertical-rl" ...>
    <itemBody>
        ...
    </itemBody>
</assessmentItem>
```

### インタラクションレベルの設定

```xml
<!-- 明示的設定がある場合（アイテムと異なる設定） -->
<choiceInteraction class="writing-mode-horizontal-tb" ...>
    ...
</choiceInteraction>

<!-- 継承の場合（クラスなし） -->
<choiceInteraction responseIdentifier="RESPONSE" ...>
    ...
</choiceInteraction>
```

---

## PCIでの実装方針

### 標準インタラクションと同様の動作を実現する場合

#### 1. Widget.js でイベントリスナーを登録

```javascript
PCIWidget.initCreator = function() {
    Widget.initCreator.call(this);

    const $itemBody = this.$container.closest('.qti-itemBody');
    $itemBody.on('item-writing-mode-changed', () => {
        // クラスを削除するだけ（追加はしない）
        this.element.removeClass('writing-mode-vertical-rl');
        this.element.removeClass('writing-mode-horizontal-tb');
    });
};
```

#### 2. Question.js で継承ロジックを実装

```javascript
// フォーム初期化時
const initWritingMode = (pciElement, isItemVertical) => {
    let isVertical;

    if (pciElement.hasClass('writing-mode-vertical-rl')) {
        isVertical = true;  // 明示的に縦書き
    } else if (pciElement.hasClass('writing-mode-horizontal-tb')) {
        isVertical = false; // 明示的に横書き
    } else {
        isVertical = isItemVertical; // ★ クラスなし = アイテムの設定を継承
    }

    return isVertical;
};
```

#### 3. コールバックでの最適化

```javascript
callbacks.writingMode = function(i, mode) {
    const isItemVertical = $form.data('isItemVertical');

    // 既存のクラスを削除
    pciElement.removeClass('writing-mode-vertical-rl');
    pciElement.removeClass('writing-mode-horizontal-tb');

    // ★ アイテムと異なる設定の場合のみクラスを追加
    if (mode === 'vertical' && !isItemVertical) {
        pciElement.addClass('writing-mode-vertical-rl');
    } else if (mode === 'horizontal' && isItemVertical) {
        pciElement.addClass('writing-mode-horizontal-tb');
    }
    // アイテムと同じ設定の場合はクラスを追加しない（継承させる）
};
```

---

## PCIでの実装上の懸念点

### 1. this.element の動作確認が必要

PCIの `this.element` が editable mixin を持ち、`attributeChange.qti-widget` イベントをトリガーするか要検証。

### 2. QTI XMLへのclass属性保存

PCIの `<customInteraction>` 要素に class 属性が正しく保存されるか要検証。

### 3. CSS継承

PCIの内部DOMにCSSの `writing-mode` が正しく継承されるか要検証。

---

## まとめ

### 標準インタラクションの設計思想

1. **クラスなし = 継承** という設計
2. アイテム変更時は**クラス削除のみ**（新しい値を書き込まない）
3. 継承は**CSS継承**と**フォームロジック**の両方で実現

### PCIで同等の実装をする場合

同じ設計パターンを適用可能ですが、以下の点を事前に検証することを推奨：

- [ ] `this.element.addClass/removeClass` が動作するか
- [ ] `attributeChange.qti-widget` イベントがトリガーされるか
- [ ] QTI XMLにclass属性が保存されるか
- [ ] CSS `writing-mode` がPCI内部に継承されるか

---

## 参考リンク

- [choiceInteraction Widget.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/interactions/choiceInteraction/Widget.js)
- [choiceInteraction Question.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/interactions/choiceInteraction/states/Question.js)
- [editable mixin (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/model/mixin/editable.js)
- [Element.js - addClass/removeClass (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiItem/core/Element.js)
