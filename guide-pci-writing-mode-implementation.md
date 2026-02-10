# PCI Writing-Mode 実装ガイド

## 本ドキュメントの位置づけ

PCI（Portable Custom Interaction）に文字列方向（writing-mode）設定を追加するための実装ガイドです。
PCI ソースのみの変更で実現可能な2つの方式を記載します。

**対象読者:** PCI 開発者
**前提知識:** TAO QTI Creator の基本構造、PCI creator の開発経験

---

## 背景: なぜ PCI の実装が標準インタラクションと異なるのか

標準インタラクション（choiceInteraction 等）では、テンプレート `customInteraction.tpl` ではなく各インタラクション固有のテンプレートが `{{attributes.class}}` を出力するため、`addClass()` で設定したクラスが runtime の DOM に自然に反映されます。

しかし PCI が使用する `customInteraction.tpl`（TAO 本体）には `{{attributes.class}}` が**含まれていません**。

```html
<!-- customInteraction.tpl（TAO 本体・変更不可） -->
<div class="qti-interaction qti-customInteraction" data-serial="{{serial}}">
    {{{markup}}}
</div>
<!-- ↑ {{attributes.class}} がない → addClass() で設定したクラスが DOM に出ない -->
```

```html
<!-- choiceInteraction.tpl（参考・標準インタラクション） -->
<div class="qti-interaction qti-blockInteraction qti-choiceInteraction{{#if attributes.class}} {{attributes.class}}{{/if}}"
     data-serial="{{serial}}">
<!-- ↑ {{attributes.class}} がある → addClass() のクラスが DOM に反映される -->
```

このため、PCI では独自の方法で writing-mode 情報を runtime に届ける必要があります。

---

## TAO API の動作確認済み事項

本ガイドの実装は、以下の TAO API の動作を実際のソースコードで確認した上で設計しています。

### addClass() / removeClass() / hasClass()

**定義:** `tao-item-runner-qti-fe/src/qtiItem/core/Element.js`

```javascript
// addClass の実装（TAO 本体）
addClass: function (className) {
    var clazz = this.attr('class') || '';
    if (!_containClass(clazz, className)) {
        this.attr('class', clazz + (clazz.length ? ' ' : '') + className);
    }
}
```

- customInteraction 要素でも正常に動作する
- 内部で `this.attr('class', value)` を呼ぶ
- editable mixin が `attr()` をオーバーライドしており、`attributeChange.qti-widget` イベントが発火する
- このイベントにより**保存が自動的にトリガーされる**

### prop()

**定義:** `tao-item-runner-qti-fe/src/qtiItem/mixin/CustomElement.js`

```javascript
// prop の実装（TAO 本体）
prop: function (name, value) {
    if (name) {
        if (value !== undefined) {
            this.properties[name] = value;
        } else {
            // value が undefined の場合は getter として動作
            if (typeof name === 'string') {
                return this.properties[name];
            }
        }
    }
    return this;
}
```

**注意すべき挙動:**

| 呼び出し | 期待する動作 | 実際の動作 |
|---------|------------|-----------|
| `prop('writingMode', 'vertical')` | setter | **setter として動作する** |
| `prop('writingMode', undefined)` | 削除 | **getter として動作する（削除されない）** |
| `prop('writingMode')` | getter | **getter として動作する** |

`prop(name, undefined)` は `value !== undefined` が `false` になるため setter に入らず、getter として動作します。**プロパティの削除には `delete interaction.properties['writingMode']` を使用してください。**

### removeProp()

**定義:** `tao-item-runner-qti-fe/src/qtiItem/mixin/CustomElement.js`

```javascript
// removeProp の実装（TAO 本体）- バグあり
removeProp: function (propNames) {
    var _this = this;
    if (typeof propNames === 'string') {
        propNames = [propNames];
    }
    _.forEach(propNames, function (propName) {
        delete _this.attributes[propName];  // ← this.properties ではなく this.attributes を削除
    });
    return this;
}
```

**このメソッドにはバグがあり、`this.properties` ではなく `this.attributes` を削除します。使用しないでください。**

### updateMarkup()

**定義:** `extension-tao-itemqti/views/js/qtiCreator/model/helper/portableElement.js`

