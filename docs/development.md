# 開發指南

本文件從舊版 README 移過來，內容是本機開發、改動流程、測試與行動版 app 的建置方式。專案介紹見 [README](../README.md)。

## 技術架構

- `backend/connect4_app/`：FastAPI、原生 WebSocket、匿名 Cookie 工作階段與房間狀態。
- `native_solver/`：PyO3 擴充，封裝 `connect-four-ai` 1.0.0 精確求解器。
- `frontend/`：Vue 3、TypeScript、Pinia、Vue I18n 與響應式棋盤；介面規格見 `design/spec.md`。
- `Containerfile`、`deploy/`：Podman 正式映像、環境設定範例與 systemd user service。

AI 會算出每個合法落子的精確終局分數，再選擇最高分手；同分時固定採中央優先。沒有深度限制、隨機弱化或啟發式備援。原生引擎若故障，該局會停止並回報錯誤。

## 本機開發

需要 Python 3.10+、stable Rust、Node.js 22+：

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -e '.[dev]'
npm --prefix frontend install
```

分別啟動 API 與 Vite：

```bash
.venv/bin/uvicorn connect4_app.app:app --host 127.0.0.1 --port 55555 --reload
npm --prefix frontend run dev
```

開啟 `http://127.0.0.1:5173`。正式執行時先跑 `npm --prefix frontend run build`，再以單一 Uvicorn worker 啟動；房間狀態目前存於記憶體，不可使用多 worker。

### 改動流程

`main` 受保護，不接受直接 push。每個改動走分支與 pull request，CI 的 `backend`、
`frontend`、`container` 三個檢查都綠才能合併（squash）。多個 session 同時開發時，各自用
獨立的 worktree，不要在共用的 checkout 切分支：

```bash
scripts/dev-worktree.sh front feat/win-animation   # 建 ../connect4-web2-worktrees/front
```

Dependabot 每週檢查 npm、pip、cargo 與 GitHub Actions 的更新；小版本與修補版在 CI 綠燈後
自動合併，大版本等人審。

## 驗證

```bash
.venv/bin/pytest
.venv/bin/ruff check backend tests
npm --prefix frontend test
npm --prefix frontend run build
```

手機矩陣測試涵蓋 390×844 至 440×956 的 iPhone 14 Pro Max～17 系列，以及 Galaxy S26 Ultra 直向／橫向：

```bash
cd frontend
npx playwright install --with-deps chromium webkit
npm run test:e2e
```

測試會檢查水平溢位、44px 觸控目標、鍵盤高度與視覺基準。

視覺基準圖由 CI 的 Ubuntu 22.04 產生，字型與瀏覽器和 CI 的 `frontend` 檢查相同。改了畫面之後，到 GitHub Actions
手動執行 `Update screenshots` 並填入分支名稱（可選填 `grep` 只跑部分測試）。它只重寫對不上或缺少的基準圖，
commit 回那個分支，再在分支上啟動 CI。

## 行動版 app

iOS／Android app「四子棋」（英文 Four In A Row，bundle ID `com.oraclelee.connect4`）放在
`mobile/`：Capacitor 把 `frontend/` 的建置結果打包進 app，app 再以 session token 跨網域連
`https://connect4.oraclelee.com`（協定見 `docs/protocol.md` 的「App 連線」）。改網頁後要發新版
app 才會帶上，網站本身的發版流程不變。

- 改到 app 會打包的檔案時，pull request 會跑 `Mobile` workflow：建置 Android debug APK
  （artifact `connect4-debug-apk`，可直接側載）與 iOS 模擬器版。它不是必要檢查。
- 發布：網頁版本上線後，從 main 手動執行 Actions 的 `Mobile release` 並填入該版本的 `v*` tag，
  產生簽章的 Android AAB 並把 iOS 版上傳到 TestFlight；缺哪個平台的簽章 secrets 就跳過哪個，並寫出原因。
- Android 上傳金鑰只存在 GitHub secrets 與 `~/.config/connect4-mobile/`，務必另外備份到密碼管理器。
  iOS 簽章要等有 Apple Developer 會員後才啟用。
- 隱私權政策：[PRIVACY.md](../PRIVACY.md)。

建置、簽章、iOS 啟用步驟與上架準備見 `docs/mobile-release.md` 和 `docs/mobile-store-checklist.md`。

