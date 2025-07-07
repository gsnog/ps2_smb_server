# Servidor Samba para PS2 com OPL

## Descrição do Projeto

Este projeto consiste na configuração de um **servidor de arquivos Samba** em uma máquina virtual Linux, especificamente para compartilhar jogos de PlayStation 2 (PS2) via rede Ethernet. O objetivo principal é permitir que o console PS2, utilizando o **Open PS2 Loader (OPL)**, acesse e execute jogos armazenados na máquina virtual, oferecendo uma alternativa ao uso de CDs/DVDs ou HDDs/USB diretamente conectados ao console.

O setup inclui a configuração da rede em modo bridge para comunicação direta entre a VM e o PS2, ajustes no Samba para compatibilidade com o protocolo SMBv1 (necessário para o OPL), e orientações sobre a preparação dos arquivos de jogo (.ISO) e a configuração correspondente no OPL. Além disso, o projeto demonstra como monitorar o tráfego de rede usando o Wireshark para depuração.

## Pré-requisitos

Antes de iniciar, certifique-se de ter o seguinte:

-   Uma **máquina virtual (VM)** com uma distribuição Linux (ex: Ubuntu, Debian, Fedora).
-   Um **PlayStation 2 (PS2)** com o **Open PS2 Loader (OPL)** instalado e funcional (geralmente via Free MCBoot ou modchip).
-   Conhecimento básico de linha de comando Linux.
-   Um arquivo de imagem de jogo do PS2 no formato **.iso** para teste.

## Instalação e Uso

Siga os passos abaixo para configurar o servidor Samba na sua VM Linux e acessar os jogos pelo PS2.

### 1. Configuração da Máquina Virtual e Rede

1.  **Modo de Rede da VM:**
    * Desligue a VM.
    * Nas configurações da VM (VirtualBox, VMware, etc.), vá em **Rede** e configure o adaptador para **Adaptador em Bridge (Ponte)**.
    * Selecione a interface de rede física correta do seu computador host (Ethernet ou Wi-Fi).
    * Inicie a VM.

2.  **Configuração de IP Estático na VM:**
    É crucial que sua VM tenha um endereço IP fixo para que o PS2 possa encontrá-la consistentemente.
    * Verifique o nome da sua interface de rede (geralmente `enp0s3`, `eth0`):
        ```bash
        ip a
        ```
    * **Para sistemas com Netplan (Ubuntu 18.04+, Debian 10+):**
        * Edite o arquivo de configuração do Netplan (ex: `/etc/netplan/00-installer-config.yaml`):
            ```bash
            sudo nano /etc/netplan/00-installer-config.yaml
            ```
        * Modifique o conteúdo para:
            ```yaml
            network:
              ethernets:
                enp0s3: # Substitua pela sua interface de rede real
                  dhcp4: no
                  addresses: [192.168.1.150/24] # **ESCOLHA UM IP FIXO FORA DA FAIXA DHCP DO SEU ROTEADOR**
                  routes:
                    - to: default
                      via: 192.168.1.1 # Substitua pelo IP do seu roteador (gateway)
                  nameservers:
                    addresses: [192.168.1.1, 8.8.8.8] # Substitua pelos seus DNS (roteador e/ou DNS público)
              version: 2
            ```
        * Aplique as alterações:
            ```bash
            sudo netplan apply
            ```
    * **(Para outras distribuições ou métodos de configuração de IP estático, consulte `setup_commands.md`.)**

### 2. Instalação e Configuração do Samba

1.  **Atualizar e Instalar Samba:**
    ```bash
    sudo apt update
    sudo apt install samba samba-common-bin
    ```

2.  **Criar e Configurar a Pasta de Compartilhamento:**
    * Crie o diretório para os jogos:
        ```bash
        sudo mkdir -p /srv/ps2games
        ```
    * Defina permissões amplas para o acesso guest do Samba:
        ```bash
        sudo chmod -R 777 /srv/ps2games
        ```

3.  **Configurar o `smb.conf`:**
    * Faça um backup do arquivo original:
        ```bash
        sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
        ```
    * Edite o arquivo `smb.conf`:
        ```bash
        sudo nano /etc/samba/smb.conf
        ```
    * Na seção `[global]`, adicione as linhas para compatibilidade SMBv1 (essencial para OPL):
        ```ini
        [global]
        # ... outras configurações globais existentes ...
        server min protocol = NT1
        client min protocol = NT1
        client max protocol = SMB3
        ```
    * No final do arquivo, adicione a seção do seu compartilhamento:
        ```ini
        [PS2SMB] # Este é o nome do compartilhamento que o OPL verá
           path = /srv/ps2games
           comment = PS2 Games Share
           browseable = yes
           read only = no
           guest ok = yes
           force user = nobody
           force group = nogroup
        ```
    * Salve e feche o arquivo.

4.  **Reiniciar o Serviço Samba:**
    ```bash
    sudo systemctl restart smbd nmbd
    sudo systemctl enable smbd nmbd # Para iniciar automaticamente no boot
    ```

### 3. Configuração do Firewall (se aplicável)

Se você usa `ufw` ou `firewalld`, permita o tráfego Samba:

-   **Para UFW:**
    ```bash
    sudo ufw allow samba
    sudo ufw enable
    sudo ufw status
    ```