```javascript
// updateMarkup の実装（TAO 本体）
updateMarkup: function() {
    this.markup = this.renderMarkup();
}

renderMarkup: function() {
    var creatorModule = registry.getCreator(this.typeIdentifier).module;
    var markupTpl = creatorModule.getMarkupTemplate();     // ← PCI の markup.tpl
    var markupData = this.getDefaultMarkupTemplateData();   // ← { responseIdentifier }
    if (_.isFunction(creatorModule.getMarkupData)) {
        markupData = creatorModule.getMarkupData(this, markupData);  // ← PCI の getMarkupData()
    }
    return markupTpl(markupData);
}
```

- PCI の `getMarkupTemplate()` と `getMarkupData()` に処理を委譲する
- テンプレートを再レンダリングし、結果の HTML 文字列で `interaction.markup` を上書きする
- **テンプレート全体を再レンダリングするため、動的データ（prompt 等）は `getMarkupData()` で渡す必要がある**
- イベントは発火しない（保存トリガーにはならない）
- mathEntryInteraction が prompt 変更時に使用している既存パターン

### formElement.setChangeCallbacks()

**定義:** `extension-tao-itemqti/views/js/qtiCreator/widgets/helpers/formElement.js`

- ガイドや既存コードで `formElement.initDataBinding` と記載されている場合があるが、**実際の関数名は `setChangeCallbacks`**
- フォームの変更イベントを監視し、コールバックを呼び出す
- コールバック呼び出し後に**イベントは発火しない**（保存トリガーにはならない）

---

## 方式A: markup.tpl + updateMarkup()

### 概要

writing-mode のクラス情報を PCI の `markup.tpl` に埋め込み、`updateMarkup()` で `interaction.markup` を再生成する方式です。runtime では `<pci:markup>` に保存された HTML がそのまま DOM に展開されるため、クラスが DOM 上に存在し、CSS `writing-mode` が自然に継承します。

### 特徴

| 項目 | 内容 |
|------|------|
| 保存トリガー | `addClass()` / `removeClass()` が `attributeChange.qti-widget` を発火 |
| markup への反映 | `updateMarkup()` がテンプレートを再レンダリング |
| runtime での取得 | DOM のクラス属性から直接判定 |
| CSS 継承 | markup ルート要素にクラスがあるため自然に機能 |

### 変更対象ファイル

| ファイル | 変更内容 |
|---------|---------|
| `creator/tpl/markup.tpl` | ルート要素に `{{writingModeClass}}` を追加 |
| `imsPciCreator.js` | `getMarkupData()` で writing-mode クラスをテンプレートに渡す |
| `creator/widget/Widget.js` | `item-writing-mode-changed` ハンドラ |
| `creator/widget/states/Question.js` | フォーム処理とコールバック |
| `runtime/*.js` | DOM のクラスから writing-mode を判定 |

### QTI XML 保存形式

```xml
<customInteraction class="writing-mode-vertical-rl" responseIdentifier="RESPONSE">
    <pci:portableCustomInteraction customInteractionTypeIdentifier="myPci">
        <pci:properties>
            <!-- writing-mode は properties には保存しない -->
            <pci:entry key="otherProp">value</pci:entry>
        </pci:properties>
        <pci:markup>
            <div class="mathEntryInteraction writing-mode-vertical-rl">
                <!-- ↑ markup 内にもクラスが含まれる -->
                <div class="prompt">...</div>
                ...
            </div>
        </pci:markup>
    </pci:portableCustomInteraction>
</customInteraction>
```

`class` 属性が `<customInteraction>` 要素と `<pci:markup>` 内の両方に記録されます。`<customInteraction>` 側は `addClass()` による保存トリガーの結果として自動的に付与されます。`<pci:markup>` 側が runtime で DOM に反映される実体です。

### 実装手順

#### 1. markup.tpl の変更

```html
<div class="mathEntryInteraction{{#if writingModeClass}} {{writingModeClass}}{{/if}}">
    <div class="prompt">{{{prompt}}}</div>
    <div class="math-entry">
        <div class="toolbar"></div>
        <div>
            <span class="math-entry-input" data-allow-copy="true"></span>
        </div>
    </div>
</div>
```

**変更点:** ルート要素の `class` に `{{#if writingModeClass}} {{writingModeClass}}{{/if}}` を追加。

#### 2. imsPciCreator.js の getMarkupData() を変更

