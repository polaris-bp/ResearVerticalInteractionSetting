# 標準インタラクション文字列方向設定の追従メカニズム調査レポート

## 概要

標準インタラクションの文字列方向設定（writing-mode）がアイテムの設定に追従する仕組みについて調査しました。また、PCI（Portable Custom Interaction）に対して同様の動作を実現するための実装方針を検討しました。

---

## 調査結果

### 1. 仕組みの全体像

```
┌────────────────────────────────────────────────────────────────────┐
│                    文字列方向設定の追従メカニズム                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 1. アイテム設定フェーズ（QTI Creator）                         │  │
│  │    - 言語設定に基づきwriting-modeのサポートを判定              │  │
│  │    - itemに`writing-mode-vertical-rl`クラスを追加/削除        │  │
│  │    - イベント`item-writing-mode-changed`を発火                │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 2. DOM構造への反映                                            │  │
│  │    - .qti-itemBodyに`writing-mode-vertical-rl`クラスが付与    │  │
│  │    - CSSで`writing-mode: vertical-rl`が適用される             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 3. CSS継承による自動伝播                                       │  │
│  │    - CSSの`writing-mode`プロパティは子要素に継承される         │  │
│  │    - インタラクションはitemBodyの子要素として配置される         │  │
│  │    → 自動的に縦書きモードが適用される                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 4. レンダラーでの追加処理                                      │  │
│  │    - getIsWritingModeVerticalRl()で親のwriting-modeを判定     │  │
│  │    - インストラクションテキストの縦書き対応処理等               │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

### 2. 詳細な処理フロー

#### 2.1 アイテムでのwriting-mode設定

**ファイル:** `extension-tao-itemqti/views/js/qtiCreator/widgets/item/states/Active.js`

```javascript
const writingModeVerticalRlClass = 'writing-mode-vertical-rl';

// 言語設定変更時のコールバック
'xml:lang': function langChange(i, lang) {
    // ... RTL処理 ...

    toggleVerticalWritingModeByLang(lang).then(() => {
        // writing-mode変更イベントを発火
        $itemBody.trigger('item-writing-mode-changed');
    });
},

// writing-modeの直接設定
writingModeItem(i, mode) {
    if (mode === 'vertical') {
        item.addClass(writingModeVerticalRlClass);
    } else {
        itemRemoveClasses([writingModeVerticalRlClass]);
    }
    $itemBody.trigger('item-writing-mode-changed');
}
```

#### 2.2 言語に基づくwriting-modeサポート判定

**ファイル:** `extension-tao-itemqti/views/js/qtiCreator/widgets/static/helpers/verticalWritingEditing.js`

```javascript
checkItemWritingMode: function (widget) {
    const rootElement = widget.element.getRootElement();
    const itemLang = rootElement.attr('xml:lang');

    return languages.getVerticalWritingModeByLang(itemLang).then(supportedVerticalMode => {
        return {
            isVerticalSupported: supportedVerticalMode === 'vertical-rl',
            isItemVertical: !!rootElement.hasClass(verticalWriting.WRITING_MODE_VERTICAL_RL_CLASS)
        };
    });
}
```

**ポイント:** 日本語などの縦書きをサポートする言語が設定されている場合のみ、writing-mode設定パネルが表示されます。

#### 2.3 DOM構造への反映

**テンプレート:** `tao-item-runner-qti-fe/src/qtiCommonRenderer/tpl/item.tpl`

```html
<div class="qti-item tao-scope runtime" ...>
    <div class="qti-itemBody {{#if attributes.class}} {{attributes.class}}{{/if}}">
        {{{body}}}  <!-- インタラクションはここに配置される -->
    </div>
</div>
```

`attributes.class`にはアイテムに設定されたクラス（`writing-mode-vertical-rl`を含む）が反映されます。

#### 2.4 レンダラーでのwriting-mode判定

**ファイル:** `tao-item-runner-qti-fe/src/qtiCommonRenderer/helpers/verticalWriting.js`

```javascript
export const WRITING_MODE_VERTICAL_RL_CLASS = 'writing-mode-vertical-rl';
export const WRITING_MODE_HORIZONTAL_TB_CLASS = 'writing-mode-horizontal-tb';

// アイテム全体のwriting-mode判定
export const getIsItemWritingModeVerticalRl = () => {
    const itemBody = $('.qti-itemBody');
    return itemBody.hasClass(WRITING_MODE_VERTICAL_RL_CLASS);
};