-   **Para Firewalld:**
    ```bash
    sudo firewall-cmd --permanent --add-service=samba
    sudo firewall-cmd --reload
    ```

### 4. Preparação dos Jogos

1.  **Formato:** Todos os jogos devem estar no formato **.iso**.
2.  **Nomenclatura:** Renomeie seus arquivos ISO para incluir o ID do jogo. Este é o formato mais confiável para o OPL.
    * **Formato:** `Nome do Jogo (Região-ID_do_Jogo).iso`
    * **Exemplo:** `Grand Theft Auto - San Andreas (NTSC-U SLUS-20946).iso`
    * O ID do jogo (ex: `SLUS-20946`, `SLES-XXXXX`) pode ser encontrado em sites como Redump.org.
3.  **Localização na Pasta de Compartilhamento:**
    * Mova seus arquivos ISO para dentro da subpasta **`DVD/`** ou **`CD/`** (a maioria dos jogos de PS2 são DVD) dentro do seu compartilhamento (`/srv/ps2games`).
    * Exemplo de estrutura:
        ```
        /srv/ps2games/
        ├── DVD/
        │   └── Grand Theft Auto - San Andreas (NTSC-U SLUS-20946).iso
        ├── APPS/
        ├── CFG/
        # ... outras pastas de OPL ...
        ```
    * Comandos de exemplo para mover um jogo para a pasta `DVD`:
        ```bash
        cd /srv/ps2games
        sudo mkdir -p DVD 
        sudo mv "Grand Theft Auto - San Andreas (NTSC-U SLUS-20946).iso" DVD/
        ```

### 5. Configuração do PS2 (OPL)

1.  **Inicie o OPL** no seu PS2.
2.  Vá em **Network Config** (Configurações de Rede).
3.  **Configure o Endereço IP do PS2:**
    * `IP Address Type`: Defina como `Static`.
    * `IP Address`: Escolha um IP livre na mesma sub-rede da VM (ex: `192.168.1.10`).
    * `Netmask`: `255.255.255.0`.
    * `Gateway`: O IP do seu roteador (ex: `192.168.1.1`).
    * `DNS Server`: O IP do seu roteador ou um DNS público (ex: `8.8.8.8`).
4.  **Configure o Servidor SMB (SMB Server):**
    * `IP Address Type`: `Static`.
    * `IP Address`: O IP estático da sua VM Linux (ex: `192.168.1.150`).
    * `Port`: `445`.
    * `Share Name`: `PS2SMB` (nome exato do compartilhamento que você definiu no `smb.conf`).
    * `User`: (Deixar em branco).
    * `Password`: (Deixar em branco).
5.  **Salve as Configurações** (`OK` e depois `Save Changes`).
6.  Volte ao menu principal do OPL e vá para **Settings**.
7.  Verifique/Configure:
    * `ETH device start mode`: `Auto`.
    * `SMB Games`: Se disponível, defina como `Auto`.
8.  **Salve as Configurações** novamente.
9.  Na tela **ETH Games** do OPL, pressione **START** ou **TRIÂNGULO** para forçar a atualização da lista de jogos. Se o jogo não aparecer, reinicie o PS2 completamente.

### 6. Monitoramento e Depuração

Para verificar o tráfego de rede e depurar problemas:

1.  **Instale o Wireshark** na sua VM Linux:
    ```bash
    sudo apt install wireshark
    sudo usermod -a -G wireshark $USER 
    ```
2.  Inicie o Wireshark, selecione sua interface de rede (`enp0s3`) e comece a capturar.
3.  Use o filtro `smb` para visualizar apenas o tráfego do Samba enquanto tenta acessar os jogos no PS2.

### 7. Divisão de Tarefas para Configuração de Servidor Samba para PS2 com OPL:

Arben Tafili: Configuração Inicial da Máquina Virtual e Rede.

Responsável por configurar a Máquina Virtual (VM) em modo "Adaptador em Bridge" para comunicação direta com o PS2.

Define um endereço IP estático para a VM, garantindo que o PS2 a encontre consistentemente na rede.

Giovana Nogueira: Instalação e Configuração do Servidor Samba.

Encarregada de instalar o Samba na VM.

Cria a pasta de compartilhamento (/srv/ps2games) e configura as permissões.

Edita o arquivo smb.conf para compatibilidade com SMBv1 (essencial para o OPL) e define o compartilhamento [PS2SMB].

Reinicia e habilita os serviços Samba.

Enzo Pavaneli: Preparação dos Jogos e Configuração do PS2 (OPL).

Prepara os jogos em formato .iso, renomeando-os com o ID do jogo e os organiza nas pastas DVD/ ou CD/ dentro do compartilhamento Samba.

Configura o Open PS2 Loader (OPL) no PS2, definindo o IP estático do console e os detalhes do servidor SMB, como o IP da VM e o nome do compartilhamento (PS2SMB).

Sofia Recreio: Configuração de Firewall e Monitoramento/Depuração.

Verifica e configura o Firewall (UFW ou Firewalld) na VM para permitir o tráfego do Samba.

Instala o Wireshark para monitorar o tráfego de rede e auxiliar na depuração de problemas, garantindo que a comunicação entre o PS2 e o servidor Samba esteja ocorrendo corretamente.


### 8. Video de Denonstração

Link: https://youtu.be/wiEo0yIC6C0
