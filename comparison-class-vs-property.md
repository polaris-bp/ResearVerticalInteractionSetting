# クラス方式 vs プロパティ方式 検証結果

## 結論

ユーザーの分析は多くの点で正しいが、いくつかの誤解がある。また、**致命的なバグ**と**Test Runnerへの影響**という重要な発見があった。

---

## 0. 前提確認: クラス方式はPCIで実現可能か？

**結論: 可能。** ソースコードで全経路を確認済み。

### QTI仕様

QTI 2.1/2.2 仕様において、`customInteraction`は`bodyElement`を継承しており、`bodyElement`は`class`属性を定義している。したがって `<customInteraction class="writing-mode-vertical-rl">` は**QTI仕様に準拠**している。

### 全経路の検証（ソースコード根拠）

`class`属性がSave → Load の完全なラウンドトリップを生き残るか、6段階すべてを確認した。

#### ① PHP XML パース（ロード時）

**ソース:** `extension-tao-itemqti/model/qti/ParserFactory.php` — `extractAttributes()`

```php
protected function extractAttributes(DOMElement $data)
{
    $options = [];
    foreach ($data->attributes as $attr) {
        if ($attr->nodeName === 'xsi:schemaLocation') { continue; }
        $options[$this->attributeMap[$attr->nodeName] ?? $attr->nodeName] = (string) $attr->nodeValue;
    }
    return $options;
}
```

`xsi:schemaLocation`以外の**全DOM属性**を抽出する。`class`もここで取得される。

#### ② PHP モデルへの格納

**ソース:** `extension-tao-itemqti/model/qti/Element.php` — `setAttribute()`

`Interaction`の`getUsedAttributes()`は`ResponseIdentifier`のみを返す。`class`は既知属性に含まれないため、`Generic`属性オブジェクトとして格納される：

```php
} else {
    $this->attributes[$name] = new Generic($value);
}
```

#### ③ PHP → JSON シリアライズ

**ソース:** `Element.php` — `getAttributeValues()`

```php
public function getAttributeValues($filterNull = true)
{
    $returnValue = [];
    foreach ($this->attributes as $name => $attribute) {
        if (!$filterNull || !$attribute->isNull()) {
            $returnValue[$name] = $attribute->getValue();
        }
    }
    return $returnValue;
}
```

全属性（`responseIdentifier`と`class`を含む）がJSON出力に含まれる。

#### ④ JS モデルへのロード

**ソース:** `tao-item-runner-qti-fe/src/qtiItem/core/Loader.js` — `loadElementData()`

```javascript
const attributes = _.defaults(data.attributes || {}, element.attributes || {});
element.setAttributes(attributes);
```

**ソース:** `Element.js` — `setAttributes()`

```javascript
setAttributes: function (attributes) {
    this.attributes = attributes;  // ← フィルタリングなし、そのまま代入
    return this;
}
```

`class`を含む全属性がフィルタリングなしでJSモデルに格納される。

#### ⑤ JS → XML レンダリング（保存時）

**ソース:** `extension-tao-itemqti/views/js/qtiXmlRenderer` のテンプレート

```handlebars
<customInteraction {{{join attributes '=' ' ' '"'}}}>
    {{{portableCustomInteraction}}}
</customInteraction>
```

`{{{join attributes ...}}}`は`this.getAttributes()`の全キーバリューペアをXML属性文字列に変換する。`class`も含まれる。

#### ⑥ PHP XML シリアライズ（保存時）

**ソース:** `Element.php` — `xmlizeOptions()`

```php
foreach ($options as $key => $value) {
    if (is_string($value) || is_numeric($value)) {
        $returnValue .= ' ' . $key . '="' . htmlspecialchars($value) . '"';
    }
}
```

**出力結果:** `<customInteraction class="writing-mode-vertical-rl" responseIdentifier="RESPONSE">`

### 実績: audioRecordingInteraction (IMS版)

**ソース:** `extension-tao-itemqti-pci/views/js/pciCreator/ims/audioRecordingInteraction/creator/widget/states/Question.js`

```javascript
interaction.toggleClass('sequential', value);
```

このPCIは実際に`toggleClass()`でクラス属性を操作しており、Save/Loadサイクルを通じて`class`属性が永続化されている。これは**クラス方式がPCIで動作する実証**である。

### まとめ