```javascript
getMarkupData: function getMarkupData(pci, defaultData) {
    defaultData.prompt = pci.data('prompt');

    // writing-mode クラスをテンプレートに渡す
    var cls = pci.attr('class') || '';
    if (cls.indexOf('writing-mode-vertical-rl') !== -1) {
        defaultData.writingModeClass = 'writing-mode-vertical-rl';
    } else if (cls.indexOf('writing-mode-horizontal-tb') !== -1) {
        defaultData.writingModeClass = 'writing-mode-horizontal-tb';
    } else {
        defaultData.writingModeClass = '';
    }

    return defaultData;
}
```

**解説:** `pci.attr('class')` は Element モデルの `this.attributes['class']` を返します。`addClass()` / `removeClass()` で設定された値がここに格納されています。`updateMarkup()` 呼び出し時、この関数がテンプレートにデータを渡します。

#### 3. Widget.js の変更

```javascript
initCreator: function() {
    this.registerStates(states);
    Widget.initCreator.call(this);

    var self = this;
    var $itemBody = this.$container.closest('.qti-itemBody');

    // アイテムの writing-mode 変更を監視
    $itemBody.on('item-writing-mode-changed', function() {
        // クラスを削除して継承状態にする
        // removeClass() → attr('class', ...) → attributeChange 発火 → 保存トリガー
        self.element.removeClass('writing-mode-vertical-rl');
        self.element.removeClass('writing-mode-horizontal-tb');

        // markup を再生成（クラスなしの HTML になる）
        self.element.updateMarkup();
    });
}
```

**処理の流れ:**
1. `removeClass()` が `attr('class', ...)` を呼び、`attributeChange.qti-widget` が発火 → 保存がトリガーされる
2. `updateMarkup()` が `getMarkupData()` を呼び、`attr('class')` が空なので `writingModeClass` は空文字
3. `interaction.markup` がクラスなしの HTML に更新される
4. 保存時に `<pci:markup>` にクラスなしの HTML が出力される

#### 4. Question.js の変更

```javascript
define([
    'taoQtiItem/qtiCreator/widgets/states/lib/formElement',
    'taoQtiItem/qtiCreator/widgets/static/helpers/verticalWritingEditing'
], function(formElement, verticalWritingEditing) {
    'use strict';

    // writingMode 定数
    var WRITING_MODE_VERTICAL_RL_CLASS = 'writing-mode-vertical-rl';
    var WRITING_MODE_HORIZONTAL_TB_CLASS = 'writing-mode-horizontal-tb';

    var QuestionState = stateFactory.extend(Question, function() {
        var widget = this.widget;
        var interaction = widget.element;
        var $form = widget.$form;

        // --- フォーム初期化 ---
        verticalWritingEditing.checkItemWritingMode(widget)
            .then(function(result) {
                var isItemVertical = result.isItemVertical;
                $form.data('isItemVertical', isItemVertical);

                // 現在の設定を判定
                var isVertical;
                if (interaction.hasClass(WRITING_MODE_VERTICAL_RL_CLASS)) {
                    isVertical = true;   // 明示的に縦書き
                } else if (interaction.hasClass(WRITING_MODE_HORIZONTAL_TB_CLASS)) {
                    isVertical = false;  // 明示的に横書き
                } else {
                    isVertical = isItemVertical;  // 継承
                }

                // ラジオボタンを設定
                $form.find('input[name="writingMode"][value="vertical"]')
                    .prop('checked', isVertical);
                $form.find('input[name="writingMode"][value="horizontal"]')
                    .prop('checked', !isVertical);
            });

        // --- コールバック ---
        var callbacks = {
            writingMode: function(interaction, mode) {
                var isItemVertical = $form.data('isItemVertical');

                // 1. 既存のクラスを削除
                interaction.removeClass(WRITING_MODE_VERTICAL_RL_CLASS);
                interaction.removeClass(WRITING_MODE_HORIZONTAL_TB_CLASS);

                // 2. アイテムと異なる場合のみクラスを追加
                if (mode === 'vertical' && !isItemVertical) {
                    interaction.addClass(WRITING_MODE_VERTICAL_RL_CLASS);
                } else if (mode === 'horizontal' && isItemVertical) {
                    interaction.addClass(WRITING_MODE_HORIZONTAL_TB_CLASS);
                }
                // アイテムと同じ → クラスなし（継承）

                // 3. markup を再生成
                interaction.updateMarkup();
            }
        };

        formElement.setChangeCallbacks($form, interaction, callbacks);
    });

    return QuestionState;
});
```

