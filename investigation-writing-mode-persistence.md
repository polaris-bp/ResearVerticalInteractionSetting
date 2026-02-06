# Writing-Mode 親子継承・上書き構造の調査レポート

## 問題の背景

以下の構造を実現したいという要件があります：
- **親（アイテム）の文字列方向設定が子（インタラクション）にも反映される**
- **子の設定をしたときは子の設定のみが更新される**

この「継承と上書き」の仕組みが標準インタラクションでどのように実現されているかを調査しました。

---

## 調査結果

### 核心的な発見

**標準インタラクション（choiceInteraction等）は、個別にwriting-mode設定を持っています。**

親子間の継承と上書きの構造が実装されています：
1. インタラクションに明示的な設定がない場合 → **アイテムの設定を継承**
2. インタラクションに明示的な設定がある場合 → **インタラクションの設定を優先**
3. **アイテムの設定が変更された場合** → **インタラクションの明示的設定がリセットされる**

---

## 実装の詳細

### 1. インタラクション個別のwriting-mode設定

**ファイル:** `extension-tao-itemqti/views/js/qtiCreator/widgets/interactions/choiceInteraction/states/Question.js`

#### フォームでのwriting-mode設定UI

```javascript
// choice.tpl（フォームテンプレート）
<div class="panel writingMode-panel" style="display:none;">
    <h3>{{__ "Direction of writing"}}</h3>
    <div>
        <label class="smaller-prompt">
            <input type="radio" name="writingMode" value="horizontal" />
            {{__ "Horizontal text"}}
        </label>
        <br>
        <label class="smaller-prompt">
            <input type="radio" name="writingMode" value="vertical" />
            {{__ "Vertical text"}}
        </label>
    </div>
</div>
```

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

    itemScrollingMethods.initSelect($form, isScrolling, writingModeInitialScrollingHeight);
    itemScrollingMethods.wrapContent(widget, isScrolling, 'interaction');
};
```

### 2. 継承ロジックの実装

**フォーム初期化時のwriting-mode判定:**

```javascript
const toggleVerticalWritingModeByLang = (widget, $form, interaction) =>
    verticalWritingEditing
        .checkItemWritingMode(widget)
        .then(({ isVerticalSupported, isItemVertical }) => {
            // アイテムのwriting-mode設定を保存
            $form.data('isItemVertical', isItemVertical);

            // writing-modeパネルの表示/非表示
            $form.find('.writingMode-panel').toggle(isVerticalSupported);

            // インタラクションのwriting-modeを判定
            let isVertical = null;

            if (interaction.hasClass(writingModeVerticalRlClass)) {
                // ケース1: インタラクションに縦書き設定がある
                isVertical = true;
            } else if (interaction.hasClass(writingModeHorizontalTbClass)) {
                // ケース2: インタラクションに横書き設定がある
                isVertical = false;
            } else {
                // ケース3: インタラクションに設定がない → アイテムの設定を継承
                isVertical = isItemVertical;
            }

            return new Promise(resolve => setTimeout(() => resolve(isVertical), 0));
        })
        .then(isVertical => {
            // フォームのラジオボタンを更新
            $form.find('input[name="writingMode"][value="vertical"]').prop('checked', isVertical);
            $form.find('input[name="writingMode"][value="horizontal"]').prop('checked', !isVertical);
        });
```

---

## 継承・上書き構造の図解

```
┌─────────────────────────────────────────────────────────────────────┐
│                 親子継承・上書き構造                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ アイテム（親）                                                │   │
│  │   class="writing-mode-vertical-rl"                           │   │
│  │   └→ アイテム全体の文字列方向を設定                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ インタラクション（子）の判定ロジック                           │   │
│  │                                                               │   │
│  │   if (interaction.hasClass('writing-mode-vertical-rl')) {     │   │
│  │       // インタラクション個別設定: 縦書き                      │   │
│  │   } else if (interaction.hasClass('writing-mode-horizontal-tb')) {  │
│  │       // インタラクション個別設定: 横書き                      │   │
│  │   } else {                                                    │   │
│  │       // 設定なし → アイテムの設定を継承                       │   │
│  │   }                                                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### ケース別の動作