| 経路 | class属性の扱い | ソース |
|------|----------------|--------|
| XML → PHP パース | `extractAttributes()`で抽出 | `ParserFactory.php` |
| PHP モデル格納 | `Generic`属性として格納 | `Element.php` |
| PHP → JSON | `getAttributeValues()`で出力 | `Element.php` |
| JSON → JS モデル | フィルタリングなしで格納 | `Loader.js`, `Element.js` |
| JS → XML レンダリング | `{{{join attributes}}}`で出力 | テンプレート |
| PHP → XML | `xmlizeOptions()`で出力 | `Element.php` |

**全6段階でclass属性はフィルタリング・除去されることなく通過する。**

---

## 検証結果サマリー

| ユーザーの主張 | 検証結果 | 根拠 |
|--------------|---------|------|
| Widget.jsからクラス方式で保存可能 | **正しい** | `addClass()`→`attr()`→editable mixinが`attributeChange.qti-widget`イベント発火（ソースコード確認済み） |
| Widget.jsからプロパティ方式で保存不可 | **正しい（部分的に）** | `prop()`はイベントを発火しない。手動Save時に保存されるが「即座の保存」ではない |
| Question.jsから両方式で保存可能 | **正しい** | |
| アイテム変更時のクラス方式の自動同期 | **正しい** | |
| アイテム変更時のプロパティ方式の同期不可 | **正しい（さらに深刻）** | `prop('writingMode', undefined)`は削除ではなく**getter**として動作する（後述） |
| 標準インタラクションとの一貫性はクラス方式 | **正しい** | |
| 既存PCIはプロパティを使用 | **ほぼ正しい** | 1つだけ例外あり（audioRecordingInteraction IMS版がclassを使用） |

---

## 1. `prop()` vs `attr()` の保存メカニズム（ソースコード根拠）

### `attr()` の保存経路

**ソース:** `extension-tao-itemqti/views/js/qtiCreator/model/mixin/editable.js`

```javascript
// editable mixin が attr() をオーバーライド
attr: function (key, value) {
    const ret = this._super(key, value);
    if (typeof key !== 'undefined' && typeof value !== 'undefined') {
        $(document).trigger('attributeChange.qti-widget', {
            element: this,
            key: key,
            value: entity.encode(value)
        });
    }
    return _.isString(ret) ? entity.decode(ret) : ret;
},
```

`addClass()` → `attr('class', ...)` → editable mixinのオーバーライド → `attributeChange.qti-widget`イベント発火

### `prop()` の保存経路

**ソース:** `tao-item-runner-qti-fe/src/qtiItem/mixin/CustomElement.js`

```javascript
prop: function (name, value) {
    if (name) {
        if (value !== undefined) {
            this.properties[name] = value;   // ← イベント発火なし、サイレントに書き込み
        } else {
            if (typeof name === 'object') {
                for (var prop in name) {
                    this.prop(prop, name[prop]);
                }
            } else if (typeof name === 'string') {
                if (this.properties[name] === undefined) {
                    return undefined;
                } else {
                    return this.properties[name];  // ← getter として動作
                }
            }
        }
    }
    return this;
},
```

- `prop()`は**いかなるイベントも発火しない**
- editable mixin は `prop()` を**オーバーライドしていない**（`attr`, `removeAttr`, `remove`のみ）
- 保存は手動Save時（`ItemWidget.save()`でモデル全体をXMLシリアライズ）にのみ発生

### 重要: `attributeChange.qti-widget` は自動保存イベントではない

このイベントはウィジェットUI更新のために使用され、ファイルへの永続化をトリガーするものではない。ただし、`attr()`呼び出しでモデルが変更された事実はchangeTrackerに検知され、ユーザーが画面を離れる際に「保存しますか？」ダイアログが表示される。

**結果: どちらの方式も、最終的な永続化はユーザーの手動Save時にのみ行われる。**

しかし、`attr()`経由の変更はイベントを発火するため、他のウィジェットやプラグインがリアクションできる。`prop()`はサイレントなので、他のコンポーネントは変更に気づけない。

---

## 2. 致命的なバグ: `prop('writingMode', undefined)` は削除ではなくgetter

### 問題のコードパス

```javascript
self.element.prop('writingMode', undefined);
```

このコードは一見プロパティを削除するように見えるが、実際には：