**重要:** `formElement.setChangeCallbacks` を使用してください（`initDataBinding` ではありません）。

#### 5. PCI runtime の変更

```javascript
// PCI runtime の initialize 内
initialize: function(id, dom, properties) {
    // dom は PCI markup のルート要素
    // → <div class="mathEntryInteraction writing-mode-vertical-rl">

    // クラスから writing-mode を判定
    if (dom.classList.contains('writing-mode-vertical-rl')) {
        // 縦書き
    } else if (dom.classList.contains('writing-mode-horizontal-tb')) {
        // 横書き
    } else {
        // クラスなし → アイテムの設定を継承
        // アイテム側のクラスは .qti-itemBody に付与されている
        var itemBody = dom.closest('.qti-itemBody');
        if (itemBody && itemBody.classList.contains('writing-mode-vertical-rl')) {
            // アイテムが縦書き
        } else {
            // アイテムが横書き（デフォルト）
        }
    }
}
```

**注意:** `dom` パラメータが指す要素は PCI モデルにより異なります。

| PCI モデル | `dom` が指す要素 |
|-----------|----------------|
| Common/OAT PCI | markup のルート要素（`<div class="mathEntryInteraction ...">` 自体） |
| IMS PCI | `.qti-customInteraction` ラッパー要素（markup の親） |

IMS PCI の場合、markup ルートは `dom.querySelector('.mathEntryInteraction')` で取得してください。

### 処理フロー図

#### ユーザーが Question.js で writing-mode を変更したとき

```
ユーザーがラジオボタンを変更
    ↓
formElement.setChangeCallbacks のイベントハンドラが発火
    ↓
callbacks.writingMode(interaction, mode) が呼ばれる
    ↓
interaction.removeClass('writing-mode-vertical-rl')
interaction.removeClass('writing-mode-horizontal-tb')
    ↓ （内部で attr('class', value) → editable mixin → attributeChange.qti-widget 発火）
    ↓
条件に応じて interaction.addClass('writing-mode-vertical-rl')
    ↓ （内部で attr('class', value) → editable mixin → attributeChange.qti-widget 発火）
    ↓
interaction.updateMarkup()
    ↓
    ├─ getMarkupData(pci, defaultData) が呼ばれる
    │   ├─ pci.attr('class') → 'writing-mode-vertical-rl'
    │   └─ defaultData.writingModeClass = 'writing-mode-vertical-rl'
    │
    ├─ markup.tpl がレンダリングされる
    │   └─ <div class="mathEntryInteraction writing-mode-vertical-rl">...</div>
    │
    └─ interaction.markup が新しい HTML 文字列に更新される
         ↓
    保存時に QTI XML に出力:
    ├─ <customInteraction class="writing-mode-vertical-rl"> （attr 由来）
    └─ <pci:markup><div class="mathEntryInteraction writing-mode-vertical-rl">... （markup 由来）
```

#### アイテムの writing-mode が変更されたとき

```
ユーザーがアイテムの設定を変更
    ↓
Active.js: $itemBody.trigger('item-writing-mode-changed')
    ↓
Widget.js: イベントを受信
    ↓
self.element.removeClass('writing-mode-vertical-rl')
self.element.removeClass('writing-mode-horizontal-tb')
    ↓ （attributeChange.qti-widget 発火 → 保存トリガー）
    ↓
self.element.updateMarkup()
    ↓
    ├─ getMarkupData: attr('class') → '' → writingModeClass = ''
    ├─ markup.tpl: <div class="mathEntryInteraction">...</div>
    └─ interaction.markup 更新
         ↓
    保存時: <pci:markup> にクラスなしの HTML が出力
         ↓
    runtime: クラスなし → アイテムの設定を継承
```

#### runtime（テスト配信時）

```
QTI XML からアイテムを読み込み
    ↓
customInteraction.tpl でレンダリング
    ↓
<div class="qti-interaction qti-customInteraction" data-serial="...">
    ← customInteraction.tpl は attr('class') を出力しない
    ↓
    {{{markup}}} が展開される
    ↓
    <div class="mathEntryInteraction writing-mode-vertical-rl">
        ← pci:markup に保存された HTML がそのまま展開される
        ← クラスが DOM 上に存在する
        ↓
        CSS writing-mode: vertical-rl が子要素に継承される ✓
    </div>
</div>
    ↓
PCI runtime の initialize(id, dom, properties) が呼ばれる
    ↓
dom.classList.contains('writing-mode-vertical-rl') → true
```

