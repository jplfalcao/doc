# Lab IPtables

## Cenário 1 - Ubuntu Server

### Proposta

Montar um ambiente, de acordo com a topologia abaixo, provendo comunicação entre os servidores.  

![Topologia](https://github.com/jplfalcao/doc/blob/main/lab-iptables/topologia.png)

### Objetivo

Realizar a instalação, configuração e comunicação entre os servidores, utilizando GNU/Linux, definindo regras de Firewall para ambos.

Para este laboratório serão criados quatro máquinas virtuais (VMs), que deverão seguir, **sem exceção**, as seguintes especificações:

VM01 – Roteador e Firewall:
* Quatro placas de rede;
* Ativar roteamento;
* Aplicar regras de iptables, liberando a comunicação das VMs (2, 3 e 4) para a Internet;
* Ativar serviço SSH para a porta 7654;
* Criar usuário adm_lab com poder de sudo;
* Usar o usuário adm_lab como usuário administrativo;
* Bloquear o usuário root via console e ssh.

VM02:
* Configurar a VM com IP da Rede A;
* Liberar e validar comunicação via ICMP com a VM04;
* Bloquear comunicação externa, exceto ICMP para a WEB;
* Ativar serviço SSH para a porta 7654;
* Criar usuário adm_lab com poder de sudo;
* Usar o usuário adm_lab como usuário administrativo;
* Bloquear o usuário root via console e ssh.

VM03:
* Configurar a VM com IP da Rede B;
* Liberar acesso a WEB;
* Liberar e validar comunicação via SSH e ICMP com o Firewall;
* Ativar serviço SSH para a porta 7654;
* Criar usuário adm_lab com poder de sudo;
* Usar o usuário adm_lab como usuário administrativo;
* Bloquear usuário root via console e ssh.

VM04:
* Configurar a VM com IP da Rede C;
* Liberar acesso ICMP para o DNS do google (8.8.8.8 e 8.8.4.4);
* Liberar e validar comunicação via ICMP com a VM02;
* Ativar serviço SSH para a porta 7654;
* Criar usuário adm_lab com poder de sudo;
* Usar o usuário adm_lab como usuário administrativo;
* Bloquear usuário root via console e ssh.

<br>

> As políticas padrão do iptables, em todas as VMs, devem estar definidas como **DROP**.

### Configuração das máquinas virtuais

### VM01 - Roteador e Firewall

O software de virtualização utilizado foi o Oracle VirtualBox 7.0, com as seguintes especificações:
* Ubuntu Server: 20.04.5 LTS;
* Kernel: 5.4.0-137;
* CPU: 1;
* Memória RAM: 512 MB;
* Disco HD de 20 GB particionado, onde:
    * Partição 1º, com 10 GB, destinada a “raiz” (/), com sistema de arquivo ext4;
    * Partição 2°, com 10 GB, destinada ao uso de LVM (*Logical Volume Manager*), onde será criado um grupo de volume *vg0*, que terá dois volumes lógicos:
        * lv0-var: Com 9 GB destinada ao */var*, com sistema de arquivo ext4;
        * lv1-swap: Com 1 GB destinada ao *swap*, com o sistema de arquivo Swap.
* Quatro placas de rede, onde:
    * WAN: Modo *Bridge*, com endereço: 192.168.13.140/24;
    * Rede A: Rede Interna *netA*, com endereço: 192.168.100.1/24;
    * Rede B: Rede Interna *netB*, com endereço: 192.168.150.1/24;
    * Rede C: Rede Interna *netC*, com endereço: 192.168.200.1/24.

#### Gerenciamento de usuário

Criando um usuário, adm_lab, com acesso administrativo:
```shell
useradd -m -s /bin/shell adm_lab
usermod -aG sudo adm_lab
passwd adm_lab
```

Bloqueando o usuário root:
```shell
passwd -l root 
usermod -p '!' root
```

#### Configurando as interfaces de rede

As interfaces de rede serão configuradas utilizando um arquivo de configuração individual, localizado no diretório */etc/netplan*, para cada interface.
O [Netplan](https://netplan.io/) trabalha com arquivos [.yaml](https://yaml.org/), e eles tem uma particularidade: Precisam, e é obrigatório, que respeitem sua indentação.
Qualquer erro de digitação ou indentação, o arquivo será invalidado.

> Foram utilizados dois espaços em branco, como indentação, em todos os arquivos *.yaml*.

WAN:
```shell
vi /etc/netplan/00-installer-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses: [192.168.13.140/24]
      gateway4: 192.168.13.1
      nameservers:
        addresses: [192.168.13.1, 8.8.8.8]
```

Rede A:
```shell
vi etc/netplan/enp0s8.yaml 
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      addresses: [192.168.100.1/24]
```

Rede B:
```shell
vi etc/netplan/enp0s9.yaml 
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s9:
      addresses: [192.168.150.1/24]
```

Rede C:
```shell
vi etc/netplan/enp0s10.yaml 
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s10:
      addresses: [192.168.200.1/24]
```

Validando as configurações:
```shell
netplan generate
netplan apply
```

#### Configurando o serviço SSH

Alterando a porta padrão e bloqueando acesso com usuário root:
```shell
vi /etc/ssh/sshd_config
Port 7654
PermitRootLogin no
```

Em seguida, precisamos reiniciar o serviço do SSH:
```shell
systemctl restart sshd.service
```

#### Considerações sobre o firewall

O iptables [(Netfilter)](https://www.netfilter.org/) é um firewall utilizado para filtragem de pacotes de redes.
Todo o seu funcionamento se baseia na criação e administração de regras, referente a endereços/portas de origem/destino e protocolos.

Após a instalação do servidor Ubuntu o iptables, por padrão, não vem com regras definidas e suas políticas padrão estão definidas para aceitar qualquer tráfego com origem de qualquer lugar.
Esse cenário é um grave problema de segurança, que deve ser evitado com o conhecimento do administrador do sistema, para manipulação precisa das regras que serão utilizadas.

#### Como o iptables funciona

As opções e parâmetros do comando [iptables](https://man7.org/linux/man-pages/man8/iptables.8.html), que foram utilizados durante todo o laboratório, serão descritos a seguir (letras maiúsculas **precisam** ser respeitadas):
* **-A**: Adiciona uma regra ao final da cadeia (--append);
* **-I**: Insere (--insert) uma regra sempre, por padrão, ao topo da cadeia (pode ser especificado um número para indicar a posição);
* **-P**: Define a política padrão da cadeia (--policy);
* **-t**: Seleciona a tabela (--table);
* **-i**: Interface de entrada (--in-interface);
* **-o**: Interface de saída (--out-interface);
* **-p**: Específica o protocolo que será utilizado (--protocol);
* **-s**: Endereço IP de origem (--source);
* **-d**: Endereço IP de destino (--destination);
* **--dport**: (--destination-port) Porta de destino (quando é declarado um protocolo (*-p*), é necessário declarar uma porta);
* **-j**: Define qual ação será tomada com cada regra (--jump). Os alvos utilizados foram:
    * **ACCEPT**: Aceita a passagem dos pacotes;
    * **DROP**: Bloqueia e descarta os pacotes;
    * **LOG**: Realiza o registro dos pacotes.
* **-m state**: Habilita o módulo *state*, no qual monitora o estado da conexão;
* **--state ESTABLISHED,RELATED**: Informa ao módulo state que todas as conexões que já foram estabelecidas e relatadas, sejam autorizadas.
* **nat**: Tabela utilizada para redirecionamento de pacotes, onde:
    * **POSTROUTING**: Altera os pacotes durante o encaminhamento:
        * **MASQUERADE**: Permite o mascaramento de endereços IP de uma rede interna, utilizando um endereço IP externo;

#### Aplicando regras

Como as VMs (2,3 e 4) estão sem conexão a internet, precisamos liberar o acesso, através do encaminhamento de pacotes:
```shell
iptables -t nat -I POSTROUTING -o enp0s3 -s 192.168.100.135 -j MASQUERADE
iptables -t nat -I POSTROUTING -o enp0s3 -s 192.168.150.230 -j MASQUERADE
iptables -t nat -I POSTROUTING -o enp0s3 -s 192.168.200.30 -j MASQUERADE
```

Para habilitar a comunicação entre as interfaces de rede, precisamos descomentar ou adicionar (caso não exista) a linha no arquivo a seguir:
```shell
vi /etc/sysctl.conf
net.ipv4.ip_forward=1
```

Isso é necessário devido ao kernel, por padrão e segurança, não permitir comunicação entre as interfaces de rede.

Validando a modificação:
```shell
sysctl -p
```

Opcionalmente podemos habilitar da seguinte forma, porém ficando apenas em memória:
```shell
echo "1" > /proc/sys/net/ipv4/ip_forward
```

#### Tabela filter

Todas as regras de filtragem estão nesta tabela, que tem por finalidade: permitir, bloquear, negar e realizar logs.<br>
Ela é constituída por três cadeias: **INPUT, FORWARD e OUTPUT**.
Caso não seja informado a tabela (opção *-t* do comando `iptables`), a tabela *filter* sempre será a padrão.

> O iptables trabalha lendo as regras de cima para baixo. A posição/ordem das regras tem total importância.

#### Cadeia INPUT

As regras aplicadas, na cadeia INPUT, são referentes aos pacotes que chegam ao servidor (lado externo):
```shell
iptables -I INPUT -i enp0s9 -s 192.168.150.230 -p tcp --dport 7654 -j ACCEPT
iptables -I INPUT -i enp0s9 -s 192.168.150.230 -p icmp -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -j LOG --log-prefix "*** DROP INPUT ***" --log-level 4
iptables -P INPUT DROP
```

Descrevendo as regras aplicadas acima:
1. Liberando comunicação através da interface enp0s9, com origem da VM03, utilizando o protocolo TCP e a porta 7654;
2. Liberando comunicação através da interface enp0s9, com origem da VM03, utilizando o protocolo ICMP;
3. Liberando comunicação interna (*loopback*);
4. Liberando comunicações que já foram estabelecidas e relatadas, ou seja, comunicações baseadas em estado da conexão [(Stateful Firewall)](https://en.wikipedia.org/wiki/Stateful_firewall);
5. Realizando o log, apenas para o prefixo especificado ("*** DROP INPUT ***"), de todos os pacotes bloqueados;
6. Aplicando a política (*policy*) padrão para a cadeia. O alvo dessa política é DROP, bloqueando/descartando todos os pacotes que **NÃO** têm regras definidas.

#### Cadeia FORWARD

As regras definidas aqui são para os pacotes que não estão destinados ao servidor/firewall, porém ele “conhece” o destino e os encaminha (entra e sai):
```shell
iptables -I FORWARD -i enp0s9 -s 192.168.150.230 -o enp0s3 -p tcp --dport 80 -j ACCEPT
iptables -I FORWARD -i enp0s9 -s 192.168.150.230 -o enp0s3 -p tcp --dport 443 -j ACCEPT
iptables -I FORWARD -i enp0s9 -s 192.168.150.230 -o enp0s3 -p udp --dport 53 -j ACCEPT
iptables -I FORWARD -i enp0s8 -s 192.168.100.135 -o enp0s10 -d 192.168.200.30 -p icmp -j ACCEPT
iptables -I FORWARD -i enp0s8 -s 192.168.100.135 -o enp0s3 -p icmp -j ACCEPT
iptables -I FORWARD -i enp0s8 -s 192.168.100.135 -o enp0s3 -p udp --dport 53 -j ACCEPT
iptables -I FORWARD -i enp0s10 -s 192.168.200.30 -o enp0s8 -d 192.168.100.135 -p icmp -j ACCEPT
iptables -I FORWARD -i enp0s10 -s 192.168.200.30 -o enp0s3 -d 8.8.8.8 -p icmp -j ACCEPT
iptables -I FORWARD -i enp0s10 -s 192.168.200.30 -o enp0s3 -d 8.8.4.4 -p icmp -j ACCEPT
iptables -I FORWARD -i enp0s10 -s 192.168.200.30 -o enp0s3 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -j LOG --log-prefix "*** DROP FORWARD ***" --log-level 4
iptables -P FORWARD DROP
```

Descrição das regras:
1. Encaminhar os pacotes com origem na VM03 e destino a interface WAN da VM01, utilizando o protocolo HTTP; 
2. Encaminhar os pacotes com origem na VM03 e destino a interface WAN da VM01, utilizando o protocolo HTTPS;
3. Encaminhar os pacotes com origem na VM03 e destino a interface WAN da VM01, utilizando o sistema de resolução de nomes (DNS), através do protocolo UDP;
4. Pacotes com origem na VM02 serão encaminhados para VM04, utilizando o protocolo ICMP;
5. Pacotes ICMP da VM02 estão liberados para consulta externa;
6. Liberando consultas DNS para VM02;
7. Liberando comunicação ICMP da VM04 para VM02;
8. Liberando a VM04 para realizar consulta ICMP, apenas para o DNS primário do Google;
9. Liberando a VM04 para realizar consulta ICMP, apenas para o DNS secundário do Google;
10. Liberando consultas DNS para VM04;
11. Liberando comunicações que já foram estabelecidas e relatadas;
12. Realizando o log apenas para o prefixo especificado;
13. Aplicando a política padrão.

#### Cadeia OUTPUT

As regras para esta cadeia, são referentes aos pacotes gerados pelo servidor (lado interno):
```shell
iptables -I OUTPUT -o enp0s3 -p tcp --dport 80 -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p tcp --dport 443 -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p udp --dport 53 -j ACCEPT
iptables -I OUTPUT -d 192.168.150.230 -o enp0s9 -p tcp --dport 7654 -j ACCEPT
iptables -I OUTPUT -d 192.168.150.230 -o enp0s9 -p icmp -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -j LOG --log-prefix "*** DROP OUTPUT ***" --log-level 4
iptables -P OUTPUT DROP
```

Onde:
1. Liberando acesso externo, utilizando o protocolo HTTP;
2. Liberando acesso externo, utilizando o protocolo HTTPS;
3. Liberando consultas DNS;
4. Liberando acesso, através da interface enp0s9, com destino a VM03, utilizando o protocolo TCP e a porta 7654;
5. Liberando comunicação, através da interface enp0s9, com destino a VM03, utilizando o protocolo ICMP;
6. Liberando a interface loopback;
7. Liberando comunicações que já foram estabelecidas e relatadas;
8. Realizando o log apenas para o prefixo especificado;
9. Aplicando a política padrão.

#### LOG do iptables

Como boa prática e organização dos arquivos de log, é interessante direcionar os logs do iptables para um arquivo separado. Por padrão, o iptables envia suas mensagens para o buffer de mensagens do kernel, que popula o arquivo */var/log/syslog*.

Para evitar isso, precisamos configurar o arquivo *50-default.conf* no diretório */etc/rsyslog.d*, adicionando a seguinte linha:
```shell
vi /etc/rsyslog.d/50-default.conf
kern.warning    -/var/log/iptables.log
```

Com isso, o arquivo *iptables.log* pode ser consultado apenas para visualizar as informações referente aos prefixos definidos nas regras.

Como o nível de log, na regra do iptables, foi definido como alerta [(--log-level 4)](https://man7.org/linux/man-pages/man8/iptables-extensions.8.html) para os prefixos DROP, a prioridade também precisa ser alerta [(kern.warning)](https://man7.org/linux/man-pages/man5/rsyslog.conf.5.html).

Após editar o arquivo, precisamos reiniciar o serviço do rsyslog:
```shell
systemctl restart rsyslog.service
```

#### Salvando as regras

Até o momento, todas as regras que adicionamos estão guardadas em memória.
Se reiniciarmos o servidor, perderemos todas as regras. Evitamos tal situação, instalando o seguinte pacote:
```shell
apt-get update
apt-get install iptables-persistent
```

Ao finalizar a instalação, as regras serão salvas, de forma persistente, no arquivo */etc/iptables/rules.v4*.
Se precisarmos adicionar/deletar/editar alguma regra, salvamos com:
```shell
netfilter-persistent save
```

Opcionalmente, podemos salvar as regras, se quisermos direcionar para algum diretório específico, com o comando:
```shell
iptables-save > /diretorio/arquivo.regras
```

Ou podemos restaurar regras salvas com:
```shell
iptables-restore < /diretorio/arquivo.regras
```

> Durante o boot do sistema operacional, a leitura das regras será realizada, apenas, se forem salvas no arquivo */etc/iptables/rules.v4*.

### VM02

Configuração:
* Ubuntu Server: 20.04.5 LTS;
* Kernel: 5.4.0-137;
* CPU: 1;
* Memória RAM: 512 MB;
* Disco HD de 20 GB particionado, onde:
* Partição 1º, com 10 GB, destinada ao “/” com sistema de arquivo ext4;
* Partição 2°, com 10 GB, destinada ao uso de LVM (*Logical Volume Manager*), onde será criado um grupo de volume *vg0*, que terá dois volumes lógicos:
    * lv0-var: Com 9 GB destinada ao */var*, com sistema de arquivo ext4;
    * lv1-swap: Com 1 GB destinada ao *swap*, com sistema de arquivo Swap.
* Placa de rede Interna *netA*, com endereço IP estático: 192.168.100.135/24;

Configurando a interface de rede:
```shell
vi /etc/netplan/00-installer-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses: [192.168.100.135/24]
      gateway4: 192.168.100.1
      nameservers:
        addresses: [192.168.13.1, 8.8.8.8]
```

Criando um usuário, adm_lab, com acesso administrativo:
```shell
useradd -m -s /bin/shell adm_lab
usermod -aG sudo adm_lab
passwd adm_lab
```

Bloqueando o usuário root:
```shell
passwd -l root
usermod -p '!' root
```

Alterando a porta padrão, bloqueando acesso com usuário root, e reiniciando o serviço SSH:
```shell
vi /etc/ssh/sshd_config
Port 7654
PermitRootLogin no

systemctl restart sshd.service
```

Aplicando regras para o iptables:
```shell
iptables -I INPUT -i enp0s3 -s 192.168.200.30 -p icmp -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

iptables -I OUTPUT -o enp0s3 -d 192.168.200.30 -p icmp -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p icmp -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP
```

Salvando as regras:
```shell
apt-get update
apt-get install iptables-persistent
netfilter-persistent save
```

### VM03

Configuração:
* Ubuntu Server: 20.04.5 LTS;
* Kernel: 5.4.0-137;
* CPU: 1;
* Memória RAM: 512 MB;
* Disco HD de 20 GB particionado, onde:
* Partição 1º, com 10 GB, destinada ao “/” com sistema de arquivo ext4;
* Partição 2°, com 10 GB, destinada ao uso de LVM (*Logical Volume Manager*), onde será criado um grupo de volume *vg0*, que terá dois volumes lógicos:
    * lv0-var: Com 9 GB destinada ao */var*, com sistema de arquivo ext4;
    * lv1-swap: Com 1 GB destinada ao *swap*, com sistema de arquivo Swap.
* Placa de rede Interna *netB*, com endereço IP estático: 192.168.150.230/24;

Configurando a interface de rede:
```shell
vi /etc/netplan/00-installer-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses: [192.168.150.230/24]
      gateway4: 192.168.150.1
      nameservers:
        addresses: [192.168.13.1, 8.8.8.8]
```

Criando um usuário, adm_lab, com acesso administrativo:
```shell
useradd -m -s /bin/shell adm_lab
usermod -aG sudo adm_lab
passwd adm_lab
```

Bloqueando o usuário root:
```shell
passwd -l root
usermod -p '!' root
```

Alterando a porta padrão, bloqueando acesso com usuário root, e reiniciando o serviço SSH:
```shell
vi /etc/ssh/sshd_config
Port 7654
PermitRootLogin no

systemctl restart sshd.service
```

Aplicando regras para o iptables:
```shell
iptables -I INPUT -i enp0s3 -s 192.168.150.1 -p tcp --dport 7654 -j ACCEPT
iptables -I INPUT -i enp0s3 -s 192.168.150.1 -p icmp -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

iptables -I OUTPUT -o enp0s3 -d 192.168.13.140 -p tcp --dport 7654 -j ACCEPT
iptables -I OUTPUT -o enp0s3 -d 192.168.13.140 -p icmp -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p tcp --dport 80 -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p tcp --dport 443 -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP
```

Salvando as regras:
```shell
apt-get update
apt-get install iptables-persistent
netfilter-persistent save
```

### VM04

Configuração:
* Ubuntu Server: 20.04.5 LTS;
* Kernel: 5.4.0-137;
* CPU: 1;
* Memória RAM: 512 MB;
* Disco HD de 20 GB particionado, onde:
* Partição 1º, com 10 GB, destinada ao “/” com sistema de arquivo ext4;
* Partição 2°, com 10 GB, destinada ao uso de LVM (*Logical Volume Manager*), onde será criado um grupo de volume *vg0*, que terá dois volumes lógicos:
    * lv0-var: Com 9 GB destinada ao */var*, com sistema de arquivo ext4;
    * lv1-swap: Com 1 GB destinada ao *swap*, com sistema de arquivo Swap.
* Placa de rede Interna *netC*, com endereço IP estático: 192.168.200.30/24;

Configurando a interface de rede:
```shell
vi /etc/netplan/00-installer-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses: [192.168.200.30/24]
      gateway4: 192.168.200.1
      nameservers:
        addresses: [192.168.13.1, 8.8.8.8]
```

Criando um usuário, adm_lab, com acesso administrativo:
```shell
useradd -m -s /bin/shell adm_lab
usermod -aG sudo adm_lab
passwd adm_lab
```

Bloqueando o usuário root:
```shell
passwd -l root 
usermod -p '!' root
```

Alterando a porta padrão, bloqueando acesso com usuário root, e reiniciando o serviço SSH:
```shell
vi /etc/ssh/sshd_config
Port 7654
PermitRootLogin no

systemctl restart sshd.service
```

Aplicando regras para o iptables:
```shell
iptables -I INPUT -i enp0s3 -s 192.168.100.135 -p icmp -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

iptables -I OUTPUT -o enp0s3 -d 192.168.100.135 -p icmp -j ACCEPT
iptables -I OUTPUT -o enp0s3 -d 8.8.8.8 -p icmp -j ACCEPT
iptables -I OUTPUT -o enp0s3 -d 8.8.4.4 -p icmp -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP
```

Salvando as regras:
```shell
apt-get update
apt-get install iptables-persistent
netfilter-persistent save
```

<br>

## Cenário 2 - Rocky Linux

O [Rocky](https://rockylinux.org/pt_BR/about/) Linux é um projeto idealizado pelo mesmo fundador do CentOS, em detrimento da decisão da Red Hat, em descontinuar o desenvolvimento do CentOS.

Meu objetivo em reproduzir o mesmo laboratório, com outra distribuição, foi mostrar pequenas nuances que diferem ambos, porém obtendo o mesmo resultado.

### Configurando as VMs

Todas as configurações seguem, exatamente, o mesmo padrão conforme o tópico **Configuração das máquinas virtuais** deste documento, exceto as configurações das interfaces de rede e:
* Rocky Linux: 9.1 Blue Onyx;
* Kernel: 5.14.0-162.

### Configurando as interfaces de rede

### VM01 - Roteador e Firewall

Antes da versão Rocky Linux 9, as [configurações de rede](https://docs.rockylinux.org/guides/network/basic_network_configuration/) eram realizadas em um arquivo *ifcfg*, dentro do diretório */etc/sysconfig/network-scripts/*.

Atualmente o gerenciamento de rede é feito pelo serviço [NetworkManager](https://docs.rockylinux.org/gemstones/RL9_network_manager/), que já está instalado por padrão.
Para este laboratório, fiz uso do utilitário [nmcli](https://docs.rockylinux.org/gemstones/RL9_network_manager/#nmcli-command-recommended), que realiza a interação com o NetworkManager para configurar as interfaces:

WAN:<br>
(**nmcli>** é o prompt)

```shell
nmcli connection down enp0s3
nmcli connection edit enp0s3
 
nmcli> set ipv4.method manual
nmcli> set ipv4.addresses 192.168.13.140/24
nmcli> set ipv4.gateway 192.168.13.1
nmcli> set ipv4.dns 192.168.13.1, 8.8.8.8
nmcli> save
nmcli> quit

nmcli connection up enp0s3
```

Rede A:
```shell
nmcli connection down enp0s8
nmcli connection edit enp0s8

nmcli> set ipv4.method manual
nmcli> set ipv4.addresses 192.168.100.1/24
nmcli> save
nmcli> quit

nmcli connection up enp0s8
```

Rede B:
```shell
nmcli connection down enp0s9
nmcli connection edit enp0s9

nmcli> set ipv4.method manual
nmcli> set ipv4.addresses 192.168.150.1/24
nmcli> save
nmcli> quit

nmcli connection up enp0s9
```

Rede C:
```shell
nmcli connection down enp0s10
nmcli connection edit enp0s10

nmcli> set ipv4.method manual
nmcli> set ipv4.addresses 192.168.200.1/24
nmcli> save
nmcli> quit

nmcli connection up enp0s10
```

### VM02
```shell
nmcli connection down enp0s3
nmcli connection edit enp0s3
 
nmcli> set ipv4.method manual
nmcli> set ipv4.addresses 192.168.100.135/24
nmcli> set ipv4.gateway 192.168.100.1
nmcli> set ipv4.dns 192.168.13.1, 8.8.8.8
nmcli> save
nmcli> quit

nmcli connection up enp0s3
```

### VM03
```shell
nmcli connection down enp0s3
nmcli connection edit enp0s3
 
nmcli> set ipv4.method manual
nmcli> set ipv4.addresses 192.168.150.230/24
nmcli> set ipv4.gateway 192.168.150.1
nmcli> set ipv4.dns 192.168.13.1, 8.8.8.8
nmcli> save
nmcli> quit

nmcli connection up enp0s3
```

### VM04
```shell
nmcli connection down enp0s3
nmcli connection edit enp0s3
 
nmcli> set ipv4.method manual
nmcli> set ipv4.addresses 192.168.200.30/24
nmcli> set ipv4.gateway 192.168.200.1
nmcli> set ipv4.dns 192.168.13.1, 8.8.8.8
nmcli> save
nmcli> quit

nmcli connection up enp0s3
```

### SSH e SELinux

>**Configuração utilizada em todas as VMs.**

Alterando a porta padrão e bloqueando acesso com usuário root:
```shell
vi /etc/ssh/sshd_config
Port 7654
PermitRootLogin no
```

Após configurar o arquivo sshd_config, precisamos informar ao [SELinux](https://docs.rockylinux.org/guides/security/learning_selinux/) (que não permite a conexão via SSH por outra porta que não seja a padrão (**22**)) permitir que a “nova porta" seja utilizada.
Precisamos instalar o utilitário [semanage](https://man7.org/linux/man-pages/man8/semanage.8.html) (caso não venha instalado):
```shell
dnf install policycoreutils-python-utils
```

Em seguida executamos o comando:
```shell
semanage port -a -t ssh_port_t -p tcp 7654
```

Ao final, reiniciamos o serviço:
```shell
systemctl restart sshd.service
```

### Firewalld e iptables

>**Configuração utilizada em todas as VMs.**

Outra mudança que o Rocky Linux 9 trouxe foi a adoção, por padrão, do [firewalld](https://docs.rockylinux.org/en/guides/security/firewalld-beginners/) como sistema de firewall.
Porém, nosso laboratório foi implementado com iptables e, apesar de ter se tornado [obsoleto](https://docs.rockylinux.org/en/guides/security/enabling_iptables_firewall/), vamos utilizá-lo.

> O procedimento a seguir é desencorajado, e será utilizado apenas para fins de testes e laboratório.

Parando o serviço do firewalld:
```shell
systemctl stop firewalld.service
```

Desabilitando na inicialização:
```shell
systemctl disable firewalld.service
```

Mascarando o serviço:
```shell
systemctl mask firewalld.service
```

Após este procedimento, instalaremos o iptables:
```shell
dnf install iptables-services iptables-utils
```

Quando instalamos um pacote em distribuições baseadas em RHEL (*Red Hat Enterprise Linux*), e o mesmo possui algum serviço que rodará como *daemon* (processos que são executados em segundo plano), eles não iniciam e não habilitam a inicialização automática, após um boot ou reboot.

Como o comando a seguir, faremos as duas coisas:
```shell
systemctl enable --now iptables.service
```

Todas as regras que serão utilizadas, permanecem as mesmas, conforme o tópico **Aplicando Regras**, para todas as VMs, **exceto a definição do arquivo de log**.

Para salvar as regras de forma permanente, utilizamos o comando:
```shell
service iptables save
```

#### LOG do iptables

O aquivo de log padrão do Rocky Linux, fica localizado em */var/log/messages*.
Precisamos editar o arquivo de configuração */etc/rsyslog.conf*, e adicionar a seguinte linha:
```shell
vi /etc/rsyslog.conf
kern.warning    -/var/log/iptables.log
```

Ao final, reiniciamos o serviço:
```shell
systemctl restart rsyslog.service
```
