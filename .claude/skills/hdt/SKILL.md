---
name: hdt
description: "Alias for the herdr-tab skill. Delegate a task to a coding agent running in a separate Herdr tab (not a sibling pane), fire-and-forget. Use when the user asks to hand work off to another tab / 別タブ, e.g. 「別タブで〜させて」「別タブのエージェントに〜を頼む」, or invokes /hdt. Creates a new tab, starts the agent, sends one self-contained prompt, and returns the tab ID and agent name without waiting. Requires HERDR_ENV=1."
---

# hdt — `herdr-tab` のエイリアス

このスキルは `herdr-tab` の短縮呼び出し口でしかない。手順は本体にだけ書かれている。

**いま最初にやること**: Read ツールで次のファイルを読み、その内容を手順書として**そのまま**実行する。

```
/Users/koji.kobayashi.wsu/.claude/skills/herdr-tab/SKILL.md
```

- 読んだ内容が指示の本体。このファイルに手順を要約して持たせていないのは意図的で、
  本体を更新したときにこちらが古くなるのを防ぐため。**推測で代替しない。必ず読む。**
- `/hdt` に渡された ARGUMENTS は、`/herdr-tab` に渡されたものとして扱う。
- 本体が読めなかった場合（移動・削除された等）は、勝手に手順を再構成せず、
  ファイルが見つからないことをユーザーに伝えて止まる。
