# LaTeX原稿

- `body.raw.tex`: 記事本文、見出し、図の挿入位置
- `main.tex`: 用紙、余白、フォント、パッケージ、全体設定
- `codexcode.sty`: コード表示用のスタイル
- `media/`: 本文で実際に使用している図・写真

ビルドはこのディレクトリで `latexmk main.tex` を実行してください。完成PDFは `build/main.pdf` に生成されます。
