# トラブルシューティング記録

このドキュメントは、検証中に発生した2つの問題（ASA-04のTCP到達性問題、ASA1コンソール異常）について、
実際に確認した事実と、そこから導いた判断を分けて記録したものです。推測部分は「推測」であることを明記しています。

---

## 1. ASA-04: R-OUTのTCP/80(HTTP)への接続が拒否される問題

### 事象（事実）

R-IN上で以下を実行したところ、TCPコネクションが即座に拒否された。

```
R-IN# telnet 203.0.113.1 80
Trying 203.0.113.1, 80 ...
% Connection refused by remote host
```

このときASA1の`show conn`は終始「0 in use」で、コネクションテーブルに何も登録されなかった。

### 切り分け手順（実施した事実）

1. **R-OUT自身から自分自身の203.0.113.1:80へ接続を試行** → 同様に即座に`Connection refused`。
   ASAを経由しないローカル接続でも失敗したため、ASAのACL/セキュリティレベルの問題ではないと判断。
2. **`show ip http server status`をR-OUTで確認** → `HTTP server status: Enabled`、`HTTP server port: 80`と表示され、
   設定上は有効になっていた。
3. **ポート23（Telnet、vty）への接続も同様に試行** → これも即座に`Connection refused`。
   HTTP固有ではなく、コントロールプレーンのTCPサービス全般が応答しないことを確認。

### 判断（切り分け結果に基づく結論）

設定（`ip http server`等）は正しく投入されており、ASA側のACL/セキュリティレベルにも問題がないことを確認した上で、
それでも接続が確立しないことから、**このCML環境の`iol-xe`（IOL XE 17.18.02、Dockerコンテナ型）イメージにおいて、
データプレーンのEthernetインターフェース経由でのコントロールプレーンTCPサービス（HTTP/Telnet）へのインバウンド接続が
確立しない**という、このイメージ固有の制約が原因と判断した。

この時点でのroot causeの断定（例: コンテナのネットワーク名前空間分離、iptables、特定サービスのバインド先制限など）は
行っていない。上記は観測事実からの推論であり、CMLやIOL-XEコンテナの内部実装を直接確認したものではない。

### 解決策（実施内容）

`service tcp-small-servers`（Echo/Discard/Chargen等のレガシーTCPサービス群、ポート7のEcho含む）をR-IN・R-OUTの両方に
追加した。

```
configure terminal
service tcp-small-servers
end
```

この設定はASAを経由したTCP/7への接続で**正常に動作**（`Open`後、送信データがエコーバック）することを確認した。
これにより、同一データプレーンパス上で、ASAのステートフル・インスペクション機能自体は問題なく検証できた
（結果は[test-status.md](test-status.md)のASA-04を参照）。

この設定は最小限・可逆であり、`no service tcp-small-servers`で削除可能。CML上のNode Definitionの`SECURITY WARNING`
（tcp-small-serversは不要なサービスを公開するため非推奨、というCML側の警告）が投入時に表示されたが、
このラボは隔離された検証環境であり、本番運用を意図したものではないため、検証目的での一時利用として許容している。

---

## 2. ASA1コンソール異常

### 事象（事実のみ）

証跡の再取得作業中、ASA1のコンソール（line 0）に新規WebSocket接続を行ったところ、通常のASA CLIプロンプトや
コマンド応答ではなく、以下のような内容が継続的に出力される状態を複数回にわたり観測した。

```
7ff05e6f0000-7ff05e6f1000 r--p 00000000 00:02 15096                      /usr/lib64/dpdk/pmds-23.0/librte_mempool_bucket.so.23.0
7ff05e6f1000-7ff05e6f5000 r-xp 00001000 00:02 15096                      /usr/lib64/dpdk/pmds-23.0/librte_mempool_bucket.so.23.0
...
```

これはLinuxの`/proc/<pid>/maps`形式に類似した、プロセスのメモリマッピング情報で、
`/usr/lib64/dpdk/pmds-23.0/librte_*.so`（DPDKのPMDライブラリ群）へのパスを含んでいた。

### 確認した事実

- 新規に開いた複数回のWebSocket接続すべてで同様の内容が観測された。
- 空Enter（`\r`のみ送信）を送っても、通常のプロンプト（`ASA1#`等）には遷移せず、同種の内容が続けて出力された。
- 無操作のまま受動的に受信を続けた場合、出力が0バイトの期間と、再び出力が流れる期間の両方を観測した
  （完全に停止しているわけではないが、常時大量出力し続けているわけでもない、という挙動）。
- この間、**R-INおよびR-OUTのコンソールはいずれも正常**（通常のプロンプト・コマンド応答）だった。問題はASA1のみ。
- CML REST API (`GET /api/v0/labs/{lab}/nodes/{node}/state`) は、この間も一貫して`"state": "BOOTED"`を返し続けた。
  オーケストレーション層はこの異常を検知していない。
- こちらから投入したコマンドは、通常の`show`系コマンド・`packet-tracer`・`configure terminal`によるpolicy-map設定
  ・`enable`のみであり、このような出力を意図的に引き起こすコマンドは一切実行していない。
- 後日（本ドキュメント作成の直前）改めて接続を確認したところ、空Enter送信後に`ASA1> `という通常のプロンプト
  （ただし特権EXECではなく一般EXECモード）が返るようになっていることを確認した。ただし、ユーザーの指示により
  これ以上の追加調査（`reload`や特権モードへの昇格を含む）は行っていない。

### 行っていないこと

- `reload`（再起動）は一切実行していない。
- 上記異常の原因を特定するための追加のデバッグコマンド（`show tech-support`、`show crashinfo`等）も実行していない。
- ASA1の完全な`show running-config`を、この異常発生後に強引に取得しようとする試みは行っていない。

### 影響範囲

この異常が発生する**前**に取得済みだった以下のデータは、正常なASA CLI応答として取得されたものであり、信頼できる
データとして本リポジトリに収録している。

- `show interface ip brief` / `show nameif` / `show running-config interface`（ASA-01、[outputs/ASA-01-interface.txt](../outputs/ASA-01-interface.txt)）
- `packet-tracer` 各種（ASA-02、ASA-04、対応する`outputs/`配下のファイル）
- `show route` / `ping`（ASA-03）
- `show running-config policy-map` / `show service-policy`（ASA-05）

一方、単一の統合された`show running-config`（ASA1のホスト名設定・enableパスワード行・route文等を含む完全版）の
再取得はこの異常のため中断しており、[configs/ASA1.cfg](../configs/ASA1.cfg) には異常発生前に取得できた
インターフェース設定断片とpolicy-map設定断片のみを収録している。それ以外の部分（`route outside`文など）は
本セッション中の別の時点で画面表示を確認してはいるが、単一の`show running-config`として再取得・再確認できて
いないため、本ファイルには含めていない（推測や記憶による再構成はしていない）。

### 原因についての所見（推測であることを明記）

DPDKライブラリのメモリマップという内容の性質上、ASAvのデータプレーン（Lina）プロセスに関連する
クラッシュダンプやデバッガ出力が、CMLのシリアルコンソールに漏れて表示された可能性が考えられる。
ただし、これはログの見た目から推測される仮説であり、CMLやASAv内部のプロセス状態を直接確認したものではない。
断定はできない。