### 方式A の注意事項

#### updateMarkup() はテンプレート全体を再レンダリングする

`updateMarkup()` は `markup.tpl` を丸ごと再レンダリングし、`interaction.markup` を上書きします。テンプレート外で DOM に加えた動的な変更は `interaction.markup` には反映されません。

**対策:** テンプレートが使用する全ての動的データを `getMarkupData()` で渡してください。mathEntryInteraction の場合、`prompt` は `pci.data('prompt')` から取得しているため、`updateMarkup()` を呼んでも失われません。PCI 固有の動的データがある場合は同様に `getMarkupData()` に追加する必要があります。

#### updateMarkup() は creator の DOM を更新しない

`updateMarkup()` は `interaction.markup`（モデルの文字列プロパティ）を更新しますが、現在表示中の DOM は更新しません。creator 上でのライブプレビュー（CSS writing-mode の即時反映）が必要な場合は、DOM の直接操作を別途行ってください。

```javascript
// Question.js のコールバック内（必要な場合のみ）
// markup 更新とは別に、creator 上の DOM にもクラスを反映する
var $markupRoot = widget.$container.find('.mathEntryInteraction');
$markupRoot.removeClass('writing-mode-vertical-rl writing-mode-horizontal-tb');
if (interaction.hasClass(WRITING_MODE_VERTICAL_RL_CLASS)) {
    $markupRoot.addClass(WRITING_MODE_VERTICAL_RL_CLASS);
} else if (interaction.hasClass(WRITING_MODE_HORIZONTAL_TB_CLASS)) {
    $markupRoot.addClass(WRITING_MODE_HORIZONTAL_TB_CLASS);
}
```

この DOM 操作は表示目的のみです。永続化には影響しません（保存時に使われるのは `interaction.markup` の文字列です）。

#### QTI XML に同じ情報が2箇所に記録される

`<customInteraction class="...">` と `<pci:markup>` 内の両方にクラスが記録されます。これは `addClass()` が自動的に `<customInteraction>` の `class` 属性を設定するためです。不整合を防ぐため、クラスの追加・削除と `updateMarkup()` は常にセットで呼んでください。

---

## 方式B: クラス属性 + Properties 併用

### 概要

永続化のトリガーとして `addClass()` / `removeClass()` を使い、runtime への情報伝達として `prop()` を併用する方式です。markup.tpl の変更は不要です。ただし runtime の DOM にはクラスが存在しないため、PCI runtime 内で CSS を自前で適用する必要があります。

### 特徴

| 項目 | 内容 |
|------|------|
| 保存トリガー | `addClass()` / `removeClass()` が `attributeChange.qti-widget` を発火 |
| runtime での取得 | `properties.writingMode` パラメータから |
| CSS 継承 | **機能しない**（PCI runtime で自前対応が必要） |

### 変更対象ファイル

| ファイル | 変更内容 |
|---------|---------|
| `creator/widget/Widget.js` | `item-writing-mode-changed` ハンドラ |
| `creator/widget/states/Question.js` | フォーム処理とコールバック |
| `runtime/*.js` | `properties.writingMode` の読み取りと CSS 適用 |

markup.tpl と imsPciCreator.js の変更は**不要**です。

### QTI XML 保存形式

```xml
<customInteraction class="writing-mode-vertical-rl" responseIdentifier="RESPONSE">
    <pci:portableCustomInteraction customInteractionTypeIdentifier="myPci">
        <pci:properties>
            <pci:entry key="writingMode">vertical</pci:entry>
            <!-- ↑ runtime 用 -->
            <pci:entry key="otherProp">value</pci:entry>
        </pci:properties>
        <pci:markup>
            <div class="mathEntryInteraction">
                <!-- ↑ markup にはクラスなし -->
                ...
            </div>
        </pci:markup>
    </pci:portableCustomInteraction>
</customInteraction>
```

`class` 属性（`<customInteraction>` 上）と `<pci:entry key="writingMode">` の2箇所に同じ情報が記録されます。`class` 属性は保存トリガー用、`<pci:entry>` は runtime 伝達用です。

### 実装手順

#### 1. Widget.js の変更

