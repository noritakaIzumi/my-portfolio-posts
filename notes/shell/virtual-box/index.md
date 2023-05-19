---
title: VirtualBox
menu:
  notes:
    name: VirtualBox
    identifier: notes-shell-virtual-box
    parent: notes-shell
    weight: 10
---

{{< note title="Shutdown All VMs" >}}

VM を自動的にすべて終了させるスクリプト  
すべての VM が終了するまで待ちます。

```bash
#!/usr/bin/env bash
INTERVAL=3

VBoxManage list runningvms | cut -d' ' -f1 | xargs -I{} VBoxManage controlvm {} acpipowerbutton

while [ $(VBoxManage list runningvms | wc -l) -ne 0 ]
do
    echo "shutting down all VMs..."
    sleep ${INTERVAL}
done
```

{{< /note >}}