// 特定要素のwriting-mode判定（親要素を遡って判定）
export const getIsWritingModeVerticalRl = $container => {
    if (!$container.length) {
        return false;
    }
    const $writingModeParent = $container.closest(
        `.${WRITING_MODE_VERTICAL_RL_CLASS}, .${WRITING_MODE_HORIZONTAL_TB_CLASS}`
    );
    return Boolean(
        $writingModeParent.length &&
        $writingModeParent.hasClass(WRITING_MODE_VERTICAL_RL_CLASS)
    );
};
```

#### 2.5 標準インタラクションでの活用例

**ファイル:** `tao-item-runner-qti-fe/src/qtiCommonRenderer/renderers/interactions/ChoiceInteraction.js`

```javascript
import { getIsWritingModeVerticalRl, wrapDigitsInCombineUpright }
    from 'taoQtiItem/qtiCommonRenderer/helpers/verticalWriting';

const render = function render(interaction) {
    const $container = containerHelper.get(interaction);

    // 親要素からwriting-modeを判定
    const isVertical = getIsWritingModeVerticalRl($container);

    // インストラクションの設定（縦書き対応）
    _setInstructions(interaction, isVertical);
    // ...
};

// 縦書き時のテキスト処理
const _setInstructions = function _setInstructions(interaction, isVertical) {
    // 数字を縦書き用にラップ
    msg = wrapDigitsInCombineUpright(__('You need to select at least 1 choice.'), isVertical);
    // ...
};
```

---

### 3. 伝播メカニズムのポイント

#### 3.1 CSS継承による自動伝播

CSSの`writing-mode`プロパティは子要素に継承されるため、itemBodyに設定されたwriting-modeは自動的に子要素（インタラクション）に適用されます。

```css
/* 想定されるスタイル定義 */
.writing-mode-vertical-rl {
    writing-mode: vertical-rl;
}

.writing-mode-horizontal-tb {
    writing-mode: horizontal-tb;
}
```

#### 3.2 標準インタラクションの特別処理

標準インタラクションでは、CSS継承に加えて以下の追加処理が行われます：

1. **インストラクションテキストの縦書き対応**: 数字を`<span class="txt-combine-upright-all">`でラップ
2. **サイズ計算の方向切り替え**: 縦書き時はwidth/height計算の方向を変更
3. **フォーム要素のサポート判定**: 古いSafariでの縦書きフォーム要素サポートを確認

---

## PCIへの実装方針

### 現状の課題

PCIレンダラー（`PortableCustomInteraction.js`）では、writing-modeに関する処理が実装されていません：

```javascript
// PortableCustomInteraction.js - writing-mode関連の処理なし
var render = function render(interaction, options) {
    // PCI初期化処理...
    // writing-modeの情報は渡されていない
};
```

### 実装方針の検討

#### 方針1: CSS継承のみに依存（最小限の実装）

**概要:** PCIはitemBodyの子要素として配置されるため、CSS継承によりwriting-modeは自動的に適用される。

**メリット:**
- TAO側の実装変更不要
- PCI側で`writing-mode`を意識したCSSを書くだけで対応可能

**デメリット:**
- PCI内部のJavaScript処理でwriting-modeを考慮できない
- 動的なレイアウト変更などに対応困難

**実装例（PCI側）:**
```css
/* PCI内部のスタイル */
.my-pci-container {
    /* writing-modeはCSS継承で自動的に適用される */
    /* 縦書き時に適切に表示されるようレイアウトを調整 */
}