| アイテム設定 | インタラクション設定 | 最終的なwriting-mode |
|-------------|-------------------|---------------------|
| 縦書き | なし（継承） | **縦書き** |
| 縦書き | 横書き（上書き） | **横書き** |
| 横書き | なし（継承） | **横書き** |
| 横書き | 縦書き（上書き） | **縦書き** |

---

## アイテム変更時のインタラクション追従処理

### 核心的な発見

**アイテムのwriting-modeが変更されると、インタラクションの明示的設定（クラス）がリセットされます。**

**ファイル:** `extension-tao-itemqti/views/js/qtiCreator/widgets/interactions/choiceInteraction/Widget.js`

```javascript
var ChoiceInteractionWidget = Widget.clone();

ChoiceInteractionWidget.initCreator = function () {
    this.registerStates(states);
    Widget.initCreator.call(this);

    // ...

    const $itemBody = this.$container.closest('.qti-itemBody');
    $itemBody.on('item-writing-mode-changed', () => {
        // アイテムのwriting-modeが変更されたら、インタラクションの設定をリセット
        this.element.removeClass(verticalWriting.WRITING_MODE_VERTICAL_RL_CLASS);
        this.element.removeClass(verticalWriting.WRITING_MODE_HORIZONTAL_TB_CLASS);
        itemScrollingMethods.wrapContent(this, false, 'interaction');
    });
};
```

### 動作フロー

