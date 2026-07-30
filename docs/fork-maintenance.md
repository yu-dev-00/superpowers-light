# フォーク運用方針(superpowers-light)

このリポジトリは [obra/superpowers](https://github.com/obra/superpowers) のフォークで、自前の改良を加えた
カスタマイズ版プラグインを自前マーケットプレイスとして配布するために運用する。

## ブランチ構成

| ブランチ | 役割 |
|---|---|
| `main` | upstream/main の純粋なミラー。**自分のコミットは一切置かない**(fast-forward のみ) |
| `dev` | 改良ブランチ。全カスタマイズ+マーケットプレイス定義をここに置く。GitHub のデフォルトブランチ |

注意: 本家にも `upstream/dev`(本家の開発ブランチ)が存在するが、ローカル/origin の `dev` とは別物。

## リモート構成

- `origin` = `yu-dev-00/superpowers-light`(自分のフォーク)
- `upstream` = `obra/superpowers`(fetch 専用。push 禁止設定にする)

```bash
git remote add upstream https://github.com/obra/superpowers.git
git remote set-url --push upstream DISABLED
```

## プラグイン名の方針

- `plugin.json` の `name: "superpowers"` は**変更しない**。
  スキル本文に `superpowers:brainstorming` などプラグイン名を含む内部参照が 25 箇所以上あり、
  改名するとその書き換えが upstream 追従時の恒常的なコンフリクト源になるため。
- 区別はマーケットプレイス名で行う: インストール表記は `superpowers@superpowers-light`。
- 利用時は公式版 superpowers プラグインを disable(off)にして衝突を避ける。

## マーケットプレイス定義

`.claude-plugin/marketplace.json`(`dev` ブランチのみに存在):

- マーケットプレイス名: `superpowers-light`
- プラグインエントリ 1 件、`source: "./"` で自リポジトリを指す

## upstream 追従手順(定期・手動)

```bash
git fetch upstream
git checkout main
git merge --ff-only upstream/main
git push origin main
git checkout dev
git merge main          # コンフリクトはここで解決
git push origin dev
```

- コンフリクトが発生するのは自分が改変したスキルファイルのみ。
- `plugin.json` の `version` 等のメタデータは基本 upstream 側を採用する。

## 利用側(Claude Code)のセットアップ

1. 公式版 superpowers プラグインを disable(アンインストールではなく off)
2. `/plugin marketplace add yu-dev-00/superpowers-light`
3. `/plugin install superpowers@superpowers-light`

以後の更新は `dev` に push → プラグイン側で update すると反映される。

## スコープ外(必要になったら検討)

- スキル本文の実際の改良(この基盤の上で別作業として実施)
- 追従の自動化(CI での自動 sync)
