# Proxmox IOMMU確認・設定メモ

年刊EDP2026「Proxmoxやりましょう」の補足資料です。ルーターをProxmox上へ載せる案を検討した際に行った、IOMMUとPCI passthroughの確認、設定、検証、復旧手順をまとめます。

- 対象ホスト: `terumox1`
- 機種: HPE ProLiant ML110 Gen9
- NIC: Intel X520-DA2
- PCIアドレス: `0000:09:00.0`、`0000:09:00.1`

IOMMUが有効でも、同じIOMMU groupに入ったデバイスは原則としてまとめてVMへ渡す必要があります。目的のNICがほかの機器から分離されているかを、設定前後で確認します。

## 現状を確認する

```bash
# VM、LXC、ストレージの一覧を確認する。
qm list
pct list
pvesm status

# ブートローダーの管理方式を確認する。
findmnt /boot/efi
proxmox-boot-tool status

# NICの型番とPCIアドレスを確認する。
lspci -nn | grep -i -E ' ethernet| network|82599| x520| intel| realtek'

# 現在のkernel起動パラメータとIOMMU関連ログを確認する。
cat /proc/cmdline
dmesg | grep -i -E ' DMAR| IOMMU'

# 対象NICが属するIOMMU groupを確認する。
find /sys/kernel/iommu_groups -type l | sort -V | \
  grep -E '0000:09:00.0|0000:09:00.1'
```

`terumox1`で確認したX520-DA2の2ポートは次のとおりです。

```text
0000:09:00.0 Ethernet controller:
  Intel Corporation 82599ES 10-Gigabit SFI/SFP+
0000:09:00.1 Ethernet controller:
  Intel Corporation 82599ES 10-Gigabit SFI/SFP+
```

## ブートローダーの管理方式を判定する

Proxmoxでは、同じUEFI起動でも`proxmox-boot-tool`で管理している場合と、GRUBで管理している場合があります。

- `proxmox-boot-tool`管理: `/etc/kernel/cmdline`を編集し、`proxmox-boot-tool refresh`で反映する。
- GRUB管理: `/etc/default/grub`を編集し、`update-grub`で反映する。

`terumox1`はUEFIで起動していましたが、`proxmox-boot-tool status`では管理対象のUUIDを確認できませんでした。そのため、このホストはGRUB管理として扱いました。

## BIOS/UEFIとGRUBを設定する

BIOS/UEFIでIntel VT-dを有効にします。VT-xはCPUの仮想化支援、VT-dはPCIデバイスをVMへ割り当てるための機能です。

このホストの初回確認では、`/proc/cmdline`に`intel_iommu=on`がなく、IOMMU関連ログと対象NICのgroupも確認できなかったため、起動パラメータを明示的に追加しました。

```bash
# 編集前にバックアップを取り、GRUB設定を開く。
cp /etc/default/grub /root/grub.before-iommu
nano /etc/default/grub
```

`GRUB_CMDLINE_LINUX_DEFAULT`を次のように設定しました。

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

設定を反映して再起動します。

```bash
# GRUB設定を反映し、再起動する。
update-grub
reboot
```

再起動後、`/proc/cmdline`に`intel_iommu=on iommu=pt`が入り、`dmesg`に`DMAR: IOMMU enabled`が出ていることを確認します。

## IOMMU groupを確認する

```bash
# 起動パラメータとIOMMU関連ログを再確認する。
cat /proc/cmdline
dmesg | grep -i -E 'DMAR|IOMMU'

# passthrough候補のPCIデバイスが属するIOMMU groupを確認する。
readlink /sys/bus/pci/devices/0000:09:00.0/iommu_group
readlink /sys/bus/pci/devices/0000:09:00.1/iommu_group
lspci -D | grep -i ethernet
find /sys/kernel/iommu_groups -type l | sort -V | \
  grep -E '0000:09:00.0|0000:09:00.1'
```

`terumox1`では、X520の2ポートが別々のIOMMU groupへ分かれました。

```text
0000:09:00.0 -> group 56
0000:09:00.1 -> group 57
```

この結果なら、片方のポートだけをWAN用NICとしてpassthroughし、もう片方をProxmoxホスト側へ残せます。ただし、groupの分かれ方は機種、BIOS/UEFI、PCIeスロットによって異なります。

## 設定を戻す

起動しなくなった場合は、物理コンソールからGRUBの編集画面で追加したパラメータを一時的に外して起動します。起動後、退避した設定を戻してGRUBを再生成します。

```bash
# IOMMU設定前のGRUBへ戻す。
cp /root/grub.before-iommu /etc/default/grub
update-grub
reboot
```

技術的にはNICを分離できましたが、ルーターをVM化するとProxmoxホストの再起動や障害で家全体の通信まで止まります。今回は保守が面倒になると判断し、ルーターVM案そのものは採用しませんでした。

## 参照先

- [How to enable IOMMU for PCI Passthrough](https://www.reddit.com/r/Proxmox/comments/1brl7vw/guide_how_to_enable_iommu_for_pci_passthrough/?tl=ja)
- [Intel X520-DA2 IOMMU groups](https://forum.proxmox.com/threads/intel-x520-da2-iommu-groups.118261/)
- [SR-IOVを有効にする](https://tenforward.hatenablog.com/entry/20111206/1323171401)
- [ProxmoxでIntel NICをSR-IOV](https://note.com/enokiko/n/n4b4b4e4c399b)
