# 可重用 agent skills（systemd / boot-autostart / push-repo）

三個可重用的 agent skill，處理 Linux systemd 服務、開機自動啟動，以及安全的 git repo 推送。從本機 skill library 抽出成獨立 repo，便於版控與分享。

## Skills

| Skill | 用途 |
|---|---|
| `setting-up-boot-autostart/` | 自託管服務在重開機後自動啟動：判斷是否需要 supervisor、避免重複實例、開機後驗證。 |
| `systemd-unit-authoring/` | 撰寫、審查、除錯 systemd `.service` unit：system vs user scope、`Type=`、排序/`Requires`、`/run` runtime dir、secret 外部化、stamp handshake、開機/login 驗證。 |
| `push-repo/` | 把目錄獨立成 git repo 並推到遠端：preflight 排除帳本、巢狀 repo／大檔／機密處置、GIT_ASKPASS 憑證安全、404/403 診斷、fresh-clone 逐檔驗證。 |

每個目錄的 `SKILL.md` 即完整 skill（frontmatter + 指令），可直接複製到 agent 的 skills 目錄使用。`push-repo/` 另含 `references/` 與 `scripts/`。

## 安裝

```sh
cp -r setting-up-boot-autostart systemd-unit-authoring push-repo ~/.agents/skills/
```

## 由來

這三個 skill 以 RED → GREEN → REFACTOR（TDD for skills）流程產出：

- `systemd-unit-authoring` 的 Condition/Assert 與 `Restart=` 行為經 systemd 官方 man page 查證，避免把常見誤解寫進 skill。
- `push-repo` 由 `git-push` 與原 `push-repo` 合併而成，消除「token 寫進 remote URL」與「憑證不可進 URL」的指令衝突；憑證統一為 `gh auth setup-git` 或 `GIT_ASKPASS`。合併過程實測發現 `GIT_ASKPASS=<(printf ...)` 的 process substitution 即使 bash 也失敗（pipe 不可 exec），已改為 `mktemp` 暫存 script。

## 版控說明

- 排除帳本：`_repo_excluded.tsv`（本 repo 無排除項目 — 所有檔案皆為可分享的純文字 skill）。
- 無 secret、無大檔、無巢狀 repo（經 `push-repo` preflight 驗證）。
