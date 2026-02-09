# PCI 文字列方向設定 実装ガイド

## 目的

PCIに標準インタラクションと同等の文字列方向（writing-mode）設定機能を追加する。

---

## 標準インタラクションの仕組み（参考）

### 設定値の保存

標準インタラクションは**クラス属性**で設定を保存します。

| 状態 | クラス属性 |
|------|-----------|
| 縦書き（明示的） | `class="writing-mode-vertical-rl"` |
| 横書き（明示的） | `class="writing-mode-horizontal-tb"` |
| 継承 | クラスなし |

### 継承の仕組み

- **クラスなし = アイテムの設定を継承**
- アイテム変更時: インタラクションのクラスを削除 → 継承状態になる
- CSS継承とフォームロジックの両方で継承が機能

---

## PCIでの実装方針

### 設定値の保存

PCIは**プロパティ**で設定を保存します。

| 状態 | プロパティ |
|------|-----------|
| 縦書き（明示的） | `writingMode: 'vertical'` |
| 横書き（明示的） | `writingMode: 'horizontal'` |
| 継承 | プロパティなし（または`undefined`） |

### QTI XMLでの保存形式

```xml
<customInteraction responseIdentifier="RESPONSE">
    <pci:portableCustomInteraction customInteractionTypeIdentifier="myPci">
        <pci:properties>
            <!-- 明示的設定がある場合のみ保存 -->
            <pci:entry key="writingMode">vertical</pci:entry>
            <!-- その他のプロパティ -->
            <pci:entry key="otherProp">value</pci:entry>
        </pci:properties>
        <pci:markup>...</pci:markup>
    </pci:portableCustomInteraction>
</customInteraction>
```

---

## 実装詳細

### 1. プロパティ操作メソッド

```javascript
// プロパティの設定
this.element.prop('writingMode', 'vertical');

// プロパティの取得
const mode = this.element.prop('writingMode');  // 'vertical' or undefined

// 全プロパティの取得
const allProps = this.element.getProperties();
```

### 2. プロパティの削除（注意）

**`removeProp()`メソッドにはバグがあります。** 代わりに以下の方法を使用してください。

```javascript
// 方法1: undefined を設定
this.element.prop('writingMode', undefined);

// 方法2: 直接削除
delete this.element.properties['writingMode'];
```

---

## 実装コード例

### Widget.js（アイテム変更時の処理）

```javascript
define([
    'taoQtiItem/qtiCreator/widgets/interactions/customInteraction/Widget',
    'myPci/creator/widget/states/states'
], function(Widget, states) {
    'use strict';

    var MyPciWidget = Widget.clone();

    MyPciWidget.initCreator = function() {
        this.registerStates(states);
        Widget.initCreator.call(this);

        // アイテムの文字列方向変更を監視
        var self = this;
        var $itemBody = this.$container.closest('.qti-itemBody');

        $itemBody.on('item-writing-mode-changed', function() {
            // プロパティを削除して継承状態にする
            self.element.prop('writingMode', undefined);
            // または: delete self.element.properties['writingMode'];
        });
    };

    return MyPciWidget;
});
```

### Question.js（フォーム処理）

```javascript
define([
    'taoQtiItem/qtiCreator/widgets/states/lib/formElement',
    'taoQtiItem/qtiCreator/widgets/static/helpers/verticalWritingEditing'
], function(formElement, verticalWritingEditing) {
    'use strict';

    var QuestionState = stateFactory.extend(Question, function() {
        var widget = this.widget;
        var interaction = widget.element;
        var $form = widget.$form;

        // アイテムの設定を取得
        verticalWritingEditing.checkItemWritingMode(widget)
            .then(function(result) {
                var isItemVertical = result.isItemVertical;
                $form.data('isItemVertical', isItemVertical);

                // インタラクションの設定を判定
                var writingMode = interaction.prop('writingMode');
                var isVertical;

                if (writingMode === 'vertical') {
                    isVertical = true;
                } else if (writingMode === 'horizontal') {
                    isVertical = false;
                } else {
                    // プロパティなし → アイテムの設定を継承
                    isVertical = isItemVertical;
                }

                // ラジオボタンを設定
                $form.find('input[name="writingMode"][value="vertical"]')
                    .prop('checked', isVertical);
                $form.find('input[name="writingMode"][value="horizontal"]')
                    .prop('checked', !isVertical);
            });

        // コールバック設定
        var callbacks = {
            writingMode: function(i, mode) {
                var isItemVertical = $form.data('isItemVertical');

                if ((mode === 'vertical' && isItemVertical) ||
                    (mode === 'horizontal' && !isItemVertical)) {
                    // アイテムと同じ設定 → プロパティを削除（継承させる）
                    interaction.prop('writingMode', undefined);
                } else {
                    // アイテムと異なる設定 → プロパティを保存
                    interaction.prop('writingMode', mode);
                }
            }
        };

        formElement.initDataBinding($form, interaction, callbacks);
    });

    return QuestionState;
});
```