```javascript
initCreator: function() {
    this.registerStates(states);
    Widget.initCreator.call(this);

    var self = this;
    var $itemBody = this.$container.closest('.qti-itemBody');

    // アイテムの writing-mode 変更を監視
    $itemBody.on('item-writing-mode-changed', function() {
        // クラスを削除（保存トリガー）
        self.element.removeClass('writing-mode-vertical-rl');
        self.element.removeClass('writing-mode-horizontal-tb');

        // プロパティを削除（runtime 用）
        // 注意: prop('writingMode', undefined) は削除にならない（getter として動作する）
        // 注意: removeProp() にはバグがある（使用不可）
        delete self.element.properties['writingMode'];
    });
}
```

**プロパティ削除の注意:** 必ず `delete self.element.properties['writingMode']` を使用してください。`prop('writingMode', undefined)` と `removeProp('writingMode')` はどちらも正しく動作しません（本ドキュメント冒頭の「TAO API の動作確認済み事項」参照）。

#### 2. Question.js の変更

```javascript
define([
    'taoQtiItem/qtiCreator/widgets/states/lib/formElement',
    'taoQtiItem/qtiCreator/widgets/static/helpers/verticalWritingEditing'
], function(formElement, verticalWritingEditing) {
    'use strict';

    var WRITING_MODE_VERTICAL_RL_CLASS = 'writing-mode-vertical-rl';
    var WRITING_MODE_HORIZONTAL_TB_CLASS = 'writing-mode-horizontal-tb';

    var QuestionState = stateFactory.extend(Question, function() {
        var widget = this.widget;
        var interaction = widget.element;
        var $form = widget.$form;

        // --- フォーム初期化 ---
        verticalWritingEditing.checkItemWritingMode(widget)
            .then(function(result) {
                var isItemVertical = result.isItemVertical;
                $form.data('isItemVertical', isItemVertical);

                // 現在の設定を判定
                // クラス属性とプロパティの両方から判定可能だが、
                // クラス属性を正とする（標準インタラクションと同じ判定方法）
                var isVertical;
                if (interaction.hasClass(WRITING_MODE_VERTICAL_RL_CLASS)) {
                    isVertical = true;
                } else if (interaction.hasClass(WRITING_MODE_HORIZONTAL_TB_CLASS)) {
                    isVertical = false;
                } else {
                    isVertical = isItemVertical;
                }

                $form.find('input[name="writingMode"][value="vertical"]')
                    .prop('checked', isVertical);
                $form.find('input[name="writingMode"][value="horizontal"]')
                    .prop('checked', !isVertical);
            });

        // --- コールバック ---
        var callbacks = {
            writingMode: function(interaction, mode) {
                var isItemVertical = $form.data('isItemVertical');

                // 1. 既存のクラスを削除
                interaction.removeClass(WRITING_MODE_VERTICAL_RL_CLASS);
                interaction.removeClass(WRITING_MODE_HORIZONTAL_TB_CLASS);

                // 2. アイテムと異なる場合のみ設定
                if (mode === 'vertical' && !isItemVertical) {
                    interaction.addClass(WRITING_MODE_VERTICAL_RL_CLASS);
                    interaction.prop('writingMode', 'vertical');
                } else if (mode === 'horizontal' && isItemVertical) {
                    interaction.addClass(WRITING_MODE_HORIZONTAL_TB_CLASS);
                    interaction.prop('writingMode', 'horizontal');
                } else {
                    // アイテムと同じ → 継承（両方削除）
                    delete interaction.properties['writingMode'];
                }
            }
        };

        formElement.setChangeCallbacks($form, interaction, callbacks);
    });

    return QuestionState;
});
```

**重要:** `addClass()` と `prop()` は常にセットで呼んでください。片方だけ更新すると不整合が発生します。

#### 3. PCI runtime の変更

```javascript
// PCI runtime の initialize 内
initialize: function(id, dom, properties) {
    // properties から writing-mode を取得
    var writingMode = properties.writingMode;
    // → 'vertical', 'horizontal', または undefined（継承）

    if (writingMode === 'vertical') {
        // 縦書き: CSS を自前で適用
        dom.style.writingMode = 'vertical-rl';
    } else if (writingMode === 'horizontal') {
        // 横書き: CSS を自前で適用
        dom.style.writingMode = 'horizontal-tb';
    } else {
        // undefined → アイテムの設定を継承
        // CSS の writing-mode は親要素から自然に継承されるため、
        // 明示的な設定は不要
    }
}
```

