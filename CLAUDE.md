# n-pendulum

n重振り子のシミュレーター。計算はPython、画面はTypeScript。
画面が条件を送り、計算サーバーが振り子の動きを計算して返し、画面がそれを描く。
課題の全体像は docs/n-pendulum-guide.md にある。

## 構成

- backend/：Python 3.12以上、FastAPI。パッケージ管理は uv
  - backend/physics/：物理計算（NumPyのみ使用）。運動方程式とRK4は別のファイルに分け、運動方程式を差し替えられるようにする
- frontend/：Vite + TypeScript。描画はCanvas
- docs/：設計書（design.md）、計画書（plan.md）、ふりかえり（retrospective.md）、課題ガイド
- .claude/skills/：よく頼む作業の手順書
  - step-report：ステップ報告を書く手順

## ルール

- 物理計算を変えたら、必ず pytest を実行する
- 数式を変えるときは、docs/design.md の式と合っているか確かめる
- RK4は自分で実装する（SciPyなどの積分器は使わない）
- nは1〜8、1回の計算の秒数には上限をつける
- 1回の変更は小さくする。終わったら変更点を3行で説明する
- 分からないことは推測で進めず、質問する
- APIキーやパスワードを、コードやGitHubに書かない
- 資料はすべてマークダウンで、日本語で書く

## よく使うコマンド

- テスト：cd backend && uv run pytest
- APIの起動：cd backend && uv run fastapi dev
- 画面の起動：cd frontend && npm run dev
