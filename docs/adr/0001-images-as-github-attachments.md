---
status: accepted
date: 2026-11-27
---

# 圖片直接上傳成 GitHub 附件，不做圖床

初版規格假設 GitHub 沒有上傳圖片的途徑，所以設計了可抽換的圖床 driver（local、S3、imgur、通用 HTTP），圖片必須是公開網址。這個假設已經不成立：gh v2.99.0（2026-09-01）加入 `--attach`，背後的端點 `POST https://uploads.github.com/user-attachments/assets` 接受 PAT。我們決定圖片只走這一條路，把整個圖床 driver 設計拿掉，因為這樣不必架圖床、不必對外公開檔案、少一套金鑰，擷圖也跟著 repo 的可見度走。

## Considered Options

- **可抽換的圖床 driver，預設 local**（初版設計）：圖片要讓 GitHub 的 camo 代理抓得到，所以一定是公開網址，安全性只靠網址難猜；還要多維護檔案服務和圖床金鑰。否決。
- **GitHub 附件作為其中一個 driver，預設仍是 local**：保留了抽象層，卻沒有第二種真實需求。否決，等有需求再加。
- **把圖 commit 進 repo**：和程式碼耦合。初版就已否決。
- **relay 呼叫 `gh` 執行檔**：Docker image 和 PHP 宿主機都得裝 `gh`。否決，改為 Rust 和 PHP 各自直接打 HTTP 端點。

## Consequences

- **依賴未文件化的端點**：這個端點不在 GitHub 官方 REST API 文件中，以 cli/cli 的 `internal/attachments/client.go` 為參考實作。GitHub 改動時我們要跟著改，所以保留一個對真實測試 repo 的上傳檢查。
- **只能用 PAT，放棄 GitHub App**：端點對 GitHub App installation token 回 404，維護者表示目前是刻意如此（cli/cli#14309）。改用專用機器帳號的 fine-grained PAT，限定單一 repo。
- **圖片無法自行刪除**：沒有刪除 API，網站上也沒有刪除按鈕。誤傳時只能編輯 issue 移除引用，再向 GitHub 客服申請刪除。原本的 `relay-server delete` 指令和 `image_delete_ref` 欄位因此移除。
- **不支援 GitHub Enterprise Server**：端點在 GHES 上不存在。
- **圖片位元組仍要暫存**：為了能重試，開單成功前圖片以備份檔留在不對外的目錄，成功後立刻刪除，失敗的最長保留 7 天。這不是圖床，不對外提供。

## 重新評估的條件

- GitHub 讓 installation token 可以上傳附件：改回以 GitHub App 為目標。
- GitHub 提供刪除附件的 API：恢復刪圖指令。
- 端點被移除或改成不接受 PAT：回頭考慮圖床 driver。