```javascript
prop: function (name, value) {
    if (name) {                          // 'writingMode' → truthy → YES
        if (value !== undefined) {        // undefined !== undefined → FALSE
            // ← ここはスキップされる！
        } else {
            // ← こちらに入る
            if (typeof name === 'string') {
                return this.properties[name];  // ← getter として現在の値を返すだけ
            }
        }
    }
}
```

**`value`が`undefined`の場合、`value !== undefined`は`false`になり、setter経路に入らず、getter経路に入る。プロパティは一切変更されない。**

### プロパティを実際に削除するには

```javascript
// 方法1: 直接delete
delete self.element.properties['writingMode'];

// 方法2: removeProp() → ただしバグあり（attributesを削除してしまう）
// self.element.removeProp('writingMode');  // ← 使ってはいけない
```

`removeProp()`のバグ（ソース: `CustomElement.js`）:
```javascript
removeProp: function (propNames) {
    _.forEach(propNames, function (propName) {
        delete _this.attributes[propName];  // ← BUG: properties ではなく attributes を削除
    });
},
```

### 影響

調査ドキュメント `investigation-writing-mode-persistence.md` に記載されている以下のコードは**動作しない**：

```javascript
// Widget.js での想定実装
$itemBody.on('item-writing-mode-changed', function() {
    self.element.prop('writingMode', undefined);  // ← 何もしない（getter）
});
```

プロパティ方式を採用する場合、以下のように書く必要がある：

```javascript
$itemBody.on('item-writing-mode-changed', function() {
    delete self.element.properties['writingMode'];  // ← 直接delete
});
```

---

## 3. Test Runnerの高さ計算への影響

**ユーザーの分析に欠けている重要なポイント。**

### Test Runnerの writing-mode 検出方法

**ソース:** `tao-test-runner-qti-fe/src/helpers/verticalWriting.js`

