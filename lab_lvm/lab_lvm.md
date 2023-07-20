# Lab LVM

## Cenário 1 - Rocky Linux

### Proposta

Provisionar espaço de armazenamento, utilizando um novo disco configurado com LVM, dedicado, especificamente, ao diretório */var*, separando do diretório "raiz" (/).

### Objetivo

Implementar tecnologia LVM em um servidor, melhorando flexibilidade e o gerenciamento dos discos de armazenamento, evitando perda de dados.

### Configuração do ambiente virtual

O software de virtualização utilizado foi o Oracle VirtualBox 7.0, com as seguintes especificações:

* Rocky Linux: 9.1 Blue Onyx;
* Kernel: 5.14.0-162;
* CPU: 1;
* Memória RAM: 1024 MB;
* Disco HD de 10 GB, onde:
    * Partição 1º, com 9 GB, destinada a “raiz” (/), com sistema de arquivo XFS;
    * Partição 2°, com 1 GB, destinada ao “swap”, com o sistema de arquivo Swap.
* Placa de rede em modo NAT.

<br>

> O nível de execução utilizado, em todo laboratório, foi o [runlevel 3](https://man7.org/linux/man-pages/man8/runlevel.8.html).
Caso não esteja definido (pode verificar com o comando `who -r` ou `runlevel`), utilize o comando `systemctl set-default multi-user.target`.

#### Gerenciador de Volume Lógico

O [LVM](https://docs.rockylinux.org/pt/books/admin_guide/07-file-systems/#logical-volume-manager-lvm) proporciona uma camada de abstração lógica aos dispositivos de armazenamento físico, incluindo suas partições. Essa abstração “esconde”, do software, o tamanho real do armazenamento dos discos.
Todo o gerenciamento é realizado de forma “on-line”, podendo ser redimencionados e movidos, sem desmontar os discos ou parar aplicações.

Essa flexibilidade só é possível devido ao funcionamento da estrutura do LVM:

* **PV - Physical Volume**: O Volume Físico é o disco (particionado ou não) “preparado” para uso do LVM (os metadados referentes ao LVM são escritos aqui), onde serão criados os grupos de volume;
* **VG - Volume Group**: O Grupo de Volume é uma coleção de um ou mais volumes físicos, servindo como um *pool* de discos, onde serão criados os volumes lógicos;
* **LV - Logical Volume**: O Volume Lógico é uma porção de um grupo de volume, fazendo alusão às partições, no qual será formatado e definido o sistema de arquivo;
* **PE - Physical Extents**: As Extensões Físicas são unidades administrativas, que manipulam o tamanho real dos discos. A quantidade dessas extensões é que definem o tamanho dos volumes lógicos.

<br>

> O dispositivo de armazenamento mais "lento", vai determinar a velocidade, de leitura e escrita, do conjunto (VG). Independente da tecnologia/tipo de dispositivo utilizado (HDD, SSD, NVME), todos os discos serão tratados uniformemente.

### Analisando o ambiente

O servidor, que será utilizado para este laboratório, está particionado de uma forma “tradicional”. Podemos verificar, a seguir, que o mesmo foi dividido apenas com duas partições: raiz (‘/’) e Swap:

```
# fdisk -l
Disk /dev/sda: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x0f13dd47

Device     Boot    Start      End  Sectors  Size Id Type
/dev/sda1  *        2048 18876415 18874368    9G 83 Linux
/dev/sda2       18876416 20971519  2095104 1023M 82 Linux swap / Solaris
```

Como “boa prática” isso não é interessante, visando longo prazo. Recomenda-se que os diretórios onde haverá uma alta demanda, sejam separados da partição raiz. Os diretórios */home* e */var* são exemplos disso: o primeiro para dados dos usuários e segundo fundamental para os serviços do sistema e auditoria (logs).

Para este laboratório, vamos simular (porém é algo bastante comum) que todo o armazenamento disponível, está sendo consumido pelo diretório */var*.
Com o utilitário [fallocate](https://man7.org/linux/man-pages/man1/fallocate.1.html) alocaremos espaço em disco, em um arquivo específico, sem realmente escrever nenhum dado nele.

Vamos criar um arquivo *vardata01* dentro do diretório */var*, utilizando todo o espaço disponível.

Verificando o espaço de armazenamento:

```
# df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/sda1      xfs       9.0G  2.0G  7.1G  22% /
```

Criando o arquivo:

```
# fallocate -l 7G /var/vardata01
```

Conferindo o armazenamento:

```
# df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/sda1      xfs       9.0G  9.0G   44M 100% /
```

E o espaço consumido pelo diretório */var*:

```
# du -shc /var
7.2G	/var
7.2G	total
```

Como já dito, esse cenário pode apresentar problemas graves para serviços essenciais ao sistema.

### Preparando o disco secundário

Precisamos adicionar um novo disco, que terá o tamanho de 20GB, ao servidor e preparar para utilizar LVM.
Para isso, vamos fazer uso do utilitário [fdisk](https://man7.org/linux/man-pages/man8/fdisk.8.html):

```
# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xff8279df.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p

Partition number (1-4, default 1): 1
First sector (2048-41943039, default 2048): 2048
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943039, default 41943039): 41943039

Created a new partition 1 of type 'Linux' and of size 20 GiB.

Command (m for help): t
Selected partition 1
Hex code or alias (type L to list all): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Verificando o disco:

```
# fdisk -l /dev/sdb
Disk /dev/sdb: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xff8279df

Device     Boot Start      End  Sectors Size Id Type
/dev/sdb1        2048 41943039 41940992  20G 8e Linux LVM
```

### Implementando LVM

Com o procedimento anterior, realizamos o particionamento do disco e escolhemos o tipo, que foi o LVM (8e), necessário para prosseguirmos.

Agora podemos criar um volume físico (PV), com o comando [`pvcreate`](https://man7.org/linux/man-pages/man8/pvcreate.8.html):

```
# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
  Creating devices file /etc/lvm/devices/system.devices
```

Verificando os dados do PV:

```
# pvdisplay
  "/dev/sdb1" is a new physical volume of "<20.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb1
  VG Name               
  PV Size               <20.00 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               ws5hvK-G6ue-jMJF-LhkY-kvgZ-TQY4-4zD6wQ
```

Com o volume físico criado, podemos criar um grupo de volume (VG), com o comando [`vgcreate`](https://man7.org/linux/man-pages/man8/vgcreate.8.html):

```
# vgcreate VGVAR /dev/sdb1 
  Volume group "VGVAR" successfully created
```

Verificando os dados do VG:

```
# vgdisplay 
   --- Volume group ---
  VG Name               VGVAR
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <20.00 GiB
  PE Size               4.00 MiB
  Total PE              5119
  Alloc PE / Size       0 / 0   
  Free  PE / Size       5119 / <20.00 GiB
  VG UUID               zsHmJp-f0i8-NQpP-3KQR-RnYj-cpH8-TKtMRw
```

Por fim, criamos o volume lógico (LV):

```
# lvcreate -l 5119 -n bkpvar VGVAR
  Logical volume "bkpvar" created.
```

O comando [`lvcreate`](https://man7.org/linux/man-pages/man8/lvcreate.8.html) possui duas opções específicas para definirmos o tamanho:

* **-l**: Baseado no total de PEs (**Free PE**);
* **-L**: Baseado no espaço total disponível (**Free Size**).

Preferir (opinião pessoal) utilizar a opção **-l**, pelo maior aproveitamento do tamanho do volume, multiplicando o valor do PE Size, que por padrão é 4MB, com o Total PE (5119).

Verificando os dados do LV:

```
# lvdisplay 
  --- Logical volume ---
  LV Path                /dev/VGVAR/bkpvar
  LV Name                bkpvar
  VG Name                VGVAR
  LV UUID                KyTLtY-1DA8-UcF4-Ew13-IEp9-nRON-Tnbzxe
  LV Write Access        read/write
  LV Creation host, time rocky-lvm, 2023-06-26 18:20:23 -0300
  LV Status              available
  LV Size                <20.00 GiB
  Current LE             5119
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:0
```

Agora precisamos construir um sistema de arquivo para o LV. Com o utilitário [mkfs](https://man7.org/linux/man-pages/man8/mkfs.8.html) esse procedimento será realizado de forma simples e rápida:

```
# mkfs.xfs /dev/VGVAR/bkpvar 
```

<br>

> O sistema de arquivo XFS, **NÃO** permite [reduzir](https://access.redhat.com/documentation/pt-br/red_hat_enterprise_linux/8/html/managing_file_systems/comparison-of-xfs-and-ext4_overview-of-available-file-systems) o tamanho.

### Transferindo o diretório

Vamos criar um diretório, que servirá de ponto de montagem temporário, para receber todo o conteúdo do */var*:

```
# mkdir -p /mnt/var_old
```

Após isso, montamos o LV no diretório criado:

```
# mount /dev/mapper/VGVAR-bkpvar /mnt/var_old/
```

Com o utilitário [rsync](https://man7.org/linux/man-pages/man1/rsync.1.html), realizamos o backup completo do diretório /var:

```
# rsync -aX /var/. /mnt/var_old/.
```

Ao finalizar:

```
# df -hT
Filesystem               Type      Size  Used Avail Use% Mounted on
/dev/sda1                xfs       9.0G  9.0G   44M 100% /
/dev/mapper/VGVAR-bkpvar xfs        20G  7.4G   13G  37% /mnt/var_old
```

Com o backup realizado, precisamos montar o conteúdo do LV, no diretório raiz.
A sequência de comandos a seguir, realizará esse procedimento:

```
# umount /dev/mapper/VGVAR-bkpvar

# cd /

# mv var var_bkp

# mkdir var
```

Agora precisamos adicionar esse novo disco à tabela de partição, para que seja montado durante o boot do sistema.
Vamos editar o arquivo [fstab](https://man7.org/linux/man-pages/man5/fstab.5.html) e adicionar a seguinte linha ao final:

```
# vi /etc/fstab
/dev/mapper/VGVAR-bkpvar /var xfs defaults 0 1
```

Ao utilizar o comando `mount -a`, que montará todos os sistemas de arquivos declarados no arquivo fstab, o sistema me apresenta essa mensagem:

```
# mount -a
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
```

Como o sistema GNU/Linux “é seu amigo”, ele informa o que fazer:

```
# systemctl daemon-reload
```

Verificando os pontos de montagem:

```
# mount | grep -E "(/dev/sda|mapper)"
/dev/sda1 on / type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/mapper/VGVAR-bkpvar on /var type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```

Por fim, removemos o diretório o diretório */var_bkp* da raiz do sistema (como boa prática, é recomendável realizar a cópia do diretório para uma unidade de armazenamento externa):

```
# rm -rf /var_bkp

# df -hT
Filesystem               Type      Size  Used Avail Use% Mounted on
/dev/sda1                xfs       9.0G  1.9G  7.2G  21% /
/dev/mapper/VGVAR-bkpvar xfs        20G  7.4G   13G  37% /var
```

<br>

## Cenário 2 - Ubuntu Server

### Proposta

Simular a falha de discos, em um *storage*, e provisionar a solução utilizando LVM.

### Objetivo

Realizar o [Troubleshooting](https://en.wikipedia.org/wiki/Troubleshooting), dos discos de armazenamento, evitando perda de dados.

### Configuração do ambiente virtual

O software de virtualização utilizado foi o Oracle VirtualBox 7.0, com as seguintes especificações:

* Ubuntu Server: 20.04.5 LTS;
* Kernel: 5.4.0-137;
* CPU: 1;
* Memória RAM: 1024 MB;
* Disco HD de 20 GB, onde:
    * Partição 1º, com 1 GB, destinada ao /boot, com sistema de arquivo ext4;
    * Partição 2°, com 19 GB, destinada ao uso do LVM, onde será criado um grupo de volume *vg0*, que terá dois volumes lógicos:
        * lv0-root: Com 18 GB, destinada ao */*,  com sistema de arquivo ext4;
        * lv1-swap: Com 1 GB, destinada ao *swap*, com o sistema de arquivo Swap.
* Placa de rede em modo NAT.

### Preparando os discos

Iremos trabalhar com adição de quatro discos:

* Três discos de 2GB, para simulação da falha, adicionados a um único VG;
* Um disco de 10GB, para a solução.

Todos os discos serão inseridos um por vez. O preenchimento dos discos de 2GB, será feito utilizando o comando `fallocate`.

### Disco 1

Todo o procedimento de como preparar um disco para uso do LVM, já foi apresentado no tópico **Preparando o disco secundário** deste documento.

```
# fdisk -l /dev/sdb
Disk /dev/sdb: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xfcf4fce6

Device     Boot Start     End Sectors Size Id Type
/dev/sdb1        2048 4194303 4192256   2G 8e Linux LVM
```

Próximo passo será criar um VG e montar o volume lógico, de forma persistente, em um diretório */backup*:

```
# pvcreate /dev/sdb1

# vgcreate VGDADOS /dev/sdb1 

# lvcreate -l 511 -n bkpdata VGDADOS

# mkfs.ext4 /dev/VGDADOS/bkpdata

# vi /etc/fstab
/dev/mapper/VGDADOS-bkpdata /backup ext4 defaults 0 1

# mkdir /backup

# mount -a

# df -hT
Filesystem                  Type      Size  Used Avail Use% Mounted on
/dev/mapper/vg0-lv0--root   ext4       18G  5.1G   12G  31% /
/dev/sda2                   ext4      974M  209M  699M  23% /boot
/dev/mapper/VGDADOS-bkpdata ext4      2.0G   24K  1.9G   1% /backup
```

Criando dois arquivos de 1GB:

```
# fallocate -l 1G /backup/data01
# fallocate -l 1G /backup/data02
```

Verificando o espaço:

```
# df -hT /dev/mapper/VGDADOS-bkpdata 
Filesystem                  Type  Size  Used Avail Use% Mounted on
/dev/mapper/VGDADOS-bkpdata ext4  2.0G  2.0G     0 100% /backup
```

Todo o espaço de armazenamento foi preenchido. Vamos desligar a VM e inserir o segundo disco.

### Disco 2

```
# fdisk -l /dev/sdc
Disk /dev/sdc: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xb30924ee

Device     Boot Start     End Sectors Size Id Type
/dev/sdc1        2048 4194303 4192256   2G 8e Linux LVM
```

```
# pvcreate /dev/sdc1
```

Após criar o PV, precisamos adicioná-lo ao grupo **VGDADOS**. Esse procedimento irá aumentar o espaço do grupo, permitindo criar ou estender volumes lógicos.<br>
Utilizaremos o comando [`vgextend`](https://man7.org/linux/man-pages/man8/vgextend.8.html):

```
# vgextend VGDADOS /dev/sdc1
  Volume group "VGDADOS" successfully extended
```

Verificando o espaço do VG:

```
# vgdisplay VGDADOS
  --- Volume group ---
  VG Name               VGDADOS
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               3.99 GiB
  PE Size               4.00 MiB
  Total PE              1022
  Alloc PE / Size       511 / <2.00 GiB
  Free  PE / Size       511 / <2.00 GiB
  VG UUID               gjD86c-z2uq-nUGi-0NpF-1gn0-0o1O-P5Wemi
```

A saída do comando anterior apresenta que, além do VG conter dois discos associados (**Cur PV/Act PV**), foram adicionados novos PEs (511).

A seguir, precisamos adicionar os novos PEs ao LV existente:

```
# lvextend -l+511 -r /dev/VGDADOS/bkpdata
  Size of logical volume VGDADOS/bkpdata changed from <2.00 GiB (511 extents) to 3.99 GiB (1022 extents).
  Logical volume VGDADOS/bkpdata successfully resized.
resize2fs 1.45.5 (07-Jan-2020)
Filesystem at /dev/mapper/VGDADOS-bkpdata is mounted on /backup; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/mapper/VGDADOS-bkpdata is now 1046528 (4k) blocks long.
```

O comando [`lvextend`](https://man7.org/linux/man-pages/man8/lvextend.8.html) é utilizado para adicionar espaço a um LV. As opções utilizadas foram:

* **-l+511**: Estamos adicionando (+) a quantidade de PE (511) disponível no VG;
* **-r**: Redimensiona o sistema de arquivo.

A grande vantagem de se utilizar LVM é poder gerenciar os discos de forma “on-line”, sem a necessidade de paralisar o servidor/serviços/aplicações.

Verificando o espaço do LV:

```
# lvdisplay /dev/VGDADOS/bkpdata
  --- Logical volume ---
  LV Path                /dev/VGDADOS/bkpdata
  LV Name                bkpdata
  VG Name                VGDADOS
  LV UUID                PJzbEc-Bg4I-Sxlz-GiQs-otd6-ZHvx-1XB0yJ
  LV Write Access        read/write
  LV Creation host, time berserk, 2023-06-28 23:51:17 -0300
  LV Status              available
  LV Size                3.99 GiB
  Current LE             1022
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:2
```

Verificando o espaço:

```
# df -hT /dev/mapper/VGDADOS-bkpdata 
Filesystem                  Type  Size  Used Avail Use% Mounted on
/dev/mapper/VGDADOS-bkpdata ext4  3.9G  2.0G  1.8G  52% /backup
```

Criando dois arquivos de 1GB:

```
# fallocate -l 1G /backup/data03
# fallocate -l 1G /backup/data04

# df -hT /dev/mapper/VGDADOS-bkpdata 
Filesystem                  Type  Size  Used Avail Use% Mounted on
/dev/mapper/VGDADOS-bkpdata ext4  3.9G  3.9G     0 100% /backup
```

### Disco 3

```
# fdisk -l /dev/sdd
Disk /dev/sdd: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xb3063cfe

Device     Boot Start     End Sectors Size Id Type
/dev/sdd1        2048 4194303 4192256   2G 8e Linux LVM

# pvcreate /dev/sdd1

# vgextend VGDADOS /dev/sdd1

# lvextend -l+511 -r /dev/VGDADOS/bkpdata

# df -hT /dev/mapper/VGDADOS-bkpdata
Filesystem                  Type  Size  Used Avail Use% Mounted on
/dev/mapper/VGDADOS-bkpdata ext4  5.9G  3.9G  1.8G  70% /backup

# fallocate -l 1G /backup/data05
# fallocate -l 1G /backup/data06

# df -hT /dev/mapper/VGDADOS-bkpdata
Filesystem                  Type  Size  Used Avail Use% Mounted on
/dev/mapper/VGDADOS-bkpdata ext4  5.9G  5.9G     0 100% /backup
```

### Disco 4

Vamos supor que um problema físico foi encontrado no storage. Foi detectado que os três discos de 2GB são a causa desse problema. Precisamos movimentar os dados para o disco novo e substituir os três discos antigos.<br>
Ao concluir a migração, removeremos os discos, do servidor, ficando apenas dois:

* O inicial de 20GB;
* E o de Backup, com 10GB.

```
# fdisk -l /dev/sde
Disk /dev/sde: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x871ae9e8

Device     Boot Start      End  Sectors Size Id Type
/dev/sde1        2048 20971519 20969472  10G 8e Linux LVM
```

```
# pvcreate /dev/sde1

# pvs /dev/vd{b..e}1 -o+devices
  PV         VG      Fmt  Attr PSize   PFree   Devices     
  /dev/sdb1  VGDADOS lvm2 a--   <2.00g      0  /dev/sdb1(0)
  /dev/sdc1  VGDADOS lvm2 a--   <2.00g      0  /dev/sdc1(0)
  /dev/sdd1  VGDADOS lvm2 a--   <2.00g      0  /dev/sdd1(0)
  /dev/sde1  VGDADOS lvm2 a--  <10.00g <10.00g
```

```
# vgextend VGDADOS /dev/sde1

# vgs -o+devices VGDADOS 
  VG       PV  LV  SN Attr   VSize  VFree   Devices     
  VGDADOS   4   1   0 wz--n- 15.98g <10.00g /dev/sde1(0)
```

Após adicionar o quarto disco de 10GB ao VG, precisamos mover as extensões físicas (PE) dos outros discos (PV).<br>
Vamos utilizar o comando [`pvmove`](https://man7.org/linux/man-pages/man8/pvmove.8.html):

```
# pvmove /dev/sdb1 /dev/sde1
# pvmove /dev/sdc1 /dev/sde1
# pvmove /dev/sdd1 /dev/sde1
```

Verificando os PVs:

```
# pvs /dev/vd{b..e}1 -o+devices
  PV         VG      Fmt  Attr PSize   PFree  Devices     
  /dev/sdb1  VGDADOS lvm2 a--   <2.00g <2.00g             
  /dev/sdc1  VGDADOS lvm2 a--   <2.00g <2.00g             
  /dev/sdd1  VGDADOS lvm2 a--   <2.00g <2.00g             
  /dev/sde1  VGDADOS lvm2 a--  <10.00g <4.01g /dev/sde1(0)
  /dev/sde1  VGDADOS lvm2 a--  <10.00g <4.01g
```

O próximo passo será remover os volumes físicos (discos defeituosos) do grupo de volume.<br>
Faremos isso como comando [`vgreduce`](https://man7.org/linux/man-pages/man8/vgreduce.8.html):

```
# vgreduce VGDADOS /dev/sdb1
  Removed "/dev/sdb1" from volume group "VGDADOS"
# vgreduce VGDADOS /dev/sdc1 
  Removed "/dev/sdc1" from volume group "VGDADOS"
# vgreduce VGDADOS /dev/sdd1 
  Removed "/dev/sdd1" from volume group "VGDADOS"
```

Verificando o VG:

```
# vgdisplay VGDADOS
  --- Volume group ---
  VG Name               VGDADOS
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  19
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <10.00 GiB
  PE Size               4.00 MiB
  Total PE              2559
  Alloc PE / Size       1533 / <5.99 GiB
  Free  PE / Size       1026 / <4.01 GiB
  VG UUID               gjD86c-z2uq-nUGi-0NpF-1gn0-0o1O-P5Wemi
```

Com os PVs removidos do VG, precisamos remover a “etiqueta” (metadados), para que o LVM não os reconheça.<br>
O comando [`pvremove`](https://man7.org/linux/man-pages/man8/pvremove.8.html) fará isso:

```
# pvremove /dev/sdb1 
  Labels on physical volume "/dev/sdb1" successfully wiped.
# pvremove /dev/sdc1 
  Labels on physical volume "/dev/sdc1" successfully wiped.
# pvremove /dev/sdd1 
  Labels on physical volume "/dev/sdd1" successfully wiped.
```

Verificando o PV:

```
# pvs -o+devices /dev/sde1
  PV         VG      Fmt  Attr PSize   PFree  Devices     
  /dev/sde1  VGDADOS lvm2 a--  <10.00g <4.01g /dev/sde1(0)
  /dev/sde1  VGDADOS lvm2 a--  <10.00g <4.01g
```

Por fim, vamos adicionar os PEs restantes ao LV:

```
# lvextend -l+1026 -r /dev/VGDADOS/bkpdata
```

Após desligar o servidor e remover os discos, verificamos os dados:

```
# fdisk -l /dev/vd{a..b}
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 16101260-E4BD-48B3-9FFE-BBC12D8C515E

Device       Start      End  Sectors Size Type
/dev/sda1     2048     4095     2048   1M BIOS boot
/dev/sda2     4096  2101247  2097152   1G Linux filesystem
/dev/sda3  2101248 41940991 39839744  19G Linux filesystem

Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x871ae9e8

Device     Boot Start      End  Sectors Size Id Type
/dev/sdb1        2048 20971519 20969472  10G 8e Linux LVM

# df -hT | grep -Ev "(udev|tmpfs|loop)"
Filesystem                  Type      Size  Used Avail Use% Mounted on
/dev/mapper/vg0-lv0--root   ext4       18G  5.1G   12G  31% /
/dev/sda2                   ext4      974M  209M  699M  23% /boot
/dev/mapper/VGDADOS-bkpdata ext4      9.8G  5.9G  3.6G  63% /backup

# ls /backup/
data01  data02  data03  data04  data05  data06
```
