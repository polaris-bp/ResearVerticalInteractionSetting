# Writing-Mode永続化メカニズムとPCIへの適用に関する調査レポート

## 問題の背景

PCIを配置後、アイテム一覧画面に戻り、再度アイテムを選択し、**PCIを一度も選択せず**にアイテムの文字列方向を変更した場合、PCIのプログラムが走っていない状態ではwriting-mode情報を取り込めないという懸念があります。

---

## 標準インタラクションでの対処方法

### 核心的な発見

**標準インタラクションは、個別にwriting-mode情報を保存していません。**

writing-modeは**アイテムレベル**で保存され、**レンダリング時に親要素（itemBody）から継承**されます。

---

### 永続化メカニズムの詳細

#### 1. 保存時の流れ

```
┌─────────────────────────────────────────────────────────────────┐
│                     保存時の流れ                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ユーザーがアイテムのwriting-modeを変更                        │
│     └→ Active.js: writingModeItem コールバック                   │
│                                                                 │
│  2. アイテムのQTIモデルにクラスを追加                             │
│     └→ item.addClass('writing-mode-vertical-rl')                │
│     └→ Element.js: this.attr('class', ...) でclass属性を更新    │
│                                                                 │
│  3. アイテム保存時にQTI XMLへシリアライズ                         │
│     └→ <assessmentItem class="writing-mode-vertical-rl" ...>    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**コード（Active.js）:**
```javascript
writingModeItem(i, mode) {
    if (mode === 'vertical') {
        item.addClass(writingModeVerticalRlClass);  // QTIモデルのclass属性を更新
    } else {
        itemRemoveClasses([writingModeVerticalRlClass]);
    }
    $itemBody.trigger('item-writing-mode-changed');
}
```

**コード（Element.js）:**
```javascript
addClass: function (className) {
    var clazz = this.attr('class') || '';
    if (!_containClass(clazz, className)) {
        this.attr('class', clazz + (clazz.length ? ' ' : '') + className);
    }
}
```

#### 2. 読み込み時の流れ

```
┌─────────────────────────────────────────────────────────────────┐
│                     読み込み時の流れ                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. QTI XMLからアイテムを読み込み                                 │
│     └→ Loader.js: loadItemData()                                │
│     └→ アイテムのattributes（class含む）が復元される              │
│                                                                 │
│  2. アイテムをレンダリング                                       │
│     └→ item.tpl テンプレートを使用                               │
│                                                                 │
│  3. itemBodyにクラスが適用される                                 │
│     └→ <div class="qti-itemBody writing-mode-vertical-rl">      │
│                                                                 │
│  4. 子要素（インタラクション）がレンダリングされる                 │
│     └→ CSSの継承によりwriting-modeが自動適用                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**テンプレート（item.tpl）:**
```html
<div class="qti-itemBody {{#if attributes.class}} {{attributes.class}}{{/if}}">
    {{{body}}}  <!-- インタラクションはここにレンダリングされる -->
</div>
```

#### 3. 重要なポイント

| ポイント | 説明 |
|---------|------|
| **保存場所** | アイテムの`class`属性（QTI XMLの`assessmentItem`要素） |
| **インタラクション個別の保存** | **なし** - インタラクションは個別にwriting-modeを保存しない |
| **伝播方法** | itemBodyのクラス属性 → CSSの継承 |
| **タイミング** | レンダリング時（インタラクションの初期化前に適用済み） |

---

## PCIへの適用可能性

### 結論

**標準インタラクションと同様の方法でPCIも対処可能です。**

理由：
1. writing-modeはアイテムレベルで保存される
2. itemBodyにクラスとして適用される
3. PCIはitemBodyの子要素としてレンダリングされる
4. CSSの継承により、PCIにもwriting-modeが自動適用される
5. PCI初期化時点で、既にDOMにwriting-modeが設定されている

### 詳細説明

```
┌─────────────────────────────────────────────────────────────────┐
│                     PCIでの動作                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. アイテム読み込み                                             │
│     └→ QTI XMLからアイテムのclass属性が復元される                 │
│                                                                 │
│  2. アイテムレンダリング                                         │
│     └→ itemBodyに writing-mode-vertical-rl クラスが適用          │
│     └→ CSS: writing-mode: vertical-rl が有効になる               │
│                                                                 │
│  3. PCI要素のレンダリング（HTML挿入）                            │
│     └→ PCIのmarkupがitemBody内に配置される                       │
│     └→ この時点でCSSの継承によりwriting-modeが適用済み            │
│                                                                 │
│  4. PCIのJavaScript初期化                                       │
│     └→ getInstance() が呼ばれる                                  │
│     └→ DOM要素には既にwriting-modeが適用されている               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### PCIが追加で対応すべきこと

#### 1. CSSの継承を妨げないようにする

```css
/* NG: writing-modeを上書きしてしまう */
.my-pci-container {
    writing-mode: horizontal-tb;  /* 親のwriting-modeが無効になる */
}

