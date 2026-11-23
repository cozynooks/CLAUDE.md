---
name: herdr-tab
description: "Delegate a task to a coding agent running in a separate Herdr tab (not a sibling pane), fire-and-forget. Use when the user asks to hand work off to another tab / 別タブ, e.g. 「別タブで〜させて」「別タブのエージェントに〜を頼む」. Creates a new tab, starts the agent, sends one self-contained prompt, and returns the tab ID and agent name without waiting. Requires HERDR_ENV=1."
---

# 別タブのエージェントに指示する

Herdr の**別タブ**に新しいエージェントを立て、自己完結した指示を 1 通投げて、待たずに制御を戻す。既存の `herdr` スキル（同じタブの隣ペインが既定）とは意図的に使い分ける。ペインではなくタブなので、相手は幅いっぱいの端末を持ち、こちらのレイアウトを狭めない。

## 0. 前提を確認する

```bash
test "${HERDR_ENV:-}" = 1
```

失敗したら「Herdr の中で動いていないので別タブに指示できない」と伝えて止まる。外から他人のセッションを操作しない。

コマンド構文の正解は常にインストール済みバイナリ。迷ったら `herdr tab` / `herdr agent` を引数なしで叩いて確認する（`herdr tab create` のような変更系をあえて引数なしで叩くのは禁止。既定値で実行されてしまう）。ID は必ず JSON レスポンスから読む。

## 1. 指示文を先に固める

タブを作る前に、投げる指示文を完成させる。相手はこちらの会話履歴を一切持たないので、**指示文だけで作業が完結**しなければならない。

- 作業ディレクトリと対象ファイル（絶対パス）
- ゴールと完了条件（どこまでやったら終わりか）
- 制約（触ってほしくない場所、既存の慣習、実行してよいコマンド）
- 成果物の形（コードを直すのか、調査して報告するのか）

非同期で投げっぱなしにするので、こちらへの質問を前提にした指示は書かない。判断は相手に委ねる。自分の見立て・原因の候補・優先順位・報告の章立て・「最重要」「必ず」などの重み付けは渡さない。相手は同じ能力でリポジトリも GCP も読めるので、自力で辿り着けることは書かない。事前に調べたことは、そのタブからは辿れない事実だけ渡す。指示文が長いと感じたら、足すのではなく削る（制約と環境固有の落とし穴は削らない）。

調査系の結論はチャットで報告させ、Markdown ファイルとして残させない（ユーザーはそのファイルを読まない）。ユーザーが明示的にドキュメント化を頼んだときだけ例外。

ユーザーの依頼が曖昧で、解釈違いで作業が丸ごと無駄になりそうなときは、**投げる前に**ユーザーに確認する。投げた後では止められない。

## 2. タブを作る

```bash
herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" --label <短いラベル> --no-focus
```

- `--cwd "$PWD"` を必ず付ける。付けないと相手が別のディレクトリで動く。
- `--no-focus` を必ず付ける。ユーザーのフォーカスを勝手に奪わない（ユーザーが「切り替えて」と言った場合のみ `--focus`）。
- `--label` は作業内容が分かる短い語にする（`review`, `migrate-gift` など）。
- レスポンスの `.result.tab.tab_id` と `.result.root_pane.pane_id` を読む。予測しない。

続けて、root pane の右に空のペインを作る。ユーザーがそのタブで手作業するためのシェル。

```bash
herdr pane split <root_pane_id> --direction right --cwd "$PWD" --no-focus
```

- エージェントは root pane（左）で起動する。右ペインには何も走らせない。
- 右ペインの pane ID もレスポンスから読み、引き継ぎ情報に含める。

## 3. エージェントを起動する

既定は `claude`。ユーザーが別の種類を指定したらそれに従う（`herdr agent` で対応 kind 一覧を確認）。

```bash
herdr agent start <name> --kind claude --pane <root_pane_id>
```

