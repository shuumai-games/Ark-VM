<div align="center">

# AeroVM

[![License](https://img.shields.io/github/license/shuumai-games/Ark-VM?style=for-the-badge)](LICENSE)
[![GitHub Stars](https://img.shields.io/github/stars/shuumai-games/Ark-VM?style=for-the-badge)](https://github.com/shuumai-games/Ark-VM/stargazers)
[![GitHub Issues](https://img.shields.io/github/issues/shuumai-games/Ark-VM?style=for-the-badge)](https://github.com/shuumai-games/Ark-VM/issues)
[![Discord](https://img.shields.io/badge/Discord-Join-5865F2?style=for-the-badge&logo=discord&logoColor=white)](https://discord.gg/UaP8DpsDEK)

**Pterodactyl 向け軽量・無料・オープンソースの QEMU ベース VM エッグ**

KVM なしでも動作します。KVM があればより高速になります。

[クイックスタート](#クイックスタート) • [変数一覧](#エッグ変数) • [サポート](#サポート) • [コントリビュート](#コントリビュート)

</div>

---

## 特徴

- **無料・オープンソース** — ライセンスキー不要、ペイウォールなし
- **KVM ハイブリッド** — デフォルトはソフトウェアエミュレーションで動作し、KVM が利用可能な場合はハードウェアアクセラレーションを使用
- **軽量** — チューニング済みの QEMU フラグ（virtio ディスク/ネット、メモリバルーン）を使用。Alpine ブランクディスクイメージが最小サイズ
- **Pterodactyl ネイティブ** — エッグを 1 つインポートするだけ。パネルの改変不要
- **2 通りのプロビジョニング方法**: ブランクディスクに自前の OS を用意するか、初回起動時からログイン可能な cloud-init イメージ（Debian・Ubuntu・Fedora・Arch・Rocky・AlmaLinux）を選択
- **SSH・VNC・noVNC・SPICE・RDP** — cloud-init イメージでは軽量デスクトップを自動プロビジョニング可能

## 動作要件

| コンポーネント | 最小バージョン |
|--------------|-------------|
| Pterodactyl Panel | 1.11.x |
| Wings | v1.11.9 以上（v1.13.0 まで動作確認済み） |
| Docker | 20.x 以上 |
| ホスト OS | Linux（アクセラレーションには KVM 対応が必要） |

## クイックスタート

**1. （任意・推奨）ノードで KVM を有効化**

ハードウェアアクセラレーション VM を使用するには、エッグをインポートする前に各 Wings ノードで以下を一度だけ実行してください。KVM がなくても AeroVM は動作しますが、低速（ソフトウェアエミュレーション）になります。

**前提条件:** Wings v1.11.9 以上、Go、root 権限が必要です。必要な Go のバージョンは Wings のリリースによって異なります。Wings v1.11.x には **Go 1.21 以上**、v1.12.0 以降には **Go 1.24 以上**（`go.mod` で要求されています）が必要です。インストーラーがこれを自動検出して案内します。

> **注意:** 多くのディストリビューションには Wings v1.12 以降が必要とするバージョンより古い Go がパッケージされています（例: Ubuntu 24 は Go 1.22 を同梱）。インストーラーが「Go が古い」と表示した場合は、最新の Go をインストールして `PATH` の先頭に追加し、同じシェルで再実行してください:
>
> ```bash
> rm -rf /usr/local/go
> curl -fsSL https://go.dev/dl/go1.24.0.linux-amd64.tar.gz -o /tmp/go.tar.gz   # arm64 の場合: amd64 -> arm64 に変更
> tar -C /usr/local -xzf /tmp/go.tar.gz
> export PATH=/usr/local/go/bin:$PATH
> go version   # go1.24.0（以上）と表示されることを確認
> ```

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/shuumai-games/Ark-VM/main/wings-patch/install.sh)
```

スクリプトの実行内容:
1. インストール済みの Wings バージョン（v1.11.9 以上）を検出し、対応するパッチと必要な Go バージョンを選択
2. サービスを停止する前にパッチ済み Wings をビルド（ビルド失敗時は既存の Wings に影響なし）
3. Wings を停止し、現在のバイナリをバックアップ（`wings.bak.<タイムスタンプ>`）してパッチ済みバイナリをインストール
4. `/dev/kvm` のパーミッションを永続的に設定（udev ルールとグループ）
5. Wings を再起動（途中でエラーが発生しても EXIT トラップで再起動を保証）

パッチ自体は、すべてのサーバーコンテナに `/dev/kvm` デバイスマッピングを追加し、コンテナプロセスにホストの `kvm` グループを付与することで、ゲストがハードウェアアクセラレーションを利用できるようにします。`container.go` は Wings v1.12 以降（新しい Docker SDK）向け、`container_legacy.go` は v1.11.x（旧 Docker SDK）向けで、インストーラーが自動的に選択します。

**元に戻す場合:**
```bash
# install.sh が作成したバックアップを復元
cp /usr/local/bin/wings.bak.<タイムスタンプ> /usr/local/bin/wings
rm /etc/udev/rules.d/99-arkvm-kvm.rules
udevadm control --reload-rules
systemctl restart wings
```

**2. エッグをダウンロード**

このリポジトリから [`egg-arkvm.json`](egg/egg-arkvm.json) をダウンロードしてください。

**3. Pterodactyl にインポート**

**管理画面 → Nests → エッグをインポート** からファイルをアップロードします。

**4. サーバーを作成**

AeroVM エッグを使用して新しいサーバーを作成し、**Docker イメージ**を選択します:

- **ブランクディスク**（Alpine、Ubuntu 22.04/24.04/26.04 LTS）: 空のディスクです。OS は自分で用意します（例: SFTP でディスクイメージをアップロード）。
- **Cloud-init**（Debian 12、Ubuntu 22.04、Ubuntu 24.04、Fedora、Arch Linux、Rocky Linux、AlmaLinux）: 公式クラウドイメージからディスクが事前プロビジョニングされ、初回起動時からログイン可能です。`OS_HOSTNAME`・`OS_PASSWORD`・`OS_PUBKEY` で設定できます。`DISPLAY_MODE` を `vnc`/`novnc`/`spice`/`rdp` に設定すると、cloud-init が自動でデスクトップ環境をインストールします（初回起動に数分追加でかかります）。

RAM、CPU、ディスクはエッグ変数で設定します。サーバーを起動すると VM が自動的に起動し、手順 1 を適用したノードでは KVM アクセラレーションが使用されます。

> **注意:** `ADDITIONAL_PORTS` に指定するポートは、Pterodactyl パネルの**アロケーション**にも割り当てる必要があります。QEMU 側でポートフォワードするだけでは不十分で、Docker でもポートを公開する必要があります。

## エッグ変数

| 変数 | 説明 | デフォルト |
|------|------|----------|
| `VM_RAM_MB` | VM に割り当てる RAM（MB）。cloud-init イメージには 1024 以上必要。ソフトウェアエミュレーション下では 2048 以上を推奨 | `1024` |
| `VM_CPU_CORES` | 仮想 CPU コア数（最大 16）。cloud-init イメージには 2 以上を推奨 | `2` |
| `VM_DISK_GB` | 仮想ディスクサイズ（GB） | `20` |
| `DISPLAY_MODE` | `ssh` / `vnc` / `novnc` / `spice` / `rdp` / `none` | `ssh` |
| `KVM` | `auto`（ベアメタルでは KVM 使用、ノード自体が VM の場合はソフトウェアエミュレーション）/ `off`（強制的にソフトウェアエミュレーション）/ `on`（ネステッドも含め強制的に KVM 使用） | `auto` |
| `ADDITIONAL_PORTS` | 追加ポートフォワード（例: `8080-80,443`） | — |
| `UEFI` | UEFI ファームウェアを有効化（`0` または `1`） | `0` |
| `OS_HOSTNAME` | ゲストのホスト名（cloud-init イメージのみ） | `arkvm` |
| `OS_PASSWORD` | root/SSH パスワード（cloud-init イメージのみ）。空白の場合は自動生成（初回起動時にコンソールに表示） | — |
| `OS_PUBKEY` | SSH 公開鍵（cloud-init イメージのみ）。設定するとパスワードによる SSH ログインが無効になる | — |
| `PACKAGE_UPDATE` | 毎回起動時にパッケージを更新（cloud-init イメージのみ、`0` または `1`） | `0` |
| `IPV4_MODE` | `disabled`（追加ポートを無視）/ `user`（追加ポートをフォワード）/ `all`（ポート 1〜1024 もフォワード、起動が遅くなる） | `user` |
| `OVERWRITE_HOST` | VM 内部に表示されるホスト名/製品名を上書き（neofetch、dmidecode 等） | — |
| `OVERWRITE_IP` | 起動時に表示される接続情報の IP を上書き | — |
| `BANNER` | カスタム起動バナー（`\n` および Bash カラーコード対応） | — |

> `VM_RAM_MB` および `VM_CPU_CORES` は Pterodactyl のリソース制限とは独立しています。実際にノードがサポートできる値を設定してください。
>
> **初回起動がタイムアウトしたり緊急モードに落ちる場合:** 使用可能な cloud-init イメージは完全な systemd ユーザーランドを起動するため、最小スペック以上のリソースが必要です。特にノードがソフトウェアエミュレーション（`KVM=off`、またはネステッドノードの `auto`）で VM を実行している場合はなおさらです。**最低 2048 MB RAM と 2 コア**を割り当ててください。リソースが不足していると、VM が systemd の 90 秒のデバイス検出タイムアウトに間に合わず、緊急シェルに落ちることがあります。実 KVM を持つベアメタルノードであれば 1024 MB / 1 コアでも通常は問題ありません。
>
> `OS_HOSTNAME`・`OS_PASSWORD`・`OS_PUBKEY`・`PACKAGE_UPDATE` は cloud-init Docker イメージにのみ効果があります。ブランクディスクイメージ（Alpine/Ubuntu LTS）は OS がインストールされていないため、これらの設定は無視されます。
>
> **Pterodactyl の「ディスクスペース」**（コンテナのディスク容量制限、MiB 単位）は `VM_DISK_GB` より大きく設定する必要があります。VM の `disk.qcow2` は最大 `VM_DISK_GB` まで拡大可能であり、Pterodactyl のディスク制限を超えるとサーバーが停止します。`VM_DISK_GB=20` の場合は、ディスクスペースを `25000` MiB 以上、または `0`（無制限）に設定してください。

> **ネステッド仮想化に関する注意:** Pterodactyl **ノード自体が仮想マシン**（例: 他の VPS ホストの VM）上で動いている場合、その中でハードウェア KVM を使用すると「ネステッド仮想化」になります。一部のホスト（特定の AMD 環境など）ではこれが不安定で、**ノード全体がカーネルパニックを起こす**可能性があります。AeroVM にはこれに対する保護機能があります。デフォルトの `KVM=auto` モードでは CPU の `hypervisor` フラグによってノードの仮想化を検出し、自動的にソフトウェアエミュレーションを使用するため、KVM パッチを適用済みのノードでもクラッシュしません。ネステッド KVM を強制したい場合のみ `KVM=on` を設定してください（対応していることが確認済みのホストに限ります）。ベアメタルノードでは `auto` が通常どおり KVM を使用します。なお、ソフトウェアエミュレーションは動作しますが**大幅に低速**です。Ubuntu Desktop のような重いゲストは実際の KVM なしでは実用的ではないため、パフォーマンスを求める場合はベアメタルノードで AeroVM を実行してください。

## ディスプレイモード

| モード | 説明 |
|--------|------|
| `ssh` | サーバーのメイン Pterodactyl ポートを経由した SSH |
| `vnc` | ポート 5900 での生の VNC |
| `novnc` | ポート 6080 でのブラウザベース VNC |
| `spice` | ポート 5900 での SPICE プロトコル（ネイティブ SPICE クライアントで接続） |
| `rdp` | ポート 3389 での RDP — **cloud-init イメージのみ**、`arkvm` ユーザーでログイン |
| `none` | ヘッドレス — ディスプレイ出力なし |

cloud-init イメージで `vnc`/`novnc`/`spice`/`rdp` を選択すると、cloud-init が初回起動時に軽量な XFCE デスクトップ（`rdp` の場合は xrdp も）をインストールします。デスクトップが使用可能になるまで数分かかります。デスクトップセッション用に `arkvm` sudo ユーザー（パスワード: `OS_PASSWORD`）が作成されます。`vnc`/`novnc`/`spice` では `arkvm` として自動ログインし、`rdp` では接続時にログイン情報の入力を求められます。ブランクディスクイメージでは `vnc`/`novnc`/`spice` は VM のコンソール/インストーラー画面を表示するだけで（まだ OS がありません）、`rdp` は利用できません。

> **注意:** `vnc`/`spice`（ポート `5900`）、`novnc`（ポート `6080`）、`rdp`（ポート `3389`）はいずれも、`ADDITIONAL_PORTS` と同様に Pterodactyl パネルで**アロケーション**としてポートを割り当てる必要があります。QEMU がポートをリッスンするだけでは、Docker/Wings がポートを公開していなければ不十分です。

## 仕組み

コンテナのエントリポイントは [`scripts/start.sh`](scripts/start.sh) で、毎回の起動時に実行されてエッグ変数から QEMU コマンドラインを構築します:

1. **入力値の検証** — 整数値は範囲チェック（int64 オーバーフロー対策済み）、`DISPLAY_MODE`/`IPV4_MODE` は許可値のセットと照合、ホスト名は `[a-zA-Z0-9-]{1,63}` に対して検証、`OS_PASSWORD`/`OS_PUBKEY` は改行が含まれていないか確認（cloud-init YAML に埋め込まれるため）。
2. **KVM の検出** — `/dev/kvm` が読み書き可能な場合は `-enable-kvm -cpu host` と `aio=native` で QEMU を実行。それ以外はソフトウェアエミュレーション（`-cpu qemu64`、`aio=threads`）にフォールバック。どちらの場合でも VM は起動します。
3. **ディスクのプロビジョニング**（`/home/container/disk.qcow2`、再起動をまたいで保持）:
   - *ブランクディスクイメージ*: `VM_DISK_GB` サイズの空の `qcow2` を作成。
   - *Cloud-init イメージ*: バンドルされたクラウドイメージ（`/opt/base-image/base.qcow2`）をコピーし、`VM_DISK_GB` まで拡張（イメージ自体のサイズより小さい場合はスキップ）。
4. **cloud-init シードの構築**（cloud-init イメージのみ）— `xorriso` で NoCloud `cidata` ISO（`meta-data` + `user-data`）を生成して `-cdrom` で接続。ホスト名、root パスワード（`chpasswd`）、SSH キー（指定時）、オプションのパッケージ更新を設定。ランダムなインスタンス ID を保持（`.cloud-init-instance-id`）することで、次回起動時に cloud-init が再実行されないようにします。グラフィカルな `DISPLAY_MODE` の場合はデスクトップセッション用の `arkvm` sudo ユーザーも作成され、XFCE + LightDM のインストール（`rdp` の場合は `xrdp`、`spice` の場合は `spice-vdagent`）を `runcmd` で実行します（イメージの `CLOUD_OS_FAMILY` に応じたパッケージマネージャーを使用: `debian`/`fedora`/`rhel`/`arch`）。
5. **ネットワークの設定** — QEMU のユーザーモードネットワーク（`hostfwd` ルール）: メインの Pterodactyl ポート → ゲストの `22`、RDP の `3389`（rdp モード）、ポート `1-1024` 範囲（`IPV4_MODE=all`）、`ADDITIONAL_PORTS`。ディスプレイが既に使用しているポート（5900/6080）と重複は除外。
6. **ディスプレイの選択** — `ssh` はシリアルコンソール（`-nographic -serial mon:stdio`）を使用。`vnc`/`novnc` は VNC `:0`（5900）上の `-vga virtio` を使用し、`novnc` はさらに noVNC→VNC プロキシをポート 6080 で起動。`spice` は `-vga qxl` と `-spice` を使用。`rdp`/`none` はヘッドレスで実行。
7. **QEMU の起動** — virtio ディスク/ネット、メモリバルーン、オプションの `-bios`（`UEFI` 有効時は OVMF）、オプションの `-smbios`（`OVERWRITE_HOST` 設定時）で `exec qemu-system-x86_64` を実行。

**ノードでの KVM（任意パッチ）.** Wings は各サーバーコンテナを明示的な数値 `uid:gid` で起動するため、Docker はイメージ自身のグループメンバーシップを無視します。そのため Dockerfile でコンテナユーザーを `kvm` グループに追加するだけでは不十分です。[`wings-patch/`](wings-patch/) のパッチは、Wings がコンテナに `/dev/kvm` をマッピングし、Docker の `GroupAdd` 経由でホストの実際の `kvm` グループ GID を付与するようにすることで、ゲストがデバイスを実際に開けるようにします。

## イメージとバージョン

すべてのイメージは `ghcr.io/shuumai-games/arkvm:<タグ>` で公開されています。

| タグ | ベースイメージ | バンドルされたゲスト OS |
|-----|--------------|----------------------|
| `alpine` | Alpine 3.19 | なし（ブランクディスク） |
| `ubuntu-22.04` / `ubuntu-24.04` / `ubuntu-26.04` | Ubuntu LTS | なし（ブランクディスク） |
| `guest-debian-12` | Alpine 3.19 | Debian 12 (bookworm) クラウドイメージ |
| `guest-ubuntu-22.04` / `guest-ubuntu-24.04` | Alpine 3.19 | Ubuntu 22.04 (jammy) / 24.04 (noble) クラウドイメージ |
| `guest-fedora` | Alpine 3.19 | Fedora Cloud Base 44 |
| `guest-arch` | Alpine 3.19 | Arch Linux（最新クラウドイメージ） |
| `guest-rockylinux` / `guest-almalinux` | Alpine 3.19 | Rocky Linux 9 / AlmaLinux 9 GenericCloud |

cloud-init イメージはビルド時に公式の `qcow2`/`img` をバンドルするため、サーバー作成時に追加のダウンロードは発生しません。デスクトップセッションには **XFCE4 + LightDM**（RDP の場合は **xrdp**、SPICE の場合は **spice-vdagent**）を使用します。

## プロジェクト構造

```
AeroVM/
├── egg/
│   └── egg-arkvm.json               # Pterodactyl エッグ（ユーザー向け）
├── docker/
│   ├── Dockerfile.alpine               # Alpine ベースイメージ、ブランクディスク（最軽量）
│   ├── Dockerfile.ubuntu-22.04         # Ubuntu 22.04 LTS イメージ、ブランクディスク
│   ├── Dockerfile.ubuntu-24.04         # Ubuntu 24.04 LTS イメージ、ブランクディスク
│   ├── Dockerfile.ubuntu-26.04         # Ubuntu 26.04 LTS イメージ、ブランクディスク
│   ├── Dockerfile.guest-debian-12      # Debian 12 クラウドイメージをバンドル、cloud-init 対応
│   ├── Dockerfile.guest-ubuntu-22.04   # Ubuntu 22.04 クラウドイメージをバンドル、cloud-init 対応
│   ├── Dockerfile.guest-ubuntu-24.04   # Ubuntu 24.04 クラウドイメージをバンドル、cloud-init 対応
│   ├── Dockerfile.guest-fedora         # Fedora クラウドイメージをバンドル、cloud-init 対応
│   ├── Dockerfile.guest-arch           # Arch Linux クラウドイメージをバンドル、cloud-init 対応
│   ├── Dockerfile.guest-rockylinux     # Rocky Linux クラウドイメージをバンドル、cloud-init 対応
│   └── Dockerfile.guest-almalinux      # AlmaLinux クラウドイメージをバンドル、cloud-init 対応
├── scripts/
│   └── start.sh                        # VM 起動スクリプト
├── wings-patch/
│   ├── container.go                    # パッチ済み Wings ファイル（KVM 対応）Wings v1.12 以降用
│   ├── container_legacy.go             # パッチ済み Wings ファイル（KVM 対応）Wings v1.11.x 用
│   └── install.sh                      # ワンコマンドの KVM パッチインストーラー（Wings バージョン自動検出）
├── .github/workflows/
│   └── build.yml                       # 全イメージを自動ビルドして ghcr.io にプッシュ
└── .gitattributes                      # LF 改行コードを強制（CRLF は Linux 上でシェルスクリプトを壊す）
```

## サポート

質問・ヘルプ・フィードバックは Discord へ: https://discord.gg/UaP8DpsDEK

## コントリビュート

PR・Issue 歓迎です。

1. リポジトリをフォーク
2. フィーチャーブランチを作成
3. 変更をコミット
4. プルリクエストを作成

## ライセンス

MIT — 詳細は [LICENSE](LICENSE) を参照してください
