# Homelab

Este repositório contém a descrição e o código usados ​​para construir meu homelab doméstico.

-   [Resumo](#)
-   [Hardware](#)
-   [Rede](#)
-   [Programas](#)
-   [Instalação](#)
-   [Configuração de laboratório anterior](#)

## Resumo

A configuração atual usa 1 thin client como servidor, um Roteador TP-Link WR840N (V2) e um contêiner fazendo DNS sobre HTTPS usando [Pi-Hole](https://github.com/pi-hole/pi-hole) . Além disso, tenho várias VMs para teste e serviços internos.

Esta é minha primeira vez criando algo deste tipo. Os custos operacionais para criação deste projeto eram altos, sendo assim optei por utilizar equipamentos antigos que eu já tinha para reduzir meus gastos.

Este repositório e implementação foram originalmente inspirados no [Homelab de mbrancato](https://github.com/mbrancato/homelab) . Ao contrário do seguimento de *mbrancato*, estou iniciando com o Proxmox, a princípio é esta é a ferramenta que eu estou estudando e entendendo como funciona.
# Continua
## [](https://github.com/mbrancato/homelab#hardware)Hardware

**Servidores**

-   3x HP DL360e Gen8
    -   2x Intel Xeon E5-2407 (8 núcleos @ 2,2 GHz)
    -   2 fontes de alimentação de 750 W
    -   40 GB de memória
    -   4 placas de rede gigabit
    -   Adaptador Emulex / HP NC552SFP 8x PCIe 10Gb 2 portas SFP +
    -   Unidade Crucial MX500 250 GB M.2 SSD SATA
    -   1x Dual M.2 para adaptador PCIe - NVMe e SATA (somente alimentação)

O sistema operacional é instalado no SSD M.2 SATA. Originalmente, eu tinha isso configurado no controlador HP Smart Array B120i, mas percebi que depois de algum tempo, o disco seria perdido durante as reinicializações. Foi simplesmente consertado recriando o "array" RAID-0 de unidade única. Todo o crédito vai para [este post](https://community.hpe.com/t5/proliant-servers-netservers/dynamic-smart-array-loses-logical-drives-g8-b120i/m-p/7048172/highlight/true#M22095) onde aprendi que poderia usar o modo legado e trocar para o segundo controlador para inicializar diretamente do disco M.2 SATA. Ele é conectado a uma porta SATA que normalmente é projetada para executar a unidade de CDROM no DL360e.

**Rede**

-   HP ProCurve J9021A 2810-24G
    -   Versão N.11.78
    -   24 portas gigabit
-   2x Ubiquiti Networks UniFI UAP-AC-PRO-US
    -   802.11 a / b / g / n / ac 2,4 Ghz / 5 GHz
    -   3x3 MIMO
    -   PoE Powered (802.3af / 803.2at)

Em vez de um switch caro de 10 Gb, meu plano era criar uma pequena rede mesh. Na verdade, é uma pequena rede de topologia em anel, mas é tão pequena que também é uma malha. Se fosse estendido para incluir um quarto servidor, seria um anel.

[![](https://github.com/mbrancato/homelab/raw/master/images/network_layout.png)](https://github.com/mbrancato/homelab/blob/master/images/network_layout.png)

As interfaces de rede 4x 1 Gb em cada servidor são vinculadas usando [LACP (802.3ad)](https://en.wikipedia.org/wiki/Link_aggregation) com o switch. Existem pontes de software configuradas em cada nó que também reconhece VLAN, fazendo todos os links entre o nó do switch e os troncos de VLAN do nó-nó. Spanning Tree está habilitado, o que faz com que dois dos troncos LACP de 4 Gb sejam desligados em operação normal.

Os dois APs cobrem minha casa de 2 andares com um poço no porão com uma unidade montada no teto do segundo andar e outra montada no teto no porão.

**Armazenamento conectado à rede**

-   Servidor Supermicro 2U
    -   FreeNAS 11.1
    -   Chassi SC826 2U
    -   Placa-mãe X8DTN +
    -   Backplane SAS-826A
    -   2 fontes de alimentação de 800 W
    -   2x CPUs Intel Xeon L5620 (8 núcleos @ 2,13 GHz)
    -   24 GB de memória
    -   32 GB M.2 SATA SSD (unidade de inicialização)
    -   2 unidades Seagate ST6000NM0034 6 TB 7200 RPM SAS em espelho ZFS

Este sistema é meu NAS doméstico e anteriormente hospedava todas as imagens VM e um compartilhamento de arquivos SMB usado para armazenar fotos e documentos. Ele ainda é usado para SMB, mas será removido em algum ponto.

**UPS**

-   CyberPower OR2200PFCRT2U
    -   2000 VA
    -   1540 W
    -   Correção de fator de potência ativa
    -   2 × NEMA 5-20R
    -   6 × NEMA 5-15R

## [](https://github.com/mbrancato/homelab#network)Rede

A rede é composta por várias VLANs.

-   VLAN 1 - INÍCIO - Não utilizado / gerenciamento
-   VLAN 4 - PESSOAS - Convidados sem fio
-   VLAN 10 - GERAL - LAN de uso geral
-   VLAN 20 - ARMAZENAMENTO - Tráfego SAN / Ceph (não utilizado)
-   VLAN 30 - PVECLUSTER - Proxmox VE / corosync (não utilizado)
-   VLAN 60 - CÂMERAS - câmeras IP (não utilizadas)
-   VLAN 70 - EDGE - borda da Internet (não utilizada)
-   VLAN 90 - DISPOSITIVOS - Dispositivos / IoT

Inicialmente, configurei uma rede de armazenamento dedicada para minha configuração anterior com Proxmox. No entanto, com o Kubernetes, aplainei a rede e juntei minhas interfaces de rede de host de 10 Gb à árvore de abrangência. Isso permite que eles se comuniquem por meio de links de 10 Gb sem a necessidade de várias pontes de host para VLANs dedicadas entre nós.

## [](https://github.com/mbrancato/homelab#software)Programas

Essa nova configuração é [hiperconvergente](https://en.wikipedia.org/wiki/Hyper-converged_infrastructure) com armazenamento, rede e computação, tudo gerenciado e implantado no Kubernetes.

-   Debian 10 Buster
    -   [Kubernetes](https://kubernetes.io/) 1.18
        -   [Rook Ceph](https://rook.io/)
        -   [KubeVirt](https://kubevirt.io/)
        -   [MetalLB](https://metallb.universe.tf/)
        -   [Multus CNI](https://github.com/intel/multus-cni)
        -   [Chita](https://www.projectcalico.org/)
        -   [FluxCD](https://fluxcd.io/)

## [](https://github.com/mbrancato/homelab#installation)Instalação

A implantação inicial dos servidores bare metal e software subjacente é feita usando vários [manuais da Ansible](https://github.com/mbrancato/homelab/blob/master/playbooks) . A inicialização do cluster principal é realizada usando a placa HP iLO para inicializar e automatizar a instalação do Debian GNU / Linux. Depois que o cluster estiver instalado e funcionando, a implantação no cluster pode acontecer _principalmente_ por meio de implantação contínua (CD). Para CD, o Flux instalará os manifestos e gráficos como configuração de cluster declarativa em [clusters / homelab /](https://github.com/mbrancato/homelab/blob/master/clusters/homelab) .

MetalLB e Flux requerem algumas etapas extras para a instalação inicial.

## [](https://github.com/mbrancato/homelab#previous-lab-setup)Configuração de laboratório anterior

A renovação atual do meu laboratório é minha quarta grande revisão do meu laboratório doméstico. Minha configuração anterior (versão 3.x) consistia no seguinte hardware e software:

**Servidor de Máquina Virtual**

-   Dell PowerEdge R900
    -   XenServer 6.x
    -   4x CPUs Intel Xeon E7330 (16 núcleos @ 2,4 GHz)
    -   24 GB de memória
    -   2 drives SAS de 73 GB e 10k RPM em RAID 1

Esta foi uma migração natural para mim, pois eu estava usando o Xen em meu laboratório anterior. O uso do XenServer trazia algumas desvantagens, incluindo a necessidade de executar o aplicativo de gerenciamento XenCenter em uma VM do Windows. Este sistema tinha várias placas de rede gigabit, mas apenas uma foi usada.

**Armazenamento conectado à rede**

-   Chassi Supermicro SC826 2U com backplane SAS-826A
    -   FreeNAS 11.1
    -   2x CPUs Intel Xeon L5620 (8 núcleos @ 2,13 GHz)
    -   24 GB de memória
    -   32 GB M.2 SATA SSD (unidade de inicialização)
    -   Backplane SAS-826A
    -   2 unidades Seagate ST6000NM0034 6 TB 7200 RPM SAS em espelho ZFS

**Gateway de rede**

-   INFOBLOX-550-A
    -   Sophos UTM 9.6
    -   Intel Celeron a 2,93 GHz
    -   2 GB de memória
    -   SanDisk SDSSDA120G 120GB SSD SATA drive

Comprei isso por US $ 25 no eBay há alguns anos e decidi usá-lo como meu gateway. É apenas uma pequena placa-mãe Supermicro, mas carece de coisas como uma porta de saída de vídeo, tornando-a difícil de usar. Havia uma senha no BIOS que está embutida na ROM e não pode ser redefinida. A unidade SSD foi montada com fita dupla-face dentro do chassi.

**Switch de rede**

-   HP ProCurve J9021A 2810-24G
    -   24 portas gigabit

Este switch oferece suporte a atribuições de VLAN, SSH e alguns outros recursos. Pode ficar no novo laboratório por um tempo.

**Pontos de acesso sem fio**

-   2x Ubiquiti Networks UniFI UAP
    -   802.11 b / g / n 2,4 GHz
    -   2x2 MIMO
    -   PoE alimentado (UniFi 24V)
