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
| oat-sa/tao-test-runner-qti-fe | `src/helpers/verticalWriting.js` | 縦書き/横書きモード判定 |

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

#### プラグインの初期化フロー

```javascript
testRunner
    .on('renderitem', () => {
        // 1. ルートコンテナ取得
        $root = testRunner.getAreaBroker().getContainer();

        // 2. アイテムのwriting-mode判定
        const isItemVerticalWriting = getIsItemWritingModeVerticalRl();

        // 3. スクロールコンテナの選択（writing-modeにより異なる）
        const $itemScrollContainer = getItemScrollContainer(isItemVerticalWriting);

        // 4. ResizeObserverでサイズ変更を監視
        this.itemResizeObserver = new ResizeObserver(this.itemResizeCallback);
        this.itemResizeObserver.observe($itemScrollContainer.get(0));
    })
```

#### スクロールコンテナの選択ロジック

**重要:** アイテムのwriting-modeによって、監視対象のコンテナが異なります。

```javascript
function getItemScrollContainer(isItemVerticalWriting) {
    return isItemVerticalWriting
        ? $root.find('.qti-itemBody')                              // 縦書き: itemBody
        : $root.find('.test-runner-sections > .content-wrapper');  // 横書き: content-wrapper
}
```

| writing-mode | 対象コンテナ |
|--------------|-------------|
| 縦書き (`vertical-rl`) | `.qti-itemBody` |
| 横書き (`horizontal-tb`) | `.test-runner-sections > .content-wrapper` |

---

### 3. `adaptBlockSize()` 関数 - px値計算の核心

この関数がpx値を計算・適用する核心部分です。

```javascript
function adaptBlockSize() {
    const isItemVerticalWriting = getIsItemWritingModeVerticalRl();
    const $itemScrollContainer = getItemScrollContainer(isItemVerticalWriting);
    const $blockContainers = $itemScrollContainer.find('[data-scrolling="true"]');

    // ステップ1: 内部サイズ計算
    const innerItemSize =
        getItemRunnerBlockSize($itemScrollContainer, isItemVerticalWriting) -
        getQtiItemAndItemBodyPadding($itemScrollContainer, isItemVerticalWriting) -
        2;  // ボーダー調整分

    // ステップ2: コンテンツブロックサイズ計算
    const contentBlockSize =
        innerItemSize -
        getGridRowBlockMargin() -
        getExtraGridRowBlockSize(isItemVerticalWriting) -
        getSpaceAroundQtiContent($itemScrollContainer, isItemVerticalWriting);

    // ステップ3: 各スクロールブロックにpx値を適用
    $blockContainers.each(function () {
        const $block = $(this);

        // ブロック自体のwriting-mode判定
        const isBlockVerticalWriting = getIsWritingModeVerticalRl($block);
        const isDifferentWritingMode = isBlockVerticalWriting !== isItemVerticalWriting;

        // パーセンテージ値を取得（デフォルト: 100）
        const selectedBlockSize = parseFloat($block.attr('data-scrolling-height')) || 100;

        // ネストされたコンテナの親を取得
        const containerParent = $block.parent().closest('[data-scrolling="true"]');
        const containerBlockSize = isItemVerticalWriting
            ? containerParent.width()
            : containerParent.height();

        // CSSプロパティの決定（後述）
        const normalSizeProp = isItemVerticalWriting ? 'width' : 'height';
        const maxSizeProp = isItemVerticalWriting ? 'max-width' : 'max-height';
        const sizeProp = isDifferentWritingMode ? normalSizeProp : maxSizeProp;

        // px値の計算と適用
        const cssObj = {};
        if (containerParent.length > 0) {
            // ネストされたコンテナの場合：親のサイズを基準
            cssObj[sizeProp] = `${containerBlockSize * (selectedBlockSize * 0.01)}px`;
        } else {
            // トップレベルの場合：contentBlockSizeを基準
            const maxSize = contentBlockSize * (selectedBlockSize * 0.01);
            if (maxSize > minimalAcceptableSizePx) {
                cssObj[sizeProp] = `${maxSize}px`;
            }
        }

        // CSSにpx値を適用
        $block.css(cssObj);
    });
}
```

---

## 適用されるCSSプロパティの決定ロジック

**重要:** writing-modeによって適用されるCSSプロパティが異なります。

### 判定条件

