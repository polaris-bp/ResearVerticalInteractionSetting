# 標準インタラクション高さ設定の調査結果

## 概要

標準インタラクションの高さ設定がどのように設定されるかを調査しました。

**結論: px値で最終的に指定されることを確認しました。**

---

## 関連ファイル

| リポジトリ | ファイルパス | 役割 |
|-----------|-------------|------|
| oat-sa/extension-tao-itemqti | `views/js/qtiCreator/widgets/static/helpers/itemScrollingMethods.js` | 設定フェーズ（パーセンテージ値の保存） |
| oat-sa/tao-test-runner-qti-fe | `src/plugins/content/itemScrolling/itemScrolling.js` | 実行フェーズ（px値の計算・適用） |
| oat-sa/tao-test-runner-qti-fe | `src/plugins/content/itemScrolling/scss/scrolling.scss` | スクロールスタイル定義 |

---

## 処理の流れ

### 1. 設定フェーズ（QTI Creator側）

**ファイル:** `extension-tao-itemqti/views/js/qtiCreator/widgets/static/helpers/itemScrollingMethods.js`

このファイルでは、6段階の高さオプションが定義されています：

```javascript
const options = [
    { value: '100',     name: 'Full height',      class: 'tao-full-height' },
    { value: '75',      name: '3/4 of height',    class: 'tao-three-quarters-height' },
    { value: '66.6666', name: '2/3 of height',    class: 'tao-two-thirds-height' },
    { value: '50',      name: 'Half height',      class: 'tao-half-height' },
    { value: '33.3333', name: '1/3 of height',    class: 'tao-third-height' },
    { value: '25',      name: '1/4 of height',    class: 'tao-quarter-height' }
];
```

#### 設定時に行われる処理

1. `data-scrolling="true"` 属性を設定
2. `data-scrolling-height` 属性にパーセンテージ値（例: `"50"`）を設定
3. CSSクラス（例: `tao-half-height`）を付与

```javascript
// wrapContent関数内
$wrapper.attr('data-scrolling', value);
$wrapper.attr('data-scrolling-height', opt.value);
$wrapper.addClass(`${newUIclass} ${opt.class}`);
```

---

### 2. 実行フェーズ（Test Runner側）- px値計算の核心

**ファイル:** `tao-test-runner-qti-fe/src/plugins/content/itemScrolling/itemScrolling.js`

#### `adaptBlockSize()` 関数

この関数がpx値を計算・適用する核心部分です。

```javascript
function adaptBlockSize() {
    // コンテナサイズの計算
    const innerItemSize =
        getItemRunnerBlockSize($itemScrollContainer, isItemVerticalWriting) -
        getQtiItemAndItemBodyPadding($itemScrollContainer, isItemVerticalWriting) -
        2;

    const contentBlockSize =
        innerItemSize -
        getGridRowBlockMargin() -
        getExtraGridRowBlockSize(isItemVerticalWriting) -
        getSpaceAroundQtiContent($itemScrollContainer, isItemVerticalWriting);

    $blockContainers.each(function () {
        const $block = $(this);

        // パーセンテージ値を取得
        const selectedBlockSize = parseFloat($block.attr('data-scrolling-height')) || 100;

        // px値を計算して適用
        if (containerParent.length > 0) {
            // 親コンテナ内にある場合
            cssObj[sizeProp] = `${containerBlockSize * (selectedBlockSize * 0.01)}px`;
        } else {
            // トップレベルの場合
            const maxSize = contentBlockSize * (selectedBlockSize * 0.01);
            cssObj[sizeProp] = `${maxSize}px`;
        }

        // CSSにpx値を適用
        $block.css(cssObj);
    });
}
```

---

## 最終px値の算出フロー

```
┌─────────────────────────────────────────────────────────────┐
│                    px値算出の流れ                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. getItemRunnerBlockSize() でコンテナのサイズ取得           │
│     └→ getBoundingClientRect() で実際のwidth/heightを取得    │
│                                                             │
│  2. innerItemSize を計算                                    │
│     └→ コンテナサイズ - パディング - 2                        │
│                                                             │
│  3. contentBlockSize を計算                                 │
│     └→ innerItemSize                                        │
│        - getGridRowBlockMargin()        // マージン減算      │
│        - getExtraGridRowBlockSize()     // 他行のサイズ減算   │
│        - getSpaceAroundQtiContent()     // 周辺スペース減算   │
│                                                             │
│  4. 最終px値 = contentBlockSize × (パーセンテージ × 0.01)    │
│     例: 600px × (50 × 0.01) = 300px                         │
│                                                             │
│  5. $block.css({ 'max-height': '300px' }) で適用            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 補助関数の役割

| 関数名 | 役割 |
|--------|------|
| `getItemRunnerBlockSize()` | `getBoundingClientRect()`でコンテナの実サイズを取得 |
| `getQtiItemAndItemBodyPadding()` | itemBodyまでの各要素のパディングを合計 |
| `getGridRowBlockMargin()` | grid-rowのマージンを計算 |
| `getExtraGridRowBlockSize()` | スクロール対象でない他のgrid-rowのサイズを計算 |
| `getSpaceAroundQtiContent()` | rubrickブロックなどの周辺スペースを計算 |

---

## 最小サイズの保護

計算結果が極端に小さくなる場合の保護処理があります：

```javascript
const minimalAcceptableSizePx = 20;

if (maxSize > minimalAcceptableSizePx) {
    cssObj[sizeProp] = `${maxSize}px`;
} else {
    // 計算結果が小さすぎる場合はスクロールバーなしで自然サイズ表示
    // 異なるwriting-modeの場合は innerItemSize / 2 を使用
    if (isBlockVerticalWriting !== isItemVerticalWriting) {
        cssObj[sizeProp] = `${innerItemSize / 2}px`;
    }
}
```

---

## ResizeObserverによる動的更新

ウィンドウサイズ変更時にpx値が再計算されます：

```javascript
this.itemResizeCallback = _.throttle(() => requestAnimationFrame(adaptBlockSize), 200);
this.itemResizeObserver = new ResizeObserver(this.itemResizeCallback);
this.itemResizeObserver.observe($itemScrollContainer.get(0));
```

---

## まとめ

| 段階 | 場所 | 処理内容 |
|------|------|----------|
| **設定時** | itemScrollingMethods.js | `data-scrolling-height`にパーセンテージ値を保存 |
| **実行時** | itemScrolling.js | `getBoundingClientRect()`でコンテナサイズ取得 → パーセンテージ計算 → **px値でCSS適用** |

**確認結果:** 最終的にpx値で指定されることを確認しました。`$block.css()` メソッドを通じて、計算されたpx値が直接インラインスタイルとして適用されます。

---

## 参考リンク

- [itemScrollingMethods.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/static/helpers/itemScrollingMethods.js)
- [itemScrolling.js (GitHub)](https://github.com/oat-sa/tao-test-runner-qti-fe/blob/master/src/plugins/content/itemScrolling/itemScrolling.js)
