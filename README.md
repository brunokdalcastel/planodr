
Implementação de Disaster Recovery (DR) Híbrido com Azure Site Recovery

![Badge de Status](https://img.shields.io/badge/status-Conclu%C3%ADdo-brightgreen)
![Badge de Tecnologia](https://img.shields.io/badge/Tecnologia-Azure%20Site%20Recovery-blue)
![Badge de Arquitetura](https://img.shields.io/badge/Arquitetura-H%C3%ADbrida%20(Multi%20Site)-orange)
![Badge de Autor](https://img.shields.io/badge/Autor-Bruno%20Castel-blueviolet)

## 1. O Desafio: Risco Total de Continuidade de Negócios

Este projeto foi iniciado para endereçar uma vulnerabilidade crítica de negócios: a **ausência completa de um plano de Disaster Recovery (DR)** para a infraestrutura de servidores on-premises (Matriz).

**Estado Inicial (Antes do Projeto):**

* **Infraestrutura 100% On-Premises:** Todos os serviços críticos (AD, Banco de Dados, Aplicações, Arquivos) rodavam em servidores **Microsoft Hyper-V** locais.
* **Ponto Único de Falha:** Um incidente grave (incêndio, falha de hardware em cascata, ransomware) na Matriz resultaria em **paralisação total das operações** por tempo indeterminado.
* **RTO/RPO indefinidos:** Não havia métricas de Tempo de Retorno de Operação (RTO) ou Ponto de Recuperação de Operação (RPO), significando potencial perda permanente de dados e dias ou semanas de downtime.

O objetivo era claro: **projetar e implementar uma solução de DR robusta, financeiramente viável e de rápida acionamento**, garantindo a resiliência dos serviços essenciais da empresa.

---

## 2. A Arquitetura da Solução Híbrida (Multi-Site)

A solução implementada foi uma **arquitetura híbrida** que utiliza o **Microsoft Azure** como o site de *failover* principal e uma **Filial Secundária** como suporte de identidade (Active Directory Secundário), com conectividade de rede totalmente redundante.

### Diagrama da Arquitetura

```mermaid
graph TD;
    %% Definição dos "Contêineres" (Sites)
    subgraph "Ambiente On-Premises<br>(Matriz)"
        direction LR
        VM_AD1["VM: AD-01"]
        ONPREM_GW("Appliance Firewall<br>WatchGuard Matriz")
        VM_DB["VM: DB-01"]
        VM_APP["VM: APP-01"]
        VM_FS["VM: FS-01 (File Server)"]
    end

    subgraph "Microsoft Azure (Site de DR)"
        direction LR
        ASR["Recovery Services Vault"]
        VNET_DR["VNet de DR"]
        AZ_VPN_GW["Virtual Network Gateway<br>(VPN S2S/P2S)"]
        VM_AD1_REP("Replica: AD-01")
        VM_DB_REP("Replica: DB-01")
        VM_APP_REP("Replica: APP-01")
        VM_FS_REP("Replica: FS-01")
        
        ASR --> VNET_DR
        VNET_DR --- VM_AD1_REP & VM_DB_REP & VM_APP_REP & VM_FS_REP
        AZ_VPN_GW --> VNET_DR
    end

    subgraph "Filial Secundária"
        direction LR
        VM_AD2["VM: AD-02 (Secundário)"]
        EXT_GW("Appliance Firewall<br>WatchGuard Filial")
    end

    %% Definição das Conexões de Replicação
    VM_AD1 -- "Replicação ASR (via Internet)" --> ASR
    VM_DB -- "Replicação ASR (via Internet)" --> ASR
    VM_APP -- "Replicação ASR (via Internet)" --> ASR
    VM_FS -- "Replicação ASR (via Internet)" --> ASR

    VM_AD1 -. "Replicação AD" .-> VM_AD2
    
    %% Conexões VPN Site-to-Site
    ONPREM_GW -- "VPN Site-to-Site" --- AZ_VPN_GW
    EXT_GW -- "VPN Site-to-Site" --- AZ_VPN_GW 
    ONPREM_GW -- "VPN S2S (Matriz-Filial)" --- EXT_GW
    
    style ASR fill:#0078D4,stroke:#FFF,stroke-width:2px,color:#FFF
    style VNET_DR fill:#0078D4,stroke:#FFF,stroke-width:2px,color:#FFF
    style AZ_VPN_GW fill:#0078D4,stroke:#FFF,stroke-width:2px,color:#FFF
    style VM_AD1_REP fill:#BCE8FF,stroke:#0078D4,stroke-width:1px,color:#000
    style VM_DB_REP fill:#BCE8FF,stroke:#0078D4,stroke-width:1px,color:#000
    style VM_APP_REP fill:#BCE8FF,stroke:#0078D4,stroke-width:1px,color:#000
    style VM_FS_REP fill:#BCE8FF,stroke:#0078D4,stroke-width:1px,color:#000
````

**Componentes Principais da Arquitetura:**

1.  **Ambiente On-Premises (Matriz):**

      * Servidores host **Microsoft Hyper-V**.
      * VMs Críticas: `AD-01` (Active Directory), `DB-01` (Banco de Dados), `APP-01` (Aplicação), `FS-01` (File Server).
      * Gateway de VPN/Firewall: **Appliance Físico WatchGuard**, com VPN S2S para a Filial e para o Azure.

2.  **Filial Secundária (Suporte de Identidade):**

      * Provedor: **Infraestrutura Interna (Filial)**.
      * Serviços: Hospedagem de uma VM com **Active Directory Secundário (AD-02)**.
      * Conectividade: Uma **VPN Site-to-Site** persistente com a Matriz e uma **VPN Site-to-Site** secundária para o Azure.

3.  **Microsoft Azure (Site de Disaster Recovery):**

      * **Recovery Services Vault (ASR):** O "cérebro" da operação, orquestrando a replicação e o failover.
      * **Rede Virtual (VNet) de DR:** Uma rede virtual isolada, pré-configurada para receber as VMs em caso de desastre.
      * **Azure Virtual Network Gateway:** Componente central que encerra as VPNs Site-to-Site (da Matriz e da Filial) e Ponto-a-Site (para usuários).
      * **Contas de Armamento:** Recebem os dados replicados dos discos das VMs on-premises.

-----

## 3\. Stack de Tecnologias Utilizadas

  * **Plataforma de Nuvem (DR):**
      * Microsoft Azure
      * Azure Site Recovery (ASR)
      * Azure Virtual Network
      * Azure Virtual Network Gateway
      * Azure Storage Accounts
  * **Virtualização On-Premises (Matriz e Filial):**
      * Microsoft Hyper-V
  * **Sistemas e Serviços:**
      * Windows Server 2019
      * Active Directory Domain Services
      * Microsoft SQL Server 2017
      * Aplicação Interna de ERP (Baseada em IIS)
      * Serviços de Arquivo (File Server)
  * **Networking:**
      * VPN Site-to-Site (Topologia Hub-and-Spoke Híbrida)
      * VPN Ponto-a-Site (no Azure VNG)
      * Firewall: **Appliance Físico WatchGuard**

-----

## 4\. Metodologia de Implementação (Passo a Passo)

O projeto foi executado em 5 fases principais para garantir uma implementação segura e validada.

### Fase 1: Assessment e Design

1.  **Mapeamento de Dependências:** Análise de quais VMs dependiam de quais (ex: `APP-01` depende de `DB-01` e `AD-01`).
2.  **Definição de RTO/RPO:** As metas de negócio definidas foram **RTO \< 4 horas** e **RPO \< 1 hora**.
3.  **Design da VNet no Azure:** Desenho da sub-rede de DR, regras de NSG, planejamento de IP e provisionamento do **Virtual Network Gateway**.

### Fase 2: Configuração da Infraestrutura de Rede e Identidade

1.  **Provisionamento do AD Secundário:** Criação da VM `AD-02` na infraestrutura da Filial.
2.  **Configuração da VPN S2S (Matriz \<-\> Filial):** Estabelecimento do túnel VPN seguro entre a Matriz e a Filial usando os appliances WatchGuard para replicação do AD.
3.  **Configuração da VPN S2S (Matriz/Filial \<-\> Azure):** Estabelecimento de túneis VPN redundantes de ambos os sites on-premises para o Virtual Network Gateway no Azure.
4.  **Promoção do DC:** Promoção do `AD-02` como Controlador de Domínio secundário e validação da replicação do AD.

### Fase 3: Configuração do Azure Site Recovery (ASR)

1.  **Criação do Cofre:** Provisionamento do **"Recovery Services Vault"** no Azure.
2.  **Instalação de Agentes:** Instalação do servidor de configuração do ASR e agentes de mobilidade nas VMs **Hyper-V** da Matriz.
3.  **Habilitação da Replicação:** Início da replicação para as 4 VMs críticas (`AD-01`, `DB-01`, `APP-01`, `FS-01`), direcionando-as para a VNet de DR e conta de armazenamento corretas.

### Fase 4: Criação e Orquestração do Plano de Recuperação

1.  **Criação do Plano:** Criação de um "Plano de Recuperação" dentro do ASR.
2.  **Definição da Ordem de Boot:** Este é um passo crítico. A ordem de inicialização das VMs foi definida para respeitar as dependências de serviço:
      * **Grupo 1:** `AD-01` (Garantir que o DNS e a autenticação estejam online primeiro)
      * **Grupo 2:** `DB-01` (Banco de dados deve estar pronto para a aplicação)
      * **Grupo 3:** `APP-01` e `FS-01` (Serviços de aplicação e arquivos)

### Fase 5: Teste de Failover

1.  **Execução do Teste:** Utilização da função "Test Failover" do ASR, que cria as VMs em uma rede isolada no Azure **sem impactar a produção**.
2.  **Validação:** Acesso às VMs no Azure para validar:
      * Conectividade de rede.
      * Integridade dos dados (arquivos recentes estavam presentes?).
      * Acesso à aplicação (`APP-01` conseguia conectar no `DB-01`?).
      * Autenticação (`APP-01` conseguia falar com o `AD-01` replicado?).
3.  **Limpeza:** Conclusão do teste e remoção automática das VMs de teste pelo ASR.

-----

## 5\. Desafios Chave e Lições Aprendidas

### 1\. A Complexidade do Active Directory

  * **Desafio:** O AD é o "coração" da rede. Um failover incorreto de um Controlador de Domínio (DC) pode corromper todo o catálogo. A solução de ter um AD secundário *sempre ativo* na Filial foi complexa de configurar, mas crucial.
  * **Lição:** Nunca trate um DC como uma VM qualquer. O ASR tem mecanismos para lidar com DCs, mas a arquitetura de AD distribuído (Matriz + Filial) adicionou uma camada de complexidade que exigiu testes rigorosos de sincronismo.

### 2\. Orquestração é Tudo

  * **Desafio:** Na primeira tentativa de teste, as VMs de aplicação subiram antes do banco de dados, fazendo com que os serviços da aplicação falhassem.
  * **Lição:** A definição da ordem de inicialização no "Plano de Recuperação" não é opcional, é **fundamental**. O Grupo 1 (AD/DNS) e o Grupo 2 (Banco de Dados) devem estar 100% operacionais antes do Grupo 3 (Aplicação) tentar iniciar.

### 3\. Planejamento de Rede Pós-Failover

  * **Desafio:** Após o failover para o Azure, como os usuários se conectariam aos serviços? Os IPs das VMs mudaram para os da VNet do Azure.
  * **Lição:** Um plano de DR não termina quando as VMs ligam. É preciso ter um plano para **redirecionar o tráfego**. A solução de rede foi dupla:
    1.  A **VPN Site-to-Site (S2S)** já existente da Filial para o Azure permitiu que os usuários no escritório da Filial continuassem trabalhando, acessando os serviços agora na nuvem de forma transparente.
    2.  A ativação de um **Gateway de VPN Ponto-a-Site (P2S)** no Azure permitiu que usuários remotos (em casa) também se conectassem diretamente à VNet de DR.
        Isso, combinado com scripts para atualização de registros DNS, garantiu o acesso contínuo.

-----

## 6\. Impacto e Próximos Passos

**Resultado:** A empresa saiu de um estado de **risco total** para uma arquitetura resiliente com **RTO \< 4 horas e RPO \< 1 hora claramente definidos e testados**.

Este projeto prova a viabilidade de usar uma estratégia multi-site e híbrida (On-Premises Matriz/Filial + Azure) para criar uma solução de continuidade de negócios completa e robusta, protegendo os ativos mais críticos da organização.

-----

### Autor

  * **Bruno Castel**

<!-- end list -->

```
```
