# systemd / boot-autostart skills

兩個可重用的 agent skill，處理 Linux systemd 服務與開機自動啟動。從本機 skill library 抽出成獨立 repo，便於版控與分享。

## Skills

| Skill | 用途 |
|---|---|
| `setting-up-boot-autostart/` | 自託管服務在重開機後自動啟動：判斷是否需要 supervisor、避免重複實例、開機後驗證。 |
| `systemd-unit-authoring/` | 撰寫、審查、除錯 systemd `.service` unit：system vs user scope、`Type=`、排序/`Requires`、`/run` runtime dir、secret 外部化、stamp handshake、開機/login 驗證。 |

每個目錄的 `SKILL.md` 即完整 skill（frontmatter + 指令），可直接複製到 agent 的 skills 目錄使用。

## 安裝

```sh
cp -r setting-up-boot-autostart systemd-unit-authoring ~/.agents/skills/
```

## 由來

這兩個 skill 以 RED → GREEN → REFACTOR（TDD for skills）流程產出。`systemd-unit-authoring` 的 Condition/Assert 與 `Restart=` 行為經 systemd 官方 man page 查證，避免把常見誤解寫進 skill。

## 版控說明

- 排除帳本：`_repo_excluded.tsv`（本 repo 無排除項目 — 所有檔案皆為可分享的純文字 skill）。
- 無 secret、無大檔、無巢狀 repo（經 `push-repo` preflight 驗證）。