**注意:** `dom.style.writingMode` を設定した場合、その値は PCI 内部の子要素にのみ CSS 継承します。PCI 内部で独自のスタイルを適用している要素（`writing-mode` を明示的に設定している要素）には継承されません。必要に応じて PCI 内部の CSS も確認してください。

### 処理フロー図

#### ユーザーが Question.js で writing-mode を変更したとき

```
ユーザーがラジオボタンを変更
    ↓
callbacks.writingMode(interaction, mode) が呼ばれる
    ↓
interaction.removeClass(...)
    ↓ （attributeChange.qti-widget 発火 → 保存トリガー）
    ↓
interaction.addClass(WRITING_MODE_VERTICAL_RL_CLASS)
    ↓ （attributeChange.qti-widget 発火）
    ↓
interaction.prop('writingMode', 'vertical')
    ↓ （イベントなし。ただし保存は既にトリガーされている）
    ↓
保存時に QTI XML に出力:
├─ <customInteraction class="writing-mode-vertical-rl">  （attr 由来）
└─ <pci:entry key="writingMode">vertical</pci:entry>    （prop 由来）
```

#### アイテムの writing-mode が変更されたとき

```
Active.js: $itemBody.trigger('item-writing-mode-changed')
    ↓
Widget.js: イベントを受信
    ↓
self.element.removeClass(...)
    ↓ （attributeChange.qti-widget 発火 → 保存トリガー）
    ↓
delete self.element.properties['writingMode']
    ↓ （イベントなし。ただし保存は既にトリガーされている）
    ↓
保存時:
├─ <customInteraction>                        （class 属性なし）
└─ <pci:properties> に writingMode なし        （delete 済み）
```

#### runtime（テスト配信時）

```
QTI XML からアイテムを読み込み
    ↓
customInteraction.tpl でレンダリング
    ↓
<div class="qti-interaction qti-customInteraction" data-serial="...">
    ← class="writing-mode-vertical-rl" は出力されない
    <div class="mathEntryInteraction">
        ← markup にもクラスなし
    </div>
</div>
    ↓
PCI runtime の initialize(id, dom, properties) が呼ばれる
    ↓
properties.writingMode → 'vertical'
    ↓
dom.style.writingMode = 'vertical-rl' を設定（自前）
    ↓
PCI 内部の子要素に CSS 継承
```

### 方式B の注意事項

#### クラス属性とプロパティの不整合リスク

同じ情報を2箇所（`attr('class')` と `properties['writingMode']`）に保存するため、片方のみ更新してしまう実装ミスが不整合を引き起こします。

**対策:** クラス操作とプロパティ操作は必ずセットで行うコードにしてください。ヘルパー関数にまとめることを推奨します。

```javascript
// ヘルパー関数の例
function setWritingMode(interaction, mode, isItemVertical) {
    interaction.removeClass('writing-mode-vertical-rl');
    interaction.removeClass('writing-mode-horizontal-tb');

    if (mode === 'vertical' && !isItemVertical) {
        interaction.addClass('writing-mode-vertical-rl');
        interaction.prop('writingMode', 'vertical');
    } else if (mode === 'horizontal' && isItemVertical) {
        interaction.addClass('writing-mode-horizontal-tb');
        interaction.prop('writingMode', 'horizontal');
    } else {
        delete interaction.properties['writingMode'];
    }
}

function clearWritingMode(interaction) {
    interaction.removeClass('writing-mode-vertical-rl');
    interaction.removeClass('writing-mode-horizontal-tb');
    delete interaction.properties['writingMode'];
}
```

#### runtime で CSS writing-mode が自動継承しない

DOM にクラスが存在しないため、CSS の `writing-mode` プロパティは自然には適用されません。PCI runtime の `initialize()` 内で `dom.style.writingMode` を明示的に設定する必要があります。

PCI 内部の DOM 構造が複雑な場合（iframe、Shadow DOM、独自のスタイル設定がある要素など）、`dom.style.writingMode` の CSS 継承が期待通りに機能するか個別に検証してください。

#### prop() の保存はクラス属性の保存に便乗している

`prop()` は単独では保存をトリガーしません。`addClass()` / `removeClass()` が `attributeChange.qti-widget` を発火することで保存がトリガーされ、その保存処理の中で `this.properties` も一緒に XML に出力されます。