.my-pci-container .number {
    text-combine-upright: all;  /* 縦書き時の数字処理 */
}
```

---

#### 方針2: PCI APIへのwriting-mode情報の追加（推奨）

**概要:** PCI初期化時にwriting-mode情報をpropertiesとして渡す。

**実装箇所:** `tao-item-runner-qti-fe/src/qtiCommonRenderer/renderers/interactions/pci/ims.js`

```javascript
createInstance(interaction, context) {
    // 既存のコード
    const contentLanguage = interaction.attributes && interaction.attributes.language;
    const itemLanguage = interaction.rootElement &&
        interaction.rootElement.attributes &&
        interaction.rootElement.attributes['xml:lang'];
    const language = contentLanguage || itemLanguage;
    const userLanguage = sharedContext && sharedContext.locale;

    // 追加: writing-mode情報を取得
    const isWritingModeVertical = interaction.rootElement &&
        interaction.rootElement.hasClass &&
        interaction.rootElement.hasClass('writing-mode-vertical-rl');

    const properties = _.assign(_.clone(interaction.properties), {
        language,
        userLanguage,
        writingMode: isWritingModeVertical ? 'vertical-rl' : 'horizontal-tb'  // 追加
    });

    // ...
}
```

**PCI側での利用:**
```javascript
// PCI実装内
getInstance(dom, config, state) {
    const writingMode = config.properties.writingMode;

    if (writingMode === 'vertical-rl') {
        // 縦書きモード用の初期化処理
        this.initVerticalMode();
    }
    // ...
}
```

**メリット:**
- PCI内部で明示的にwriting-modeを判定可能
- 動的な処理に対応可能

**デメリット:**
- TAO側の実装変更が必要
- IMS PCI仕様への独自拡張となる

---

#### 方針3: DOM参照によるwriting-mode判定（PCI側実装）

**概要:** PCI内部で親要素のwriting-modeクラスを判定する。

**実装例（PCI側）:**
```javascript
// PCI実装内
getInstance(dom, config, state) {
    // 親要素からwriting-modeを判定
    const $container = $(dom);
    const isVertical = $container.closest('.writing-mode-vertical-rl').length > 0;

    if (isVertical) {
        this.initVerticalMode();
    }
    // ...
}
```

**メリット:**
- TAO側の実装変更不要
- PCI側のみで完結

**デメリット:**
- TAOのDOM構造に依存
- writing-modeクラス名の変更に弱い

---

#### 方針4: イベントリスナーによる動的対応

**概要:** `item-writing-mode-changed`イベントをリッスンして、PCI内部を動的に更新する。

**実装箇所:** PCIレンダラーまたはPCI自体

```javascript
// TAO側（PortableCustomInteraction.js への追加）
var render = function render(interaction, options) {
    // 既存の処理...

    // writing-mode変更イベントのリスナー登録
    const $itemBody = containerHelper.get(interaction).closest('.qti-itemBody');
    $itemBody.on('item-writing-mode-changed.pci', function() {
        const isVertical = $itemBody.hasClass('writing-mode-vertical-rl');
        const pci = instanciator.getPci(interaction);
        if (pci && typeof pci.onWritingModeChanged === 'function') {
            pci.onWritingModeChanged(isVertical ? 'vertical-rl' : 'horizontal-tb');
        }
    });
};
```

**PCI側での対応:**
```javascript
// PCI実装内
onWritingModeChanged(writingMode) {
    if (writingMode === 'vertical-rl') {
        this.switchToVerticalMode();
    } else {
        this.switchToHorizontalMode();
    }
}
```

**メリット:**
- writing-modeの動的な変更に対応可能
- アイテムとPCIの同期が保たれる

**デメリット:**
- TAO側の実装変更が必要
- PCIに追加のコールバック実装が必要

---

### 推奨実装方針

| 優先度 | 方針 | 理由 |
|-------|------|------|
| 1 | 方針1（CSS継承）+ 方針3（DOM参照） | 最小限の変更で実現可能 |
| 2 | 方針2（API追加） | より堅牢だが、TAO側の変更が必要 |
| 3 | 方針4（イベント）| 動的変更対応が必要な場合 |

**段階的実装案:**

1. **フェーズ1**: CSS継承とDOM参照で基本的な縦書き対応を実現
2. **フェーズ2**: 必要に応じてPCI APIにwriting-mode情報を追加
3. **フェーズ3**: 動的変更が必要な場合はイベントリスナーを追加

---

## 関連ファイル一覧

| リポジトリ | ファイルパス | 役割 |
|-----------|-------------|------|
| extension-tao-itemqti | `views/js/qtiCreator/widgets/item/states/Active.js` | アイテムのwriting-mode設定 |
| extension-tao-itemqti | `views/js/qtiCreator/widgets/static/helpers/verticalWritingEditing.js` | writing-modeサポート判定 |
| tao-item-runner-qti-fe | `src/qtiCommonRenderer/helpers/verticalWriting.js` | writing-mode判定ヘルパー |
| tao-item-runner-qti-fe | `src/qtiCommonRenderer/renderers/interactions/ChoiceInteraction.js` | 標準インタラクションでの活用例 |
| tao-item-runner-qti-fe | `src/qtiCommonRenderer/renderers/interactions/PortableCustomInteraction.js` | PCIレンダラー |
| tao-item-runner-qti-fe | `src/qtiCommonRenderer/renderers/interactions/pci/ims.js` | IMS PCIレンダラー |
| tao-item-runner-qti-fe | `src/qtiCommonRenderer/tpl/item.tpl` | アイテムテンプレート |

---

## まとめ

### 標準インタラクションの追従メカニズム

1. **アイテム設定**: Active.jsでitemBodyに`writing-mode-vertical-rl`クラスを設定
2. **CSS継承**: CSSの`writing-mode`プロパティが子要素（インタラクション）に継承
3. **追加処理**: レンダラーで`getIsWritingModeVerticalRl()`を使用して縦書き対応処理を実行

### PCIへの実装方針

- **最小実装**: CSS継承を活用し、必要に応じてDOM参照でwriting-modeを判定
- **推奨実装**: PCI APIにwriting-mode情報を追加し、明示的な制御を可能にする
- **高度な実装**: イベントリスナーで動的なwriting-mode変更に対応

---

## 参考リンク

- [Active.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/item/states/Active.js)
- [verticalWriting.js (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiCommonRenderer/helpers/verticalWriting.js)
- [ChoiceInteraction.js (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiCommonRenderer/renderers/interactions/ChoiceInteraction.js)
- [PortableCustomInteraction.js (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiCommonRenderer/renderers/interactions/PortableCustomInteraction.js)
- [ims.js (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiCommonRenderer/renderers/interactions/pci/ims.js)
