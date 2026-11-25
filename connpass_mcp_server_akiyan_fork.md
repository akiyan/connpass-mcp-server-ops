# connpass-mcp-server akiyan fork の変更情報

直上流は [kajidog/connpass-mcp-server](https://github.com/kajidog/connpass-mcp-server)。初回の基準は上流 `main` の [`3bad12a4`](https://github.com/kajidog/connpass-mcp-server/commit/3bad12a4fad04789b6d5832c201daf2d6784b379)。[`akiyan` ブランチ](https://github.com/akiyan/connpass-mcp-server/tree/akiyan) は基準コミットの上に独自コミットを常に一つだけ置き、コミット件名はこのページの URL にします。fork の `main` は上流同期用で、独自変更を入れません。

## 独自の変更

- 固定トークン認証付きの Cloudflare Workers エンドポイントを追加。Workers では MCP の通信セッションを作らず、MCP Apps の HTML をバンドルします。
- ChatGPT の OAuth に対応。動的クライアント登録は ChatGPT のコールバックのみに限定し、PKCE と所有者キーによる承認を要求します。既存の固定トークン認証も維持します。ローカルで承認からトークン更新まで、本番で OAuth 発見情報と未認証リクエストの拒否を確認しました。今後の本番更新では OAuth KV を含むローカル設定を使います。
- 検索結果は Durable Objects に保存し、`search_events` と `browse_events` が別リクエストでもつながります。保存から 10 分で取得期限切れとなり、アラームが削除を行います。
- Workers から connpass API へは fetch を使用し、ファイルを使う発表情報キャッシュは無効です。Node.js の stdio / HTTP は存続します。
- MCP SDK の要求する戻り値型に合わせ、ツールのエラー応答型を修正しました。

[Workers のデプロイ・検証手順](https://github.com/akiyan/connpass-mcp-server/blob/akiyan/docs/cloudflare-workers.md)を参照してください。通常のビルド、Wrangler の dry run、ローカルの認証・MCP Apps、模擬 connpass API を使った検索の順に確認します。実アカウントへのデプロイ時には `/health` に加えて、MCP クライアントから `search_events` を呼び出し、実 API キーも確認してください。

## パッチを保守する

ソース側で、通常のバグ修正や機能追加はテスト後に `git commit --amend --no-edit` で独自コミットへまとめます。変更理由と検証記録はこの ops 側へ別コミットで記録します。一時的な作業は別ブランチに置き、公開する内容は一つのコミットにまとめてください。

上流追従時は作業ツリーをクリーンにし、先にリモートの `akiyan` を確認します。次にローカルのパッチを新しい上流の上に載せ直します。

```bash
git switch akiyan
git fetch upstream
git fetch origin
old_base=$(git rev-parse HEAD^)
git rebase --onto upstream/main "$old_base" akiyan

test "$(git rev-parse HEAD^)" = "$(git rev-parse upstream/main)"
test "$(git rev-list --count upstream/main..HEAD)" = 1
```

競合を解消して再検証し、基準コミットをこのページに記録します。既に公開済みのパッチを更新するときは、作業開始時に確認したリモート SHA を lease に指定して上書きします（未取得の変更がある場合は失敗します）。初回は通常の push です。

```bash
# 初回のみ
git push -u origin akiyan
# 以降は <確認済みのリモート SHA> を実値に置き換える
git push --force-with-lease=refs/heads/akiyan:<確認済みのリモートSHA> origin akiyan
```