| アイテムのwriting-mode | ブロックのwriting-mode | 適用プロパティ |
|----------------------|---------------------|--------------|
| 横書き | 横書き（同じ） | `max-height` |
| 横書き | 縦書き（異なる） | `height` |
| 縦書き | 縦書き（同じ） | `max-width` |
| 縦書き | 横書き（異なる） | `width` |

### コードでの実装

```javascript
// アイテムのwriting-modeに基づく基本プロパティ
const normalSizeProp = isItemVerticalWriting ? 'width' : 'height';
const maxSizeProp = isItemVerticalWriting ? 'max-width' : 'max-height';

// ブロックとアイテムのwriting-modeが異なる場合は通常プロパティ、同じ場合はmaxプロパティ
const sizeProp = isDifferentWritingMode ? normalSizeProp : maxSizeProp;
```

**理由:** 異なるwriting-modeの場合、`max-height`/`max-width`だと要素が適切にサイズ調整されないため、明示的な`height`/`width`を使用します。

---

## Writing-Mode判定ロジック

**ファイル:** `tao-test-runner-qti-fe/src/helpers/verticalWriting.js`

```javascript
// アイテム全体のwriting-mode判定
export const getIsItemWritingModeVerticalRl = () => {
    const itemBody = $('.qti-itemBody');
    return itemBody.hasClass('writing-mode-vertical-rl');
};

// 特定要素のwriting-mode判定
export const getIsWritingModeVerticalRl = $container => {
    const $writingModeParent = $container.closest(
        '.writing-mode-vertical-rl, .writing-mode-horizontal-tb'
    );
    return Boolean(
        $writingModeParent.length &&
        $writingModeParent.hasClass('writing-mode-vertical-rl')
    );
};
```

判定基準は、要素またはその親に`writing-mode-vertical-rl`クラスが付与されているかどうかです。

---

## 最終px値の算出フロー

```
┌─────────────────────────────────────────────────────────────────┐
│                      px値算出の流れ                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. getItemScrollContainer() でスクロールコンテナを選択          │
│     ├→ 縦書き: .qti-itemBody                                    │
│     └→ 横書き: .test-runner-sections > .content-wrapper         │
│                                                                 │
│  2. getItemRunnerBlockSize() でコンテナのサイズ取得              │
│     └→ getBoundingClientRect() で実際のwidth/heightを取得       │
│        （縦書き: width、横書き: height）                         │
│                                                                 │
│  3. innerItemSize を計算                                        │
│     └→ コンテナサイズ - パディング - 2（ボーダー調整）            │
│                                                                 │
│  4. contentBlockSize を計算                                     │
│     └→ innerItemSize                                            │
│        - getGridRowBlockMargin()        // マージン減算          │
│        - getExtraGridRowBlockSize()     // 他行のサイズ減算      │
│        - getSpaceAroundQtiContent()     // 周辺スペース減算      │
│                                                                 │
│  5. 最終px値 = contentBlockSize × (パーセンテージ × 0.01)        │
│     例: 600px × (50 × 0.01) = 300px                             │
│                                                                 │
│  6. sizePropを決定                                              │
│     ├→ 同じwriting-mode: max-height / max-width                 │
│     └→ 異なるwriting-mode: height / width                       │
│                                                                 │
│  7. $block.css({ [sizeProp]: '300px' }) で適用                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 補助関数の役割

| 関数名 | 役割 | 詳細 |
|--------|------|------|
| `getItemRunnerBlockSize()` | コンテナの実サイズを取得 | `getBoundingClientRect()`で取得。縦書きは`width`、横書きは`height`を返す |
| `getQtiItemAndItemBodyPadding()` | パディング合計を計算 | itemScrollContainerからitemBodyまでの各要素のパディングを合計 |
| `getGridRowBlockMargin()` | grid-rowのマージン計算 | スクロールコンテナを含むgrid-rowのmargin-block-start/endを合計 |
| `getExtraGridRowBlockSize()` | 他行のサイズ計算 | スクロールコンテナを**含まない**grid-rowのサイズを合計 |
| `getSpaceAroundQtiContent()` | 周辺スペース計算 | rubrickブロックなど、qti-contentの上部スペースを計算（横書きのみ） |

---

## ネストされたスクロールコンテナの処理

スクロールコンテナがネストされている場合（親にも`data-scrolling="true"`がある場合）、計算方法が異なります。

```javascript
const containerParent = $block.parent().closest('[data-scrolling="true"]');

