# クラス方式 vs プロパティ方式 検証結果

## 結論

ユーザーの分析は**おおむね正確**ですが、いくつかの重要な訂正と追加事項があります。

---

## 検証結果サマリー

| ユーザーの主張 | 検証結果 | 備考 |
|--------------|---------|------|
| Widget.jsからクラス方式で保存可能 | **正しい** | `addClass()`→`attr()`→`editable mixin`→`attributeChange.qti-widget`→自動保存 |
| Widget.jsからプロパティ方式で保存不可 | **誤り** | `prop()`も保存可能。ただし`attributeChange`イベントではなく別の経路 |
| Question.jsから両方式で保存可能 | **正しい** | |
| アイテム変更時のクラス方式の自動同期 | **正しい** | |
| アイテム変更時のプロパティ方式の同期不可 | **誤り** | Widget.jsで`prop('writingMode', undefined)`は可能で、保存もされる |
| 標準インタラクションとの一貫性はクラス方式 | **正しい** | |
| 既存PCIはプロパティを使用 | **ほぼ正しい** | 1つだけ例外あり（audioRecordingInteraction） |

---

## 1. 重要な訂正: Widget.jsからプロパティ方式でも保存可能

ユーザーの分析では「Widget.jsからプロパティ方式で保存不可」としていますが、これは**誤り**です。

### 実際の動作

`prop()`メソッドもWidget.jsから呼び出し可能であり、プロパティの変更は保存されます。

既に調査ドキュメント `investigation-writing-mode-persistence.md` にも、Widget.jsでの実装例が記載されています：

```javascript
$itemBody.on('item-writing-mode-changed', function() {
    self.element.prop('writingMode', undefined);
});
```

**つまり、プロパティ方式でもアイテム変更時の自動同期は可能です。**

### 保存経路の違い

| 方式 | 保存経路 |
|------|---------|
| クラス方式 | `addClass()`→`attr('class', ...)`→`editable mixin`の`attr()`→`attributeChange.qti-widget`イベント発火→自動保存 |
| プロパティ方式 | `prop()`→`this.properties[name] = value`→（保存トリガーは別経路） |

プロパティの変更がどの経路で保存されるかは、`prop()`メソッド自体は`attributeChange.qti-widget`イベントを発火しません。しかし、QTI Creatorはアイテムのモデル全体をシリアライズするため、保存アクション（Ctrl+S等）の際にプロパティも含めてXMLに出力されます。

---

## 2. 重要な追加事項: Test Runnerの高さ計算への影響

**ユーザーの分析に欠けている最も重要なポイント**です。

### Test Runnerの writing-mode 検出方法

`tao-test-runner-qti-fe/src/helpers/verticalWriting.js` は**CSSクラスのみ**で writing-mode を検出します：