```javascript
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

**CSSクラスのみで検出する。PCIプロパティは参照しない。**

### 影響

`itemScrolling.js`の`adaptBlockSize()`でブロック要素のwriting-modeを判定する際：

```javascript
const isBlockVerticalWriting = getIsWritingModeVerticalRl($block);
const isDifferentWritingMode = isBlockVerticalWriting !== isItemVerticalWriting;
const sizeProp = isDifferentWritingMode ? normalSizeProp : maxSizeProp;
```

| シナリオ | クラス方式 | プロパティ方式 |
|---------|----------|--------------|
| PCIが縦書き（アイテムは横書き） | `class="writing-mode-vertical-rl"` → 検出される → `height`（正しい） | クラスなし → 検出されない → `max-height`（**誤り**） |
| PCIが横書き（アイテムは縦書き） | `class="writing-mode-horizontal-tb"` → 検出される → `width`（正しい） | クラスなし → 検出されない → `max-width`（**誤り**） |

**プロパティ方式では、アイテムと異なるwriting-modeを持つPCIの高さ計算が誤る。**

---

## 4. 既存PCIの設定保存パターン（調査結果）

**ソース:** `extension-tao-itemqti-pci` リポジトリの実コード

| PCI | 設定の保存方法 | 根拠 |
|-----|--------------|------|
| likertScoreInteraction | **プロパティ** | `interaction.prop('level', value)` |
| likertCompact | **プロパティ** | |
| likertConfig | **プロパティ** | |
| mathEntryInteraction | **プロパティ** | `interaction.prop(name, value)` |
| audioRecordingInteraction (IMS版) | **クラス + プロパティ** | `interaction.toggleClass('sequential', value)` |
| audioRecordingInteraction (dev版) | **プロパティのみ** | |
| liquidsInteraction | **プロパティ** | |

- **ほぼ全てのPCIがプロパティ方式を使用**
- **唯一の例外**: audioRecordingInteraction (IMS版) が `sequential` フラグに `toggleClass()` を使用
- これは writing-mode のような視覚的CSS設定ではなく、動作制御フラグ

---

## 5. `attributeChange.qti-widget`イベントの確認

**ソース:** `extension-tao-itemqti/views/js/qtiCreator/model/mixin/editable.js`

editable mixin がオーバーライドするメソッド：
- `init`
- `attr` ← `attributeChange.qti-widget` を発火
- `removeAttr`
- `remove`

**`prop()` はオーバーライドされていない。** `propertyChange` のようなイベントも存在しない。

**ソース:** イベントリスト（`event.js`）に `propertyChange` は含まれていない：
```javascript
var eventList = [
    'containerBodyChange',
    'containerElementAdded',
    'elementCreated.qti-widget',
    'attributeChange.qti-widget',    // ← attr() 用
    'choiceCreated.qti-widget',
    'correctResponseChange.qti-widget',
    // ... propertyChange は存在しない
];
```

---

## 6. 修正された技術的比較

| 観点 | クラス方式 | プロパティ方式 |
|------|----------|--------------|
| Widget.jsからモデル変更 | 可能（`removeClass()`） | 可能だが`prop(name, undefined)`は**動作しない**。`delete properties[name]`が必要 |
| 変更時のイベント発火 | **あり**（`attributeChange.qti-widget`） | **なし** |
| 変更の永続化タイミング | 手動Save時 | 手動Save時 |
| Question.jsから保存 | 可能 | 可能 |
| アイテム変更時の同期 | `removeClass()`で即座にモデル変更 | `delete properties[name]`で即座にモデル変更（ただしイベントなし） |
| 標準インタラクションとの一貫性 | **同じ仕組み** | 異なる仕組み |
| 既存PCIとの一貫性 | 例外的（audioRecordingInteractionのみ） | **ほぼ全てのPCIがこちら** |
| Test Runnerでのwriting-mode検出 | **自動的に機能** | **機能しない**（追加対応が必要） |
| CSS継承 | **自然に機能** | PCI内部で別途対応が必要 |
| PCI仕様への準拠 | グレー（属性は仕様外の拡張） | **準拠** |
| プロパティ削除のAPI | `removeClass()`が正常動作 | `removeProp()`にバグ、`prop(name, undefined)`も動作しない |

---

## 7. 修正された判断マトリクス

| 優先事項 | 推奨方式 | 理由 |
|---------|---------|------|
| 標準インタラクションと同じ動作 | **クラス方式** | 同じメカニズム |
| 既存PCIとの一貫性 | **プロパティ方式** | ほぼ全てのPCIがプロパティを使用 |
| 実装のシンプルさ | **クラス方式** | CSS継承が自然に機能、APIが正常動作 |
| アイテム変更時の同期 | **クラス方式が優位** | `removeClass()`は正常動作+イベント発火。プロパティ方式は`delete`が必要でイベントなし |
| PCI仕様への準拠 | **プロパティ方式** | |
| Test Runnerの高さ計算 | **クラス方式** | プロパティ方式はTest Runner改修が必要 |
| CSS自然継承 | **クラス方式** | プロパティ方式は別途CSS適用が必要 |
| APIの信頼性 | **クラス方式** | `addClass/removeClass/toggleClass`は正常。`prop(undefined)`はgetter、`removeProp()`はバグあり |

---

## 8. 最終的な見解

### ユーザーの見解「クラス方式が望ましい」は正しい

ソースコード調査により、クラス方式を支持する根拠がさらに強化された：

1. **Test Runnerの高さ計算** — `verticalWriting.js`がCSSクラスのみで検出するため、クラス方式でないとwriting-mode違いの高さ計算が誤る
2. **APIの信頼性** — `addClass()`/`removeClass()`は正常動作するが、`prop(name, undefined)`はgetterとして動作するバグがあり、`removeProp()`にも別のバグがある
3. **イベント連携** — `attr()`経由で`attributeChange.qti-widget`が発火するため、他コンポーネントとの連携が可能
4. **audioRecordingInteraction** — PCIでクラス属性を使用した前例がある

### ユーザーの分析への補足

ユーザーの比較表「アイテム変更時の自動同期: クラス方式=可能、プロパティ方式=不可」は**おおむね正しい**。ただし、正確には：

- クラス方式: `removeClass()`で**モデル変更 + イベント発火** → 正常動作
- プロパティ方式: `delete properties[name]`で**モデル変更のみ（イベントなし）** → 動作はするが、`prop(name, undefined)`と書くと**何も起きない**

### investigation-writing-mode-persistence.md の修正が必要

当ドキュメントに記載されている以下のコードはバグがあるため修正が必要：

```javascript
// ❌ 誤り: getter として動作する（プロパティは削除されない）
self.element.prop('writingMode', undefined);

// ✅ 正しい: プロパティを直接削除
delete self.element.properties['writingMode'];
```