if (containerParent.length > 0) {
    // 親コンテナのサイズを基準にパーセンテージ計算
    const containerBlockSize = isItemVerticalWriting
        ? containerParent.width()
        : containerParent.height();
    cssObj[sizeProp] = `${containerBlockSize * (selectedBlockSize * 0.01)}px`;
} else {
    // トップレベル: contentBlockSizeを基準
    cssObj[sizeProp] = `${contentBlockSize * (selectedBlockSize * 0.01)}px`;
}
```

---

## 最小サイズの保護

計算結果が極端に小さくなる場合の保護処理があります：

```javascript
const minimalAcceptableSizePx = 20;

if (maxSize > minimalAcceptableSizePx) {
    cssObj[sizeProp] = `${maxSize}px`;
} else {
    // 計算結果が小さすぎる場合はスクロールバーなしで自然サイズ表示
    // ただし、異なるwriting-modeの場合は innerItemSize / 2 を使用
    if (isBlockVerticalWriting !== isItemVerticalWriting) {
        cssObj[sizeProp] = `${innerItemSize / 2}px`;
    }
    // 同じwriting-modeでサイズが小さすぎる場合はcssObjを設定しない
    // → 自然なサイズで表示（スクロールなし）
}
```

**注意:** 最小サイズ保護が発動するケース：
- `getExtraGridRowBlockSize()`で他のgrid-rowが予想以上に大きい場合
- `getSpaceAroundQtiContent()`で周辺スペースが大きい場合

---

## ResizeObserverによる動的更新

ウィンドウサイズ変更時にpx値が自動的に再計算されます：

```javascript
// 200msのスロットリング + requestAnimationFrameで最適化
this.itemResizeCallback = _.throttle(() => requestAnimationFrame(adaptBlockSize), 200);
this.itemResizeObserver = new ResizeObserver(this.itemResizeCallback);
this.itemResizeObserver.observe($itemScrollContainer.get(0));
```

**クリーンアップ:** `unloaditem`イベントとプラグインの`destroy()`でObserverを解除します。

---

## 関連CSSスタイル

**ファイル:** `tao-test-runner-qti-fe/src/plugins/content/itemScrolling/scss/scrolling.scss`

```scss
[data-scrolling='true'] {
    overflow-y: auto;
    overflow-x: auto;

    /* 異なるwriting-modeのブロック用 */
    &.writing-mode-vertical-rl {
        width: 100%;
    }
    &.writing-mode-horizontal-tb {
        height: 100%;
    }
}
```

---

## まとめ

| 段階 | 場所 | 処理内容 |
|------|------|----------|
| **設定時** | itemScrollingMethods.js | `data-scrolling-height`にパーセンテージ値（100, 75, 50等）を保存 |
| **実行時** | itemScrolling.js | `getBoundingClientRect()`でコンテナサイズ取得 → 各種調整値を減算 → パーセンテージ計算 → **px値でCSS適用** |

### 確認結果

**最終的にpx値で指定されることを確認しました。**

`$block.css()` メソッドを通じて、計算されたpx値が直接インラインスタイルとして適用されます。

適用されるCSSプロパティは：
- 横書きアイテム内の横書きブロック: `max-height: Xpx`
- 横書きアイテム内の縦書きブロック: `height: Xpx`
- 縦書きアイテム内の縦書きブロック: `max-width: Xpx`
- 縦書きアイテム内の横書きブロック: `width: Xpx`

---

## 参考リンク

- [itemScrollingMethods.js (GitHub)](https://github.com/oat-sa/extension-tao-itemqti/blob/master/views/js/qtiCreator/widgets/static/helpers/itemScrollingMethods.js)
- [itemScrolling.js (GitHub)](https://github.com/oat-sa/tao-test-runner-qti-fe/blob/master/src/plugins/content/itemScrolling/itemScrolling.js)
- [verticalWriting.js (GitHub)](https://github.com/oat-sa/tao-test-runner-qti-fe/blob/master/src/helpers/verticalWriting.js)
- [scrolling.scss (GitHub)](https://github.com/oat-sa/tao-test-runner-qti-fe/blob/master/src/plugins/content/itemScrolling/scss/scrolling.scss)
