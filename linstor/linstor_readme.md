
Установка паралельного ssh:

`sudo apt install pdsh`

Сначала установим заголовки ядра, потому что репликация DRBD основана на модуле ядра, который должен быть создан на всех нодах:

`pdsh -w ^hosts -R ssh "apt-get install linux-headers-$(uname -r)"`

Потом добавим репозиторий ppa:

`pdsh -w ^hosts -R ssh "add-apt-repository ppa:linbit/linbit-drbd9-stack"`

`pdsh -w ^hosts -R ssh "apt-get update"`

На всех нодах установим эти пакеты:

 `pdsh -w ^hosts -R ssh "apt install drbd-utils drbd-dkms lvm2 -y"`

Загрузим модуль ядра DRBD:

`pdsh -w ^hosts -R ssh "modprobe drbd"`

Проверим, что он точно загрузился:

`pdsh -w ^hosts -R ssh "lsmod | grep -i drbd"`

 ![image](https://github.com/user-attachments/assets/9347e017-9957-406a-a6a8-d20347f104e9)

Убедимся, что он автоматически загружается при запуске:

`pdsh -w ^hosts -R ssh "echo drbd > /etc/modules-load.d/drbd.conf"`

Кластер Linstor состоит из одного активного контроллера, который управляет всей информацией о кластере, и спутников — нод, которые предоставляют хранилище. На ноде, которая будет контроллером, выполним команду:

`apt install linstor-controller linstor-satellite linstor-client`

Эта команда делает контроллер еще и спутником. В моем случае контроллером был ust-k8s-core-linstor-node1. Чтобы сразу запустить контроллер и включить его автоматический запуск при загрузке, выполним команду:

`systemctl enable --now linstor-controller`

`systemctl start linstor-controller`

На оставшихся нодах-спутниках установим следующие пакеты:

`pdsh -w ^hostssat -R ssh "apt install linstor-controller linstor-satellite linstor-client -y"`

Запустим спутник и сделаем так, чтобы он запускался при загрузке:

`pdsh -w ^hostssat -R ssh "systemctl enable --now linstor-satellite"`
`pdsh -w ^hostssat -R ssh "systemctl start linstor-satellite"` 

Теперь можно добавить к контролеру спутники, включая саму эту ноду:

`linstor node create ust-k8s-core-linstor-node1 10.10.35.200
 linstor node create ust-k8s-core-linstor-node2 10.10.35.201
 linstor node create ust-k8s-core-linstor-node3 10.10.35.202
 linstor node create ust-k8s-shb-linstor-node1 10.10.75.200
 linstor node create ust-k8s-etp-linstor-node1 10.10.245.200`

 ![image](https://github.com/user-attachments/assets/ce8de472-a2d0-4460-a370-75db29cb614f)

Проверим харды под линстор:

`pdsh -w ^hosts -R ssh "lsblk"`

Сначала подготовим физический диск или диски на каждом узле. В моем случае это /dev/sdb:


`pdsh -w ^hosts -R ssh "vgcreate k8s /dev/sdb"`

![image](https://github.com/user-attachments/assets/58520461-a90d-44e1-b33a-3bfb1e863151)


Теперь создадим «тонкий» пул для thin provisioning (то есть возможности создавать тома больше доступного места, чтобы потом расширять хранилище по необходимости) и снапшотов:

`pdsh -w ^hosts -R ssh "lvcreate -l 100%FREE --thinpool k8s/lvmthinpool"`

Пора создать пул хранилища на каждой ноде, так что на контроллере выполняем команду:

`linstor storage-pool create lvmthin ust-k8s-core-linstor-node1 linstor-pool-hdd-all k8s/lvmthinpool
linstor storage-pool create lvmthin ust-k8s-core-linstor-node2 linstor-pool-hdd-all k8s/lvmthinpool
linstor storage-pool create lvmthin ust-k8s-core-linstor-node3 linstor-pool-hdd-all k8s/lvmthinpool
linstor storage-pool create lvmthin ust-k8s-shb-linstor-node1 linstor-pool-hdd-all k8s/lvmthinpool
linstor storage-pool create lvmthin ust-k8s-etp-linstor-node1 linstor-pool-hdd-all k8s/lvmthinpool`


![image](https://github.com/user-attachments/assets/56a3804e-208e-45fa-b941-46577bab4cfa)