```javascript
// アイテム全体の判定
export const getIsItemWritingModeVerticalRl = () => {
    const itemBody = $('.qti-itemBody');
    return itemBody.hasClass('writing-mode-vertical-rl');
};

// 特定要素の判定
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

### プロパティ方式の問題

| シナリオ | クラス方式 | プロパティ方式 |
|---------|----------|--------------|
| PCIが縦書き（アイテムは横書き） | `class="writing-mode-vertical-rl"` → Test Runnerが検出 → 正しい高さ計算 | プロパティのみ → Test Runnerが検出**できない** → 誤った高さ計算 |
| PCIが横書き（アイテムは縦書き） | `class="writing-mode-horizontal-tb"` → Test Runnerが検出 → 正しい高さ計算 | プロパティのみ → Test Runnerが検出**できない** → 誤った高さ計算 |

### 具体的な影響

`itemScrolling.js`の`adaptBlockSize()`関数は、ブロック要素の writing-mode を判定して適用するCSSプロパティを決定します：

```javascript
const isBlockVerticalWriting = getIsWritingModeVerticalRl($block);
const isDifferentWritingMode = isBlockVerticalWriting !== isItemVerticalWriting;
const sizeProp = isDifferentWritingMode ? normalSizeProp : maxSizeProp;
```

| アイテム | ブロック | 正しいCSS | クラスなし時のCSS |
|---------|---------|----------|----------------|
| 横書き | 縦書き | `height` | `max-height`（誤り） |
| 縦書き | 横書き | `width` | `max-width`（誤り） |

**プロパティ方式ではクラスが付与されないため、Test Runnerがwriting-modeの違いを検出できず、高さ計算が誤る可能性があります。**

### プロパティ方式でこの問題を解決するには

プロパティ方式を採用する場合、以下のいずれかの対応が必要：

1. **Question.jsでプロパティに応じてクラスも付与する** → 実質的にクラス方式との併用
2. **Test Runner側にPCIプロパティの読み取りロジックを追加する** → Test Runnerの改修が必要
3. **PCI内部のrenderingでクラスを付与する** → Test Runner実行時のHTML出力時にクラスを含める

---

## 3. 既存PCIの設定保存パターン（調査結果）

### 調査対象

`extension-tao-itemqti-pci` リポジトリ内の既存PCI：

| PCI | 設定の保存方法 |
|-----|--------------|
| likertScoreInteraction | **プロパティ** (`prop('level', value)`) |
| likertCompact | **プロパティ** |
| likertConfig | **プロパティ** |
| mathEntryInteraction | **プロパティ** (`prop(name, value)`) |
| audioRecordingInteraction (IMS版) | **クラス + プロパティ** (`toggleClass('sequential', value)`) |
| audioRecordingInteraction (dev版) | **プロパティのみ** |
| liquidsInteraction | **プロパティ** |

### 結論

- **ほぼ全てのPCIがプロパティ方式を使用**
- **唯一の例外**: audioRecordingInteraction (IMS版) が `sequential` フラグに `toggleClass()` を使用
- ただし、これは writing-mode のような視覚的なCSS設定ではなく、動作制御フラグ

---

## 4. QTI XMLシリアライゼーションの検証

### クラス方式の場合

`customInteraction`のXMLテンプレート（`portableCustomInteraction/main.tpl`）：

```handlebars
<customInteraction {{{join attributes '=' ' ' ' '}}}}>
    {{{portableCustomInteraction}}}
</customInteraction>
```

`{{{join attributes ...}}}` がすべてのattributesを出力するため、`class`属性も正しくXMLに含まれます。

**出力例:**
```xml
<customInteraction responseIdentifier="RESPONSE" class="writing-mode-vertical-rl">
    <pci:portableCustomInteraction ...>
        ...
    </pci:portableCustomInteraction>
</customInteraction>
```

### 保存経路の確認

```
addClass('writing-mode-vertical-rl')
    ↓
Element.addClass() → this.attr('class', 'writing-mode-vertical-rl')
    ↓
editable mixin の attr() がオーバーライド
    ↓
$(document).trigger('attributeChange.qti-widget', { element, key: 'class', value })
    ↓
QTI XMLに自動保存
```

**`choiceInteraction`と完全に同じメカニズムです。** クラス方式は技術的に問題なく動作します。

---

## 5. 修正された比較表

### QTI XMLでの保存形式

クラス方式:
```xml
<customInteraction class="writing-mode-vertical-rl" responseIdentifier="RESPONSE">
    <pci:portableCustomInteraction>
        ...
    </pci:portableCustomInteraction>
</customInteraction>
```

プロパティ方式:
```xml
<customInteraction responseIdentifier="RESPONSE">
    <pci:portableCustomInteraction>
        <pci:properties>
            <pci:entry key="writingMode">vertical</pci:entry>
        </pci:properties>
    </pci:portableCustomInteraction>
