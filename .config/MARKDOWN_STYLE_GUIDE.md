# Markdown Style Guide for LINE Business Developer Documentation

このプロジェクトでは、以下のMarkdownスタイルルールに従っています。すべてのMarkdownファイルはこれらのガイドラインに準拠する必要があります。

## Lintルール

このプロジェクトでは以下の設定でmarkdownlintを使用しています：

```json
{
  "default": true,
  "MD013": false,
  "MD024": { "siblings_only": true },
  "MD029": { "style": "ordered" },
  "MD033": false,
  "MD036": false,
  "MD040": false,
  "MD041": false,
  "MD045": false
}
```

### 主要ルールの説明

- **デフォルトルール**: 基本的にはすべてのルールが有効です
- **MD013 (false)**: 行の長さ制限を無効化しています
- **MD024 (siblings_only)**: 同じ階層レベルでのみ見出しの重複をチェックします
- **MD029 (ordered)**: 順序付きリストは連番で記述します（1, 2, 3...）
- **MD033 (false)**: HTML記法の使用を許可しています
- **MD036 (false)**: 強調（\*\* \*\*や\_ \_）を見出し代わりに使用することを許可しています
- **MD040 (false)**: コードブロック(```)に言語指定がなくても許可します
- **MD041 (false)**: 文書の最初の見出しレベルの強制を無効化しています
- **MD045 (false)**: 画像にalt属性（代替テキスト）がなくても許可します

## フォーマットルール

このプロジェクトでは以下の設定でPrettierを使用しています：

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "proseWrap": "preserve",
  "endOfLine": "lf"
}
```

## エディタ設定

基本的なフォーマットはEditorConfigで定義されています：

```
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false
```

## Markdownファイルのフォーマット方法

- すべてのMarkdownファイルをフォーマットするには：`npm run format:md`
- Lintの問題をチェックするには：`npm run lint:md`
- Lintの問題を自動修正するには：`npm run fix:md`
- すべての整形と修正を一括実行するには：`npm run fix:all`

## Git連携

コミット前に自動的にフォーマットとLintチェックが実行されます。これにより、すべてのMarkdownファイルが一貫したスタイルを維持できます。
