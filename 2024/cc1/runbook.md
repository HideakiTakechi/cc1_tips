# ISUCON14 本戦RUNBOOK CC1 
[Portal](https://portal.isucon.net/)  
[参加規約](https://isucon.net/archives/58657108.html)  
[レギュレーション](https://isucon.net/archives/58657116.html)   
[cc1_o11y:攻略環境](https://github.com/HideakiTakechi/cc1_o11y)  
[cc1_webapp：ISUCON14ソース](https://github.com/HideakiTakechi/cc1_webapp)  
[HideakiTakechi](https://github.com/HideakiTakechi) [YK-marigold](https://github.com/YK-marigold) [Eri5yn4ck](https://github.com/Eri5yn4ck)

## 【事前準備】
- [ ] お昼ごはん、おやつ、ドリンク、睡眠  
- [ ] Privateリポジトリへの招待３つを承諾する。(cc1_tips/cc1_o11y/cc1_webapp)
- [ ] github codespaces環境を起動する。（ [cc1_o11y](https://github.com/HideakiTakechi/cc1_o11y) でcreate code spaceする。）
- [ ] ~/.ssh/isucon14.pem(やid_rsaなど)にssh秘密鍵を保存する。
```
$ mkdir ~/.ssh
$ vim ~/.ssh/isucon14.pub       # 公開鍵を保存
$ vim ~/.ssh/isucon14.pem        # 秘密鍵を保存
$ chmod 600 ~/.ssh/isucon14.pem  # 安全なPermissionにする
$ stat -c "%n %a" ~/.ssh/*     # Permissionを確認
/home/codespace/.ssh/isucon14.pem 600
/home/codespace/.ssh/isucon14.pub 644
```
- [ ] .bashrcに以下を追記する。
```
# SSHエージェントの自動起動
if [ -z "$SSH_AGENT_PID" ]; then
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/isucon14.pem   # 必要な秘密鍵を追加
fi
```
```
$ vim ~/.bashrc    # 上記の追記
$ ssh-add -l       # 新しいTerminalを開いて鍵登録されていることを確認
```

## 【競技開始】
> **Info:** 競技日程: 2024年12月8日（日）　競技時間: 10:00 - 18:00（JST）  
運営が事前発表する競技当日の流れを読んでおき、当日に発表する本戦当日マニュアルを読む。  
isucon14 Discordは都度確認する。可能ならOBS Studioで録画する。    

### ■EC2の起動
SREが以下の作業を行う。
- [ ] [ISUCON14 Portal](https://portal.isucon.net)からCloudFormationのテンプレートをダウンロード。  
- [ ] ダウンロードしたテンプレートファイルを元にAWSでスタックを作成しCREATE_COMPLETEを待つ。(EIPの上限5に注意)  
- [ ] 全インスタンスのIPアドレスをDiscordで連絡する。
- [ ] 各インスタンスに~/bin/isucon_toolsを導入しssh keepalive設定などを行う。

### ■ssh接続準備  
- [ ] .ssh/configへ全インスタンスのIPアドレスを追記する。(ForwardAgent=yesでssh-agentもONにしよう)　
- [ ] 出来たインスタンスに各自の攻略環境からssh接続する。user=isuconで接続できるはず。
```
Host isucon14f1
    User isucon
    HostName <ip-address>
    Port 22
    IdentityFile ~/.ssh/isucon14.pem
    ForwardAgent yes
```
```
$ ssh isucon14f1
```
### ■Ansible接続準備  
- [ ] ansibleのinventory.yamlに全インスタンスのIPアドレス追記する。
- [ ] EC2への接続試験を行う。
```
$ cd ansible
$ vi inventory.yaml
$ ansible-playbook -i inventory.yaml test_connection.yaml # EC2へのssh接続試験
```

### ■webappのソース登録とCICD準備
SREが以下の作業を行う。
- [ ] isucon14f1を代表としてソースや設定をcc1_webapp(privateリポジトリ)にpushし環境保全する。  
- [ ] /etcから主要な設定を~/webapp/etcにコピー。（nginx,mysql,systemdなど）
- [ ] webappでgit initして空commit、.gitignoreを作成しgit add,commit
- [ ] リモートリポジトリ登録して初回push。05_add_webapp_to_github.shを改変して実行。または[ここ](https://github.com/ChallengeClub/isucon_tips/blob/main/2023/20231019_webapp_to_github.md)を参考に手動で設定。
- [ ] 他のインスタンスでもpullしてコンフリクトがないことを確認する。  

作業にしばらく時間がかかるので、完了するまで以下などを行う。

### ■調査／ベンチマーク
- [ ] アプリケーションマニュアルを参照しつつブラウザでアクセスして動作を把握する。
- [ ] 初回のベンチマーク試験を行う。結果をDiscordにポストする。
- [ ] サーバ内部調査を行って結果を適宜cc1_tipsに記入しDiscordにポストする。サービス名など確認しよう。

#### ●サーバ内部調査
下記などでインスタンスの概要調査を行う。  
```
$ cat /etc/os-release                   # os確認  
$ grep -v nologin /etc/passwd    # user確認  
$ sudo lsof -P -i | grep -v sshd        # process確認
$ sudo ss -tlp | grep hoge              # process確認
$ sudo ss -tlpn | grep hoge             # process確認
$ service --status-all                  # serviceリスト確認
$ systemctl list-unit-files -t service | grep enabled  # serviceリスト確認
$ systemctl status hoge                 # service確認
$ file hoge                             # file素性確認
$ cat /etc/hosts                        # サイト情報が何か有るかも
```
想定ではWebサーバ(nginx)やRDB（MySQL）やWebアプリがportをLISTENしているはず。  
memcahedやRedisも居るかも。調査結果や不明点を適度にログる。（ログ先は必ずprivateのcc1_tipsに！isucon_tipsにあげちゃダメ！）  
~/webappのディレクトリ構成を調べ必要に応じ/etc, var/logの様子も確認。  
goのサービス名やファイル構成がisucon13(isupipe)どの程度違うか調べてAnsibleに反映/錬成する。  

### ■webappソースコードのclone
SREがwebappのソース登録を完了したら、各自のcodespacesにcloneする。
```
$ git clone git@github.com:HideakiTakechi/cc1_webapp.git
$ mv cc1_webapp webapp
$ cd webapp
$ git status
$ git remote -v               # clone元の確認
$ git checkout -b hidetake    # 各自の開発用ブランチを作成する
```

-----
### 以下2023年版。2024年はcodespacesから作業するので概ね再錬成になるはず。

## 【攻略】
### ■RDB攻略
- USER/PASSを確認しログイン。
- DBとテーブルの調査。（テーブル名、各テーブル行数・バイト数、テーブル定義、インデックスを確認し調査表に書き出し。）
- 初期化データの有無を確認。無ければこの時点でmysqldump実施。★
- [ここ](20231005_mysql_slowlog.md)を参考にslowlogをオン（競技終了時にはoff）
### ■WebApp攻略
- alpの実行
- アプリログの場所の確認、内容の確認（journaldで集めている場合下記で確認できる）
```
journalctl -xef -u <サービス名:isuportsなど>
```
- WebAPIのリストアップ
- N+1の特定と対策
- その他の重くて不要な処理の確認
- キャッシュの検討
### ■HTTPサーバ周辺攻略
- staticコンテンツの取得状況確認
- nginx.conf他の設定の見直し

### ■CI/CD
- 各ユーザごとのブランチで改変しpull,commit,push（クライアント端末が作業しやすい）
- 各自のインスタンスでベンチして問題なさそうならmainにマージ

## 【競技終了】
17:00 リーダーボードの更新停止
### ■最終動作確認(17:00-18:00)
- slowlog off、prometheusサービス停止
- RDBデータ初期化、サービス再起動、再起動試験
- 最終ベンチ
- 18:00 競技終了、ログアウト

### ■結果発表(19:30)