</customInteraction>
```

### 修正された技術的比較

| 観点 | クラス方式 | プロパティ方式 |
|------|----------|--------------|
| Widget.jsから保存 | 可能（`attributeChange`イベント経由） | 可能（モデルシリアライズ時に保存） |
| Question.jsから保存 | 可能 | 可能 |
| アイテム変更時の自動同期 | 可能（`removeClass()`） | 可能（`prop(name, undefined)`） |
| 標準インタラクションとの一貫性 | **同じ仕組み** | 異なる仕組み |
| 既存PCIとの一貫性 | 一部例外あり | **ほぼ全てのPCIがこちら** |
| Test Runnerでの writing-mode 検出 | **自動的に機能** | **機能しない**（追加対応が必要） |
| CSS継承 | **自然に機能** | PCI内部で別途対応が必要 |
| PCI仕様への準拠 | グレー（属性は仕様外の拡張） | **準拠** |

---

## 6. 修正された判断マトリクス

| 優先事項 | 推奨方式 | 理由 |
|---------|---------|------|
| 標準インタラクションと同じ動作 | クラス方式 | 同じメカニズム |
| 既存PCIとの一貫性 | プロパティ方式 | ほぼ全てのPCIがプロパティを使用 |
| 実装のシンプルさ | クラス方式 | CSS継承が自然に機能 |
| アイテム変更時の即座同期 | **どちらも可能** | ~~クラス方式のみ~~ |
| PCI仕様への準拠 | プロパティ方式 | |
| Test Runnerの高さ計算 | **クラス方式** | プロパティ方式はTest Runner改修が必要 |
| CSS自然継承 | **クラス方式** | プロパティ方式は別途CSS適用が必要 |

---

## 7. 最終的な見解

### クラス方式を推奨する理由（ユーザーの見解は正しい）

ユーザーの「技術的制約がなければクラス方式が望ましい」という見解は正しいです。加えて、以下の理由がさらにクラス方式を支持します：

1. **Test Runnerの高さ計算が正しく動作する** — `verticalWriting.js`がクラスを検出するため、追加改修不要
2. **CSS継承が自然に機能する** — ブラウザの`writing-mode`プロパティ継承がそのまま使える
3. **`audioRecordingInteraction`という前例がある** — PCI でクラス属性を使用した実績がある
4. **保存メカニズムは標準インタラクションと完全に同じ** — `editable mixin`の`attr()`を通じて自動保存

### ただし注意すべき点

- **既存PCIの大半はプロパティ方式** — チーム内の慣例に反する可能性がある
- **PCI仕様としては、設定はpropertiesに入れるのが正道** — classはPCI仕様が想定する設定保存場所ではない
- **ハイブリッド方式の検討** — プロパティに保存しつつ、Question.jsやWidget.jsでクラスも付与する方式が最も安全かもしれない

### ハイブリッド方式の例

```javascript
// Widget.js
$itemBody.on('item-writing-mode-changed', function() {
    // プロパティを削除
    self.element.prop('writingMode', undefined);
    // クラスも削除（Test Runner用）
    self.element.removeClass('writing-mode-vertical-rl');
    self.element.removeClass('writing-mode-horizontal-tb');
});

// Question.js callbacks.writingMode
if (mode === 'vertical' && !isItemVertical) {
    interaction.prop('writingMode', 'vertical');
    interaction.addClass('writing-mode-vertical-rl');
    interaction.removeClass('writing-mode-horizontal-tb');
} else if (mode === 'horizontal' && isItemVertical) {
    interaction.prop('writingMode', 'horizontal');
    interaction.addClass('writing-mode-horizontal-tb');
    interaction.removeClass('writing-mode-vertical-rl');
} else {
    interaction.prop('writingMode', undefined);
    interaction.removeClass('writing-mode-vertical-rl');
    interaction.removeClass('writing-mode-horizontal-tb');
}
```

この方式なら：
- PCIの慣例（プロパティ）に従う
- Test Runnerの高さ計算が正しく動作する
- CSS継承も機能する
- ただし、二重管理のため整合性に注意が必要
