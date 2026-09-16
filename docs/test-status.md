# 試験結果一覧 (test-status)

検証環境: CML 2.10.0-13 / ASAv 9.24.1 (`asav-9-24-1`) / IOL XE 17.18.02 (`iol-xe-17-18-02`)
ラボID: `abb2d2c6-08a6-4935-93d9-08a3734b180d`（`asa-basic-verification`）

| ID | 検証項目 | 期待結果 | 実測結果 | 判定 |
|----|---|---|---|---|
| ASA-01 | インターフェース（IPアドレス・セキュリティレベル・状態） | outside: 203.0.113.2/24 sec0, inside: 10.10.10.1/24 sec100、両方up/up | `show interface ip brief` / `show nameif` / `show running-config interface` で全項目一致を確認（[outputs/ASA-01-interface.txt](../outputs/ASA-01-interface.txt)） | **PASS** |
| ASA-02 | Security Levelによるトラフィック制御 | inside→outside(新規)はALLOW、outside→inside(新規)はDROP | `packet-tracer input inside icmp ...` → `Action: allow`／`packet-tracer input outside icmp ...` → `Action: drop`（acl-drop）を確認（[outputs/ASA-02-security-level.txt](../outputs/ASA-02-security-level.txt)） | **PASS** |
| ASA-03 | デフォルトルート・疎通性 | Gateway of last resort が 203.0.113.1、R-OUTの物理IF・Loopbackへping到達 | `show route`でS* 0.0.0.0/0 via 203.0.113.1確認、`ping 203.0.113.1`/`ping 192.0.2.10`とも成功率100%（[outputs/ASA-03-default-route.txt](../outputs/ASA-03-default-route.txt)） | **PASS** |
| ASA-04 | ステートフル・インスペクション（TCP） | R-IN起点のTCPが`show conn`に登録され、戻り通信が許可される。outside起点の未承諾新規TCPは拒否される | `show conn`にTCPセッション実登録（flags U）、`packet-tracer input inside tcp ...`→`Action: allow`、`packet-tracer input outside tcp 192.0.2.10 ... 7`→`Action: drop`(acl-drop)を確認。当初`ip http server`(TCP/80)で試験しIOL-XE側の制約により未達成だったため、`service tcp-small-servers`(TCP/7 echo)に切り替えて再試験し合格（詳細: [docs/troubleshooting.md](troubleshooting.md)）（[outputs/ASA-04-stateful-inspection.txt](../outputs/ASA-04-stateful-inspection.txt)） | **PASS** |
| ASA-05 | ICMP Inspection の有効化前後の挙動差 | 無効時: Echo Reply不許可（ping失敗）。有効時: Echo Reply許可（ping成功） | 無効時 `ping` 成功率0%(0/5)×2、`inspect icmp`追加後は成功率100%(5/5)×2。`show service-policy`のICMPカウンタ増加も確認（[outputs/ASA-05-icmp-inspection.txt](../outputs/ASA-05-icmp-inspection.txt)） | **PASS** |
| WS-01 | NAT/PAT アドレス・ポート変換（パケットキャプチャ） | NAT未設定時は両リンクで送信元IP同一。PAT設定後は外側リンクで送信元IPが203.0.113.2に変換 | before: inside/outsideリンクとも`10.10.10.10`で同一。after: insideリンクは`10.10.10.10`のまま、outsideリンクは`203.0.113.2`に変換（ポート23906は変化なし）。`show xlate`で`TCP PAT from inside:10.10.10.10/23906 to outside:203.0.113.2/23906`を確認（[wireshark/analysis/WS-01-nat-pat.txt](../wireshark/analysis/WS-01-nat-pat.txt)） | **PASS** |
| WS-02 | Stateful Inspection（inside起点 vs outside起点） | inside起点は`show conn`に登録され双方向通信が成立。outside起点の新規TCPはinsideリンクに一切到達しない | inside起点: 両リンクでSYN/SYN-ACK/ACK+データ往復を実キャプチャで確認。outside起点: outsideリンクはSYN×2（初回+再送）のみ、insideリンクは**0パケット**（到達せず）を確認（[wireshark/analysis/WS-02-stateful-inspection.txt](../wireshark/analysis/WS-02-stateful-inspection.txt)） | **PASS** |
| WS-03 | ACL permit/deny比較（パケットキャプチャ） | ACL適用前は暗黙denyで到達せず。ACL適用後は許可ポートのみ到達し、非許可ポートは引き続き到達せず | before(port9): insideリンク0パケット。ACL(`permit tcp any host 10.10.10.10 eq discard`)適用後、after-permit(port9): 両リンクで実通信成立(7パケット)。after-deny(port7): 適用後もinsideリンク0パケットのまま（[wireshark/analysis/WS-03-acl-permit-deny.txt](../wireshark/analysis/WS-03-acl-permit-deny.txt)） | **PASS** |
| WS-04 | MTU / IPフラグメンテーション / PMTUD | ASA outside MTU 1300へ縮小時、DF未設定の1400Bpingは断片化。DF設定時はICMP Fragmentation Needed応答でPMTUD動作 | no-DF: outsideリンクで1300B(MF=1)+120B(MF=0, offset=160, 同一IP ID)の2フラグメントを確認、insideリンクは1400B単一パケット。DF: `Packet sent with the DF bit set`+`Success rate 0%`、insideリンクにICMP type3/code4（Fragmentation Needed）応答を確認、outsideリンクは0パケット（[wireshark/analysis/WS-04-mtu-fragmentation-pmtud.txt](../wireshark/analysis/WS-04-mtu-fragmentation-pmtud.txt)） | **PASS** |
| WS-05 | TCP障害解析（RST/service down/retransmission/dup-ack/out-of-order/zero-window） | 各障害モードに対応する実パケット挙動を確認 | RST: 閉ポート接続でSYN→RST,ACK実キャプチャ。service down: outsideリンク停止でSYN×3回（指数バックオフ再送、応答なし）。loss 30%注入: retransmission 1件・duplicate ACK 8件を確認。reorder 40%+loss注入: out-of-order 1件・retransmission 2件・duplicate ACK 5件を確認。**zero-window: NOT RUN**（下記参照） | **PASS**（zero-windowのみNOT RUN、他は実施・確認済み） |