```
┌─────────────────────────────────────────────────────────────────────┐
│           アイテム変更時のインタラクション追従処理                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. ユーザーがアイテムのwriting-modeを変更                           │
│     └→ Active.js: $itemBody.trigger('item-writing-mode-changed')    │
│                                                                     │
│  2. インタラクションのWidgetがイベントをリッスン                      │
│     └→ Widget.js: $itemBody.on('item-writing-mode-changed', ...)    │
│                                                                     │
│  3. インタラクションの明示的設定をリセット                            │
│     └→ element.removeClass('writing-mode-vertical-rl')              │
│     └→ element.removeClass('writing-mode-horizontal-tb')            │
│                                                                     │
│  4. 結果: インタラクションはアイテムの新しい設定を継承                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### シナリオ別の動作

| # | 初期状態 | 操作 | 結果 |
|---|---------|------|------|
| 1 | アイテム:横書き, インタラクション:縦書き(明示) | アイテムを縦書きに変更 | インタラクション設定リセット → 縦書き(継承) |
| 2 | アイテム:縦書き, インタラクション:横書き(明示) | アイテムを横書きに変更 | インタラクション設定リセット → 横書き(継承) |
| 3 | アイテム:横書き, インタラクション:なし(継承) | アイテムを縦書きに変更 | そのまま縦書き(継承) |

### 重要な前提条件

**このイベントリスナーは、Widgetが初期化されている場合のみ動作します。**

つまり：
- インタラクションが**選択されている**（Widgetがアクティブ）状態でアイテム設定を変更 → **追従する**
- インタラクションが**選択されていない**状態でアイテム設定を変更 → **追従しない**（クラスが残る）

ただし、後者の場合でも：
- 保存されたQTI XMLにはインタラクションのクラスが残る
- 次回アイテムを開いてインタラクションを選択すると、そのクラスに基づいて表示される
- ユーザーがインタラクションのwriting-modeを再設定すれば、アイテムと同じ設定を選択した時点でクラスが削除される

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
1. アイテムのwriting-modeが変更されても、明示的に設定されていないインタラクションは自動的に追従
2. QTI XMLのサイズを最小化（不要なクラスを保存しない）

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

## PCIでの実装方針

### 標準インタラクションと同様の方法で実現可能

PCIでも以下の構造を実装できます：

#### 1. PCI個別のwriting-mode設定を持つ

```javascript
// PCI Creator側（Question.js相当）
callbacks.writingMode = function (i, mode) {
    const isItemVertical = $form.data('isItemVertical');

    // 既存のクラスを削除
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

#### 2. 継承ロジックの実装

```javascript
// PCI初期化時
const initWritingMode = (pciElement, isItemVertical) => {
    let isVertical;

    if (pciElement.hasClass('writing-mode-vertical-rl')) {
        // PCI個別設定: 縦書き
        isVertical = true;
    } else if (pciElement.hasClass('writing-mode-horizontal-tb')) {
        // PCI個別設定: 横書き
        isVertical = false;
    } else {
        // 設定なし → アイテムの設定を継承
        isVertical = isItemVertical;
    }

    return isVertical;
};
```

#### 3. フォームテンプレートへの追加

```html
<div class="panel writingMode-panel" style="display:none;">
    <h3>{{__ "Direction of writing"}}</h3>
    <div>
        <label>
            <input type="radio" name="writingMode" value="horizontal" />
            {{__ "Horizontal text"}}
        </label>
        <br>
        <label>
            <input type="radio" name="writingMode" value="vertical" />
            {{__ "Vertical text"}}
        </label>
    </div>
</div>
```

---

## 「PCIプログラムが走っていない状態」への対処

### シナリオ

1. PCIをアイテムに配置
2. アイテム一覧画面に戻る
3. 再度アイテムを選択
4. **PCIを一度も選択せず**
5. アイテムの文字列方向を変更

### 標準インタラクションでの動作

標準インタラクションでも同じシナリオが発生します。

**重要な発見:**
- インタラクションの**Widgetが初期化されていない場合**、`item-writing-mode-changed`イベントのリスナーは存在しない
- したがって、**インタラクションに明示的な設定がある場合、その設定は維持される**
- インタラクションに明示的な設定がない場合は、CSS継承でアイテムの設定に追従する

### ケース別の動作

| ケース | インタラクション設定 | Widget状態 | アイテム変更時の動作 |
|--------|-------------------|------------|-------------------|
| A | なし（継承） | 非アクティブ | CSS継承で自動追従 |
| B | なし（継承） | アクティブ | CSS継承で自動追従 |
| C | あり（明示） | 非アクティブ | **変更されない**（クラス維持） |
| D | あり（明示） | アクティブ | **リセットされる**（クラス削除） |

### ケースCの詳細

インタラクションに明示的な設定があり、Widgetが非アクティブな状態でアイテム設定を変更した場合：

```
初期状態:
  アイテム: 横書き
  インタラクション: 縦書き（class="writing-mode-vertical-rl"）

操作:
  インタラクションを選択せずにアイテムを縦書きに変更

結果:
  アイテム: 縦書き（class="writing-mode-vertical-rl"）
  インタラクション: 縦書き（class="writing-mode-vertical-rl"）← クラスが残っている

表示上は問題なし（両方縦書き）
ただし、インタラクションには「不要な」クラスが残っている状態
```

### PCIでの対処

PCIも同様の動作になります：

1. **PCI個別設定がない場合** → CSS継承でアイテム設定に追従
2. **PCI個別設定がある + Widgetアクティブ** → リセットされてアイテム設定を継承
3. **PCI個別設定がある + Widget非アクティブ** → 設定維持（クラスが残る）

```
アイテム変更時:
  ├─ PCI個別設定なし → 自動的にアイテム設定に追従（CSS継承）
  ├─ PCI個別設定あり + Widgetアクティブ → リセット → アイテム設定を継承
  └─ PCI個別設定あり + Widget非アクティブ → 変更なし（PCI設定を維持）
```

### 設計上の考慮点

この動作は設計上の選択です：
- **利点**: Widget非アクティブ時のパフォーマンス最適化（不要な処理を行わない）
- **欠点**: 明示的設定がある場合、アイテム変更に追従しないケースがある

標準インタラクションと同様の動作をPCIで実現すれば、一貫した挙動になります。

---

## 実装チェックリスト

### PCI側の実装

- [ ] インタラクション要素にwriting-modeクラスを設定できるようにする
- [ ] フォームにwriting-mode設定UIを追加
- [ ] アイテムのwriting-modeを判定するヘルパーを使用
- [ ] アイテムと異なる設定の場合のみクラスを追加（最適化）
- [ ] 継承ロジック（クラスがない場合はアイテムの設定を使用）

### TAO側の実装（必要に応じて）

- [ ] PCI CreatorでverticalWritingEditingヘルパーを使用可能にする
- [ ] PCI要素のclass属性をQTI XMLに保存できるようにする

---

## 参考リンク

- [choiceInteraction Question.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/interactions/choiceInteraction/states/Question.js)
- [choice.tpl (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/tpl/forms/interactions/choice.tpl)
- [verticalWritingEditing.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/static/helpers/verticalWritingEditing.js)
- [verticalWriting.js (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiCommonRenderer/helpers/verticalWriting.js)