/* OK: writing-modeを継承する */
.my-pci-container {
    /* writing-modeは指定しない（親から継承） */
}
```

#### 2. JavaScript内でwriting-modeを判定する場合

PCIの初期化時点でDOMにwriting-modeが適用されているため、DOM参照で判定可能です。

```javascript
// PCI実装内
getInstance(dom, config, state) {
    // 親要素からwriting-modeを判定
    const $container = $(dom);
    const isVertical = $container.closest('.writing-mode-vertical-rl').length > 0;

    // または、CSSのcomputed styleから判定
    const writingMode = getComputedStyle(dom).writingMode;
    const isVertical = writingMode === 'vertical-rl';

    if (isVertical) {
        this.initVerticalMode();
    }
}
```

---

## 「PCIプログラムが走っていない状態」の解釈

### シナリオの整理

ユーザーが懸念されているシナリオ：
1. PCIをアイテムに配置
2. アイテム一覧画面に戻る
3. 再度アイテムを選択
4. **PCIを一度も選択せず**（PCIのwidgetがアクティブにならない）
5. アイテムの文字列方向を変更

### 標準インタラクションでの動作

標準インタラクションでも、このシナリオでは個々のインタラクションのwidgetは初期化されません。

しかし、問題は発生しません。理由：
- writing-modeはアイテムレベルで保存される
- インタラクションは個別にwriting-modeを保存しない
- 次回レンダリング時にitemBodyから継承される

### PCIでも同様

PCIも同様に：
- writing-modeはアイテムレベルで保存される
- PCI自体にwriting-modeを保存する必要はない
- 次回レンダリング時（テスト実行時など）にitemBodyから継承される

---

## 注意が必要なケース

### ケース1: PCIが独自にwriting-mode情報を保持したい場合

もしPCIがpropertiesとしてwriting-modeを保存している場合、アイテムのwriting-modeとの不整合が発生する可能性があります。

**解決策:** PCI自体にwriting-modeを保存しない。常にアイテム（親要素）から継承する。

### ケース2: オーサリング画面でのリアルタイム反映

オーサリング画面でアイテムのwriting-modeを変更した際、PCIのプレビューにリアルタイムで反映したい場合。

**現状の動作:**
- `item-writing-mode-changed`イベントが発火される
- PCIがこのイベントをリッスンしていなければ、リアルタイム反映されない

**解決策:**
1. PCIがイベントをリッスンして再描画する
2. または、CSSの継承に任せる（CSSベースの表示はリアルタイムで反映される）

```javascript
// オプション: オーサリング画面でリアルタイム反映が必要な場合
const $itemBody = $(dom).closest('.qti-itemBody');
$itemBody.on('item-writing-mode-changed', () => {
    const isVertical = $itemBody.hasClass('writing-mode-vertical-rl');
    this.updateWritingMode(isVertical);
});
```

---

## 推奨実装方針

### 基本方針

| 項目 | 推奨 |
|------|------|
| writing-modeの保存 | アイテムレベルで保存（現状維持） |
| PCI独自のwriting-mode保存 | **しない**（アイテムから継承） |
| CSS対応 | writing-modeを上書きしない |
| JavaScript対応 | 初期化時にDOM/computed styleから判定 |

### 実装例

```javascript
// PCI実装のベストプラクティス
const myPCI = {
    getInstance(dom, config, state) {
        // 1. writing-modeをDOMから判定
        const writingMode = getComputedStyle(dom).writingMode;
        const isVertical = writingMode === 'vertical-rl';

        // 2. 判定結果に基づいて初期化
        this.isVerticalWriting = isVertical;
        this.initLayout(isVertical);

        // 3. オーサリング画面用のイベントリスナー（オプション）
        const $itemBody = $(dom).closest('.qti-itemBody');
        if ($itemBody.length) {
            $itemBody.on('item-writing-mode-changed.myPCI', () => {
                const newIsVertical = $itemBody.hasClass('writing-mode-vertical-rl');
                if (this.isVerticalWriting !== newIsVertical) {
                    this.isVerticalWriting = newIsVertical;
                    this.updateLayout(newIsVertical);
                }
            });
        }
    },

    oncompleted() {
        // クリーンアップ
        $(this.dom).closest('.qti-itemBody').off('.myPCI');
    }
};
```

---

## まとめ

### 標準インタラクションの対処方法

- writing-modeは**アイテムレベル**で保存
- インタラクションは**個別に保存しない**
- **レンダリング時にitemBodyから継承**

### PCIへの適用

- **同様の方法で対処可能**
- PCI独自にwriting-modeを保存する必要はない
- CSS継承とDOM参照で対応可能
- オーサリング画面でのリアルタイム反映が必要な場合のみ、イベントリスナーを追加

### 懸念への回答

「PCIプログラムが走っていない状態でアイテム設定が更新される」ケースは、標準インタラクションでも同様です。writing-modeがアイテムレベルで保存されるため、次回レンダリング時に正しく適用されます。

---

## 参考リンク

- [Active.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/item/states/Active.js)
- [Element.js (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiItem/core/Element.js)
- [item.tpl (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiCommonRenderer/tpl/item.tpl)
- [Loader.js (GitHub)](https://github.com/oat-sa/tao-item-runner-qti-fe/blob/master/src/qtiItem/core/Loader.js)
