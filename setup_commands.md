# Comandos de Configuração do Servidor Samba para PS2

Este documento detalha os comandos utilizados para configurar um servidor Samba em uma máquina virtual Linux, permitindo o compartilhamento de jogos com um PlayStation 2 via rede (utilizando o OPL - Open PS2 Loader).

## 1. Configuração da Máquina Virtual (Rede)

- Certificar-se de que a VM está configurada em modo **Adaptador em Bridge (Ponte)** nas configurações do virtualizador (VirtualBox, VMware, etc.).
- Anotar o endereço IP da VM. No terminal da VM:
  ```bash
  ip a

(Observação: O endereço IP da VM, por exemplo, 192.168.1.150, será usado nas configurações de rede do PS2.)

    Configurar IP estático na VM (exemplo usando Netplan - para Ubuntu/Debian mais recentes):

        Editar o arquivo de configuração do Netplan (o nome pode variar, como 00-installer-config.yaml):
        Bash

sudo nano /etc/netplan/00-installer-config.yaml

Adicionar ou modificar o conteúdo para configurar a interface (enp0s3 no exemplo) com IP estático:
YAML

network:
  ethernets:
    enp0s3: # Substitua pela sua interface de rede real (verificado com 'ip a')
      dhcp4: no # Desabilita DHCP para IPv4
      addresses: [199.168.1.150/24] # Altere para o IP estático desejado para a VM e a máscara de sub-rede
      routes:
        - to: default
          via: 192.168.1.1 # Altere para o IP do seu roteador (gateway)
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8] # Altere para seus servidores DNS (roteador e/ou DNS público)
  version: 2

Aplicar as mudanças do Netplan:
Bash

        sudo netplan apply

    (Se sua distribuição usa NetworkManager ou interfaces legacy, os comandos para IP estático seriam diferentes, conforme discutido anteriormente.)

2. Instalação e Configuração do Samba

    Atualizar a lista de pacotes:
    Bash

sudo apt update

Instalar os pacotes Samba e Samba Common Bin:
Bash

sudo apt install samba samba-common-bin # Para Debian/Ubuntu/Mint
# (Se for Fedora/CentOS/RHEL, use: sudo dnf install samba samba-client)

Criar o diretório para os jogos que será compartilhado:
Bash

sudo mkdir -p /srv/ps2games

Definir permissões de acesso completas para a pasta compartilhada. Isso garante que o usuário 'nobody' (usado pelo acesso guest do Samba) possa ler e listar o conteúdo:
Bash

sudo chmod -R 777 /srv/ps2games

Fazer um backup do arquivo de configuração original do Samba antes de modificá-lo:
Bash

sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak

Editar o arquivo de configuração principal do Samba para definir o compartilhamento e compatibilidade com PS2:
Bash

sudo nano /etc/samba/smb.conf

(Conteúdo do smb.conf para incluir ou modificar - copie a seção [global] e adicione a seção [PS2SMB] no final):
Ini, TOML

[global]
   workgroup = WORKGROUP
   server string = Samba Server
   log file = /var/log/samba/log.%m
   max log size = 1000
   logging = file
   panic action = /usr/share/samba/panic-action %d
   server role = standalone server
   obey pam restrictions = yes
   unix password sync = yes
   passwd program = /usr/bin/passwd %u
   passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
   pam password change = yes
   map to guest = Bad User
   usershare allow guests = yes
   # Adicionado para compatibilidade com PS2 (essencial para OPL)
   server min protocol = NT1
   client min protocol = NT1
   client max protocol = SMB3 # Permite que a VM use SMB3 com outros clientes

[PS2SMB] # Nome do compartilhamento que será visto pelo PS2/OPL
   path = /srv/ps2games # Caminho para a pasta de jogos na VM
   comment = PS2 Games Share
   browseable = yes
   read only = no # Permite leitura e escrita (geralmente só leitura é necessária para jogos)
   guest ok = yes # Permite acesso sem autenticação
   force user = nobody # Mapeia o usuário guest para 'nobody' no sistema Linux
   force group = nogroup # Mapeia o grupo guest para 'nogroup' no sistema Linux

Salvar o arquivo e sair do editor.

Reiniciar os serviços Samba para que as novas configurações sejam aplicadas:
Bash

sudo systemctl restart smbd nmbd

Opcional: Habilitar o Samba para iniciar automaticamente com o sistema:
Bash

    sudo systemctl enable smbd nmbd

3. Configuração do Firewall (se aplicável)

Se sua VM Linux tiver um firewall ativo, é necessário permitir o tráfego do Samba.

    Para UFW (Ubuntu/Debian/Mint):
    Bash

sudo ufw allow samba
sudo ufw enable # Habilita o firewall se ainda não estiver ativo
sudo ufw status # Verifica o status do firewall

Para Firewalld (Fedora/CentOS/RHEL):
Bash

    sudo firewall-cmd --permanent --add-service=samba
    sudo firewall-cmd --reload # Recarrega as regras do firewall
    sudo firewall-cmd --list-services # Lista os serviços permitidos

4. Teste de Acesso ao Compartilhamento (na VM Linux)

Verificar se o compartilhamento está acessível localmente como um guest:
Bash

smbclient //localhost/PS2SMB -U guest
# Quando solicitado por senha, apenas pressione Enter (já que é guest)
# No prompt 'smb: \>', digite 'ls' para listar o conteúdo da pasta compartilhada.
# Você deve ver os arquivos ISO dos seus jogos aqui.

5. Preparação dos Jogos e Estrutura de Pastas

Para que o OPL detecte os jogos via SMB, a organização e a nomenclatura são cruciais:

    As ISOs dos jogos de PS2 devem estar no formato .iso.

    Organização da pasta: É altamente recomendado que os jogos estejam dentro da subpasta DVD/ ou CD/ (a maioria dos jogos de PS2 é DVD) dentro do seu compartilhamento principal (/srv/ps2games).

    /srv/ps2games/
    ├── DVD/
    │   └── Nome Do Seu Jogo 1 (NTSC-U SLUS-XXXXX).iso
    │   └── Nome Do Seu Jogo 2 (PAL SCES-YYYYY).iso
    ├── CD/ # Se tiver jogos de CD, coloque aqui
    │   └── Nome Do Jogo CD (NTSC-U SLUS-ZZZZZ).iso
    ├── APPS/ # Pastas internas do OPL
    ├── ART/
    ├── CFG/
    ├── CHT/
    ├── LNG/
    ├── THM/
    └── VMC/

    Nomenclatura do arquivo ISO: Os nomes dos arquivos ISO devem incluir o ID do jogo para garantir a detecção e compatibilidade do OPL. O formato recomendado é:
    Nome do Jogo (Região-ID_do_Jogo).iso

        Exemplo: Grand Theft Auto - San Andreas (NTSC-U SLUS-20946).iso

        O ID do jogo (ex: SLUS-20946, SLES-XXXXX, SCES-YYYYY) pode ser encontrado no disco original do jogo ou em bancos de dados online como Redump.org.

6. Configuração do PS2 (OPL - Open PS2 Loader)

Configurar o OPL para acessar o servidor Samba:

    Iniciar o OPL no PlayStation 2.

    Navegar para Network Config (Configurações de Rede).

    Configurar Endereço IP do PS2:

        IP Address Type: Escolha Static (recomendado para consistência) ou DHCP.

            Se Static:

                IP Address: Ex: 192.168.1.10 (um IP livre na mesma sub-rede da VM)

                Netmask: 255.255.255.0

                Gateway: 192.168.1.1 (IP do seu roteador)

                DNS Server: 192.168.1.1 (ou 8.8.8.8 para Google DNS)

    Configurar Servidor SMB (SMB Server):

        IP Address Type: Static

        IP Address: 192.168.1.150 (o IP estático da sua máquina virtual Linux que está rodando o Samba)

        Port: 445 (porta padrão do SMB)

        Share Name: PS2SMB (o nome exato do compartilhamento definido no smb.conf)

        User: (Deixar em branco)

        Password: (Deixar em branco)

    Salvar as Configurações (geralmente com "OK" e depois "Save Changes").

    Voltar ao menu principal do OPL e ir para Settings.

    Verificar as seguintes opções:

        ETH device start mode: Definir como Auto ou Manual.

        SMB Games: Se disponível, definir como Auto.

    Salvar as Configurações novamente.

    Na tela ETH Games do OPL: Pressionar START ou TRIÂNGULO para forçar a atualização da lista de jogos após adicionar novos arquivos.

    Se os jogos não aparecerem, reiniciar o PS2 completamente.