## WS-05 zero-windowについて

R-IN/ASA1/R-OUTはいずれもCLIのみのシンプルなデバイスで、TCP受信バッファの読み出しを意図的に止めてゼロウィンドウを広告させるアプリケーション層の制御手段がありません（スクリプト可能なソケットアプリが存在しない）。実現不可能なものを偽装するよりも正直に**NOT RUN**として報告します。

## 補足

- ASA-04は初回試験（`ip http server`、TCP/80）では接続がR-OUT側で即座に拒否され失敗した。原因切り分けの結果、ASAのACL/設定ではなく、このCML環境の`iol-xe`（Dockerベース）イメージにおいてコントロールプレーンTCPサービス（HTTP/Telnet）がデータプレーンIF経由で応答しないという、当該イメージ固有の制約と判明。`service tcp-small-servers`（Echo/TCP7）に切り替えたところ正常に動作し、ASA自体のステートフル・インスペクション機能は問題なく合格と確認できた。
- 上記の切り分け作業として、R-IN／R-OUTに`service tcp-small-servers`を追加している（可逆設定、`no service tcp-small-servers`で削除可能）。詳細は [docs/troubleshooting.md](troubleshooting.md) を参照。
- ASA1の完全な`show running-config`は、証跡再取得中に発生したコンソール異常のため取得できていない。詳細と正直な事実関係は [docs/troubleshooting.md](troubleshooting.md) の「ASA1コンソール異常」の節、および [configs/ASA1.cfg](../configs/ASA1.cfg) のヘッダコメントを参照。ASA-01・ASA-05で使用した設定断片（インターフェース設定・policy-map設定）は異常発生前に取得した実データであり、試験結果そのものには影響していない。
- WS-01〜05はCML REST APIのリンク単位pcapキャプチャ機能（`/api/v0/labs/{lab}/links/{link}/capture/*`、`/api/v0/pcap/{link_id}`）で取得した実パケットに基づく。WS-01（NAT/PATオブジェクト）、WS-03（ACL）、WS-04（MTU）、WS-05（リンク断・パケットロス/リオーダー注入）で使用した一時設定はすべて試験後に復旧済み。復旧後にASA-01〜03の主要確認コマンドを再実行し、ベースラインの一致を再確認した（詳細: [docs/wireshark-verification.md](wireshark-verification.md)）。
