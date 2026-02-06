# Writing-Mode 親子継承・上書き構造の調査レポート

## 問題の背景

以下の構造を実現したいという要件があります：
- **親（アイテム）の文字列方向設定が子（インタラクション）にも反映される**
- **子の設定をしたときは子の設定のみが更新される**

この「継承と上書き」の仕組みが標準インタラクションでどのように実現されているかを調査しました。

---

## 調査結果の要約

### 核心的な発見

**標準インタラクション（choiceInteraction等）は、個別にwriting-mode設定を持っています。**

親子間の継承と上書きの構造が実装されていますが、**重要な制約**があります：

1. インタラクションに明示的な設定がない場合 → **アイテムの設定を継承**
2. インタラクションに明示的な設定がある場合 → **インタラクションの設定を優先**
3. **アイテムの設定が変更された場合** → **インタラクションの明示的設定がリセットされる**

### 重要な修正（以前の誤りを訂正）

**Widgetの初期化タイミングについて、以前の説明に誤りがありました。**

| 誤った理解 | 正しい理解 |
|-----------|-----------|
| Widgetはインタラクションを「選択」したときに初期化される | Widgetはアイテムを開いたときに即座に初期化される |
| インタラクション未選択時はイベントリスナーが動作しない | アイテムが開いている限り、イベントリスナーは常に動作する |

---

## Widgetライフサイクルの詳細解析

### 初期化フロー（ソースコード解析に基づく）

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
    イベントリスナーが登録される
```

**ソースコード（ChoiceInteraction.js）:**
```javascript
CreatorChoiceInteraction.render = function(interaction, options) {
    return ChoiceInteractionWidget.build(
        interaction,
        ChoiceInteraction.getContainer(interaction),
        this.getOption('interactionOptionForm'),
        this.getOption('responseOptionForm'),
        options
    );
};
```

**ソースコード（interactions/Widget.js）:**
```javascript
InteractionWidget.build = function(element, $container, $form, $responseForm, options) {
    return this.clone().init(element, $container, $form, $responseForm, options);
};
```

### 結論

**アイテムを開いた時点で、全てのインタラクションのWidgetは即座に初期化されます。**

インタラクションを「選択」する/しないに関係なく、イベントリスナーは**アイテムが開いている間は常にアクティブ**です。

---

## 実装の詳細

### 1. インタラクション個別のwriting-mode設定

**ファイル:** `extension-tao-itemqti/views/js/qtiCreator/widgets/interactions/choiceInteraction/states/Question.js`

#### writing-mode変更時のコールバック

```javascript
const writingModeVerticalRlClass = 'writing-mode-vertical-rl';
const writingModeHorizontalTbClass = 'writing-mode-horizontal-tb';

callbacks.writingMode = function (i, mode) {
    let isScrolling = false;

    // 既存のクラスを削除
    interaction.removeClass(writingModeVerticalRlClass);
    interaction.removeClass(writingModeHorizontalTbClass);

    // アイテムと異なる設定の場合のみ、クラスを追加
    if (mode === 'vertical' && !$form.data('isItemVertical')) {
        // アイテムが横書きで、インタラクションを縦書きにする場合
        interaction.addClass(writingModeVerticalRlClass);
        isScrolling = true;
    } else if (mode === 'horizontal' && $form.data('isItemVertical')) {
        // アイテムが縦書きで、インタラクションを横書きにする場合
        interaction.addClass(writingModeHorizontalTbClass);
        isScrolling = true;
    }
    // アイテムと同じ設定の場合はクラスを追加しない（継承）
};
```

### 2. アイテム変更時のリセット処理

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

        if (this.element.attr('orientation') === 'horizontal') {
            sizeAdapter.adaptSize(this);
        }

        // UIではクラスを表示しない（データのみに反映）
        this.$original
            .removeClass(verticalWriting.WRITING_MODE_VERTICAL_RL_CLASS)
            .removeClass(verticalWriting.WRITING_MODE_HORIZONTAL_TB_CLASS);

        // ★ 重要: このイベントリスナーはアイテムを開いた時点で登録される
        const $itemBody = this.$container.closest('.qti-itemBody');
        $itemBody.on('item-writing-mode-changed', () => {
            // アイテムのwriting-modeが変更されたら、インタラクションの設定をリセット
            this.element.removeClass(verticalWriting.WRITING_MODE_VERTICAL_RL_CLASS);
            this.element.removeClass(verticalWriting.WRITING_MODE_HORIZONTAL_TB_CLASS);
            itemScrollingMethods.wrapContent(this, false, 'interaction');
        });
    };

    return ChoiceInteractionWidget;
});
```

---

## 実際の動作フロー

### ユーザーが観察した現象の説明

```
手順1: アイテムを横書きで作成
└→ アイテム: class なし

手順2: choiceInteractionを追加

手順3: インタラクションを選択して縦書きに設定
└→ interaction.addClass('writing-mode-vertical-rl')
└→ インタラクション: class="writing-mode-vertical-rl"

手順4: インタラクション以外をクリック
└→ （何も変わらない。イベントリスナーは引き続きアクティブ）

手順5: アイテムを縦書きに変更
├→ $itemBody.trigger('item-writing-mode-changed')
├→ イベントリスナーが発火
├→ interaction.removeClass('writing-mode-vertical-rl') ← クラスが削除される！
├→ インタラクションはクラスなし
├→ CSS継承でアイテム（縦書き）の設定を継承
└→ 表示: 両方縦書き（見た目は変化なし）

手順6: アイテムを横書きに変更
├→ $itemBody.trigger('item-writing-mode-changed')
├→ イベントリスナーが発火
├→ インタラクションにはすでにクラスがない
├→ CSS継承でアイテム（横書き）の設定を継承
└→ 表示: 両方横書き ← インタラクションも横書きに見える！
```