このため、**`addClass()` / `removeClass()` なしで `prop()` だけを呼ぶと保存されない可能性があります。** 必ずクラス操作を先に行ってください。

---

## 方式A と方式B の比較

| 比較軸 | 方式A (markup.tpl) | 方式B (class + prop) |
|--------|---|---|
| **変更ファイル数** | 5ファイル | 3ファイル |
| **markup.tpl の変更** | 必要 | 不要 |
| **imsPciCreator.js の変更** | 必要 | 不要 |
| **runtime での CSS 継承** | 自然に機能 | 自前で CSS 適用が必要 |
| **runtime の実装量** | DOM クラス判定のみ | プロパティ読み取り + CSS 適用 |
| **データの冗長性** | class 属性 + markup 内クラス | class 属性 + properties |
| **不整合リスク** | addClass と updateMarkup のセット呼び忘れ | addClass と prop のセット呼び忘れ |
| **標準インタラクションとの類似度** | 高い（DOM にクラスがある、CSS 継承） | 中程度（creator 側は類似、runtime は異なる） |

### 方式A を選ぶべき場合

- PCI 内部で CSS `writing-mode` の自然な継承が必要な場合
- PCI 内部の DOM 構造が複雑で、`dom.style.writingMode` の手動設定では対応が難しい場合
- 標準インタラクションとの動作の一貫性を重視する場合

### 方式B を選ぶべき場合

- markup.tpl と imsPciCreator.js を変更したくない場合
- PCI 内部の writing-mode 適用が `dom.style.writingMode` の1行で十分な場合
- 変更ファイル数を最小にしたい場合

---

## 共通の注意事項

### プロパティ削除に関する既知のバグ

以下のメソッドは正しく動作しません。

| メソッド | 期待する動作 | 実際の動作 |
|---------|------------|-----------|
| `prop('writingMode', undefined)` | プロパティを削除 | **getter として動作する（何も変更されない）** |
| `removeProp('writingMode')` | プロパティを削除 | **`this.attributes` を削除する（`this.properties` ではない）** |

プロパティを削除する場合は必ず `delete interaction.properties['writingMode']` を使用してください。

### formElement の関数名

既存のドキュメントやコードで `formElement.initDataBinding` と記載されている場合がありますが、**実際の関数名は `formElement.setChangeCallbacks`** です。`initDataBinding` は存在しません。

### 保存ルール（アイテムと同じ設定は保存しない）

| アイテムの設定 | PCI の選択 | 保存する？ | 理由 |
|---------------|----------|-----------|------|
| 横書き | 横書き | しない | 継承で同じ結果 |
| 横書き | 縦書き | する | アイテムと異なる |
| 縦書き | 横書き | する | アイテムと異なる |
| 縦書き | 縦書き | しない | 継承で同じ結果 |

---

## 検証チェックリスト

実装後に以下を確認してください。

### creator 側

- [ ] PCI のフォームで「縦書き」を選択 → 保存 → 再度開く → ラジオボタンが「縦書き」になっている
- [ ] PCI のフォームで「横書き」を選択 → 保存 → 再度開く → ラジオボタンが「横書き」になっている
- [ ] アイテムが横書きの状態で PCI を「横書き」に設定 → 保存 → QTI XML に writing-mode のクラス/プロパティが**ない**ことを確認
- [ ] アイテムの writing-mode を変更 → PCI の設定がリセットされる（継承状態になる）
- [ ] アイテムの writing-mode を変更 → 保存 → 再度開く → PCI が継承状態になっている

### runtime 側

- [ ] 明示的に縦書きが設定された PCI → runtime で縦書きが適用されている
- [ ] 明示的に横書きが設定された PCI → runtime で横書きが適用されている
- [ ] 継承状態の PCI → runtime でアイテムの設定に従っている
- [ ] アイテムが縦書き、PCI が横書き（明示的） → runtime で PCI のみ横書きになっている

### 方式A 追加チェック

- [ ] `updateMarkup()` 後に prompt が失われていないことを確認
- [ ] QTI XML の `<pci:markup>` 内に writing-mode クラスが含まれていることを確認

### 方式B 追加チェック

- [ ] QTI XML の `<pci:properties>` 内に `<pci:entry key="writingMode">` が含まれていることを確認
- [ ] PCI runtime で `properties.writingMode` が正しい値であることを確認