- `<name>` は `[a-z][a-z0-9_-]{0,31}` かつ生存中のエージェント間で一意。先に `herdr agent list` で衝突を確認する。役割が分かる名前にする（`reviewer`, `gift-migrator`）。
- `agent start` はエージェントが対話入力を受け付けられる状態になってから返る（既定 30 秒でタイムアウト）。`agent_not_ready` が返った場合は名前だけ生きているので、`herdr agent get <name>` で状態を見て、`idle` になるまで待ってからプロンプトを送る。
- 権限モードなどネイティブ引数は `--` の後に渡す。ユーザーが明示的に求めない限り権限を緩めない。

## 4. 指示を投げて、待たない

```bash
herdr agent prompt <name> "<指示文>"
```

`--wait` は付けない。これはこのスキルの要点で、投げたら即座にユーザーに制御を返す。

ただし `agent prompt` は、入力欄に貼るだけで Enter が入らず未送信で終わることがある。送った後に `herdr agent get <name>` で `working` になったことを確かめてから報告する。`idle` のままなら `agent read` で入力欄に本文が残っていないか見て、`herdr agent send-keys <name> enter` で確定する。

長文や引用符が入り組んだ指示文はシェルクォートで壊れやすい。`$TMPDIR` にファイルを書いてから渡す:

```bash
herdr agent prompt <name> "$(cat "$TMPDIR/prompt.md")"
```

`agent prompt` は相手が承認ダイアログや質問待ちだと、何も送らずに `agent_blocked` を返す。起動直後にこれが出た場合は `herdr agent read <name> --source recent-unwrapped` で画面を見て、勝手に答えず**ユーザーに判断を仰ぐ**。

`agent_prompt_stalled` は、送ったのに 5 秒以内に状態変化が観測できなかったという意味。`agent get` / `agent read` で本当に届いていないか確認してから再送する。

## 5. ユーザーに引き継ぎ情報を返す

投げた直後の報告には必ず次を含める。これがないとユーザーは追跡できない。

- タブ ID とラベル、エージェント名、kind
- 右ペインの pane ID
- 投げた指示の要点（1〜2 文）
- 後から見る方法:

```bash
herdr agent get <name>                                      # idle / working / blocked / done
herdr agent read <name> --source recent-unwrapped --lines 120  # 出力を読む
herdr tab focus <tab_id>                                    # 自分で見に行く
```

状態の読み方: `working` は作業中。`idle` / `done` は入力待ちに落ち着いた状態（`done` は未確認のまま背景作業が終わったもの）。`blocked` は承認・質問ダイアログで止まっている＝ユーザーの操作が必要。`unknown` は分類できないだけで完了の証拠にはならない。

`agent read` で応答が途切れて見える場合、相手が代替スクリーンで動いていて行数を増やしても遡れないことがある。そのときだけ「回答全文を一時ディレクトリに Markdown で書いてパスだけ返して」と追加で頼み、ファイルを直接読む。最初のプロンプトでファイル出力を要求はしない。読んで内容を relay したら、そのファイルは消す。

## ルール

- 1 タスク 1 タブ。関係ない作業を同じエージェントに追い刺ししない。
- ユーザーが「別タブ」と言っていない、単に並行作業が便利そう、という理由でこのスキルを使わない。隣ペインで足りるなら `herdr` スキルの `pane split` を使う。
- 自分が作っていないタブ・ペイン・エージェントを閉じない。自分が作ったタブも、作業が終わったからと勝手に閉じない（ユーザーが結果を見る前に消える）。片付けはユーザーの指示があってから `herdr tab close <tab_id>`。
- ワークスペースやワークツリーは、ユーザーが明示的に求めない限り作らない。
- 同じリポジトリで書き込み作業を並行させると衝突する。編集を伴う指示を複数タブに投げるときは、担当ファイルを分けるか、ワークツリー分離をユーザーに提案する。
