# Proxmoxやりましょう

年刊EDP 2026向けに執筆した、Proxmox VEの導入と自宅サーバー構築の記事です。

## 公開内容

- [`article.pdf`](article.pdf): 完成した記事
- [`source/`](source/): LaTeX原稿と本文で使用している図・写真
- [`supplements/iommu-notes.md`](supplements/iommu-notes.md): IOMMUの確認・設定メモ
- [`supplements/proxmox-mcp-plus-setup.md`](supplements/proxmox-mcp-plus-setup.md): ProxmoxMCP-Plusの完全設定メモ

このリポジトリには、記事の作成と補足資料の閲覧に必要なファイルだけを収録しています。自宅環境のバックアップ、運用記録、自動設定ツール、未使用画像は含めていません。

## ビルド

macOSのTeX Live環境で、次のコマンドを実行します。

```bash
cd source
latexmk main.tex
```

生成物は `source/build/main.pdf` です。原稿ではMS明朝、MSゴシック、Times New Roman、Arial、Courier Newを使用しています。

## ライセンス

本リポジトリには現時点で利用許諾ライセンスを設定していません。