### フォームテンプレート（tpl）

```html
<div class="panel writingMode-panel">
    <h3>{{__ "Direction of writing"}}</h3>
    <div>
        <label class="smaller-prompt">
            <input type="radio" name="writingMode" value="horizontal" />
            <span class="icon-radio"></span>
            {{__ "Horizontal text"}}
        </label>
        <br>
        <label class="smaller-prompt">
            <input type="radio" name="writingMode" value="vertical" />
            <span class="icon-radio"></span>
            {{__ "Vertical text"}}
        </label>
    </div>
</div>
```

---

## 処理フロー

### 1. アイテムの文字列方向が変更されたとき

```
ユーザーがアイテムの設定を変更
    ↓
Active.js: $itemBody.trigger('item-writing-mode-changed')
    ↓
Widget.js: イベントを受信
    ↓
interaction.prop('writingMode', undefined) でプロパティを削除
    ↓
PCIは「継承」状態になる
```

### 2. PCIのフォームを開いたとき

```
ユーザーがPCIを選択
    ↓
Question.js: プロパティをチェック

if (prop('writingMode') === 'vertical') {
    → ラジオボタン「縦書き」を選択
}
else if (prop('writingMode') === 'horizontal') {
    → ラジオボタン「横書き」を選択
}
else {
    → アイテムの設定に基づいてラジオボタンを選択
}
```

### 3. ユーザーがPCIの設定を変更したとき

```
ユーザーがラジオボタンを変更
    ↓
callbacks.writingMode が実行
    ↓
アイテムと同じ設定？
    ├─ YES → プロパティを削除（継承させる）
    └─ NO  → プロパティを保存
```

---

## 保存ルール

| アイテムの設定 | PCIの選択 | 保存される？ | 理由 |
|---------------|----------|-------------|------|
| 横書き | 横書き | ❌ しない | 継承で同じ結果 |
| 横書き | 縦書き | ✅ する | アイテムと異なる |
| 縦書き | 横書き | ✅ する | アイテムと異なる |
| 縦書き | 縦書き | ❌ しない | 継承で同じ結果 |

---

## 標準インタラクションとPCIの比較

| 項目 | 標準インタラクション | PCI |
|------|-------------------|-----|
| 保存場所 | クラス属性 | プロパティ |
| 設定方法 | `addClass()` | `prop()` |
| 削除方法 | `removeClass()` | `prop(name, undefined)` |
| 継承状態 | クラスなし | プロパティなし |
| XML保存 | 要素のclass属性 | `<pci:properties>` |

---

## 実装チェックリスト

### Widget.js
- [ ] `item-writing-mode-changed` イベントリスナーを登録
- [ ] イベント受信時に `prop('writingMode', undefined)` でプロパティを削除

### Question.js
- [ ] `verticalWritingEditing.checkItemWritingMode()` でアイテムの設定を取得
- [ ] `prop('writingMode')` で現在の設定を確認
- [ ] プロパティがなければアイテムの設定を継承
- [ ] ラジオボタンの状態を設定
- [ ] `callbacks.writingMode` で変更時の処理を実装
- [ ] アイテムと同じ設定ならプロパティを削除

### テンプレート
- [ ] writing-mode用のラジオボタンを追加

---

## 注意事項

### removeProp() のバグについて

`CustomElement.js` の `removeProp()` メソッドは `this.attributes` を削除するバグがあります。
プロパティを削除する場合は、以下のいずれかを使用してください。

```javascript
// 推奨: undefined を設定
this.element.prop('writingMode', undefined);

// 代替: 直接削除
delete this.element.properties['writingMode'];
```

### CSS継承について

PCI内部のDOMに対してCSSの `writing-mode` プロパティが正しく継承されるか確認が必要です。
PCI内部で独自のスタイルを適用している場合、継承が機能しない可能性があります。