### 重要なポイント

**手順5の時点でインタラクションのクラスは削除されています。**

見た目が縦書きのまま変わらなかったため気づきにくいですが、内部的には：
- インタラクションのクラス: `writing-mode-vertical-rl` → **削除**
- 表示: アイテムの縦書き設定をCSS継承

---

## 継承・上書き構造の実際の動作

### ケース別の動作

| ケース | 操作 | 結果 |
|--------|------|------|
| インタラクション設定なし | アイテム変更 | CSS継承で自動追従 |
| インタラクション設定あり | アイテム変更 | **クラス削除 → CSS継承で追従** |

**結論: アイテムのwriting-modeが変更されると、インタラクションの明示的設定は常にリセットされます。**

### 設計上の意図

この設計は以下を実現しています：

1. **親（アイテム）の設定が優先される**
   - アイテムの設定を変更すると、全てのインタラクションがそれに追従

2. **子（インタラクション）は一時的に上書き可能**
   - ただし、親の設定が変更されると子の設定はリセットされる

---

## PCIでの実装方針

### 標準インタラクションと同様の動作を実現する場合

#### 1. initCreatorでイベントリスナーを登録

```javascript
// PCI Widget.js相当
PCIWidget.initCreator = function() {
    Widget.initCreator.call(this);

    const $itemBody = this.$container.closest('.qti-itemBody');
    $itemBody.on('item-writing-mode-changed', () => {
        // アイテム変更時にPCIの明示的設定をリセット
        this.element.removeClass('writing-mode-vertical-rl');
        this.element.removeClass('writing-mode-horizontal-tb');
        // 必要に応じてコンテンツの再ラップ処理
    });
};
```

#### 2. PCI個別のwriting-mode設定UI

```javascript
// PCI Question.js相当
callbacks.writingMode = function(i, mode) {
    const isItemVertical = $form.data('isItemVertical');

    pciElement.removeClass('writing-mode-vertical-rl');
    pciElement.removeClass('writing-mode-horizontal-tb');

    // アイテムと異なる設定の場合のみクラスを追加
    if (mode === 'vertical' && !isItemVertical) {
        pciElement.addClass('writing-mode-vertical-rl');
    } else if (mode === 'horizontal' && isItemVertical) {
        pciElement.addClass('writing-mode-horizontal-tb');
    }
};
```

#### 3. 継承ロジック（フォーム初期化時）

```javascript
const initWritingMode = (pciElement, isItemVertical) => {
    if (pciElement.hasClass('writing-mode-vertical-rl')) {
        return true;  // 縦書き（明示的）
    } else if (pciElement.hasClass('writing-mode-horizontal-tb')) {
        return false; // 横書き（明示的）
    } else {
        return isItemVertical; // アイテムの設定を継承
    }
};
```

---

## クラス設定の最適化

標準インタラクションでは、**アイテムと同じ設定の場合はクラスを追加しない**という最適化が行われています。

```javascript
// アイテムと異なる設定の場合のみクラスを追加
if (mode === 'vertical' && !$form.data('isItemVertical')) {
    interaction.addClass(writingModeVerticalRlClass);  // アイテムが横書きの場合のみ
} else if (mode === 'horizontal' && $form.data('isItemVertical')) {
    interaction.addClass(writingModeHorizontalTbClass);  // アイテムが縦書きの場合のみ
}
// アイテムと同じ設定の場合はクラスを追加しない
```

**この最適化の利点:**
1. QTI XMLのサイズを最小化（不要なクラスを保存しない）
2. アイテムの設定変更時、クラスがなければ自動的に追従

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

### インタラクションレベルの設定（上書きの場合のみ）

```xml
<choiceInteraction class="writing-mode-horizontal-tb" ...>
    <!-- アイテムが縦書きだが、このインタラクションは横書きに上書き -->
</choiceInteraction>
```

---

## 実装チェックリスト

### PCI側の実装

- [ ] Widget.initCreatorで `item-writing-mode-changed` イベントリスナーを登録
- [ ] イベント受信時にwriting-modeクラスをリセット
- [ ] フォームにwriting-mode設定UIを追加
- [ ] アイテムのwriting-modeを判定するヘルパーを使用
- [ ] アイテムと異なる設定の場合のみクラスを追加（最適化）
- [ ] 継承ロジック（クラスがない場合はアイテムの設定を使用）

### TAO側の実装（必要に応じて）

- [ ] PCI CreatorでverticalWritingEditingヘルパーを使用可能にする
- [ ] PCI要素のclass属性をQTI XMLに保存できるようにする

---

## 参考リンク

- [choiceInteraction Widget.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/interactions/choiceInteraction/Widget.js)
- [choiceInteraction Question.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/interactions/choiceInteraction/states/Question.js)
- [interactions/Widget.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/interactions/Widget.js)
- [ChoiceInteraction Renderer (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/renderers/interactions/ChoiceInteraction.js)
