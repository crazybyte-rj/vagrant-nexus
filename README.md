# README

Pequeno laboratório para subir um repositório do Nexus para  atuar como proxy para o Docker Hub.

Foi utilizado o Vagrant com Virtualbox para subir a máquina virtual e foi criado um script para instalar e configurar o Nexus na mesma.

### **Procedimentos**

1. Instalação do Vagrant
   * Acesse [Download Vagrant](https://developer.hashicorp.com/vagrant/downloads?product_intent=vagrant) e escolha o seu Sistema operacional. Aqui vamos usar o ambiente **Linux** como base.
   * 
   * Para distribuições baseadas em **Debian/Ubuntu**, podemos configurar o repositório com os passos abaixo:

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```
   * Ou você pode baixar o binário da versão que desejar

   * Após instalar o vagrant, vamos instalar o plugin **vbguest** que traz algumas facilidades. Para isso, execute o comando abaixo:
     `vagrant plugin install vagrant-vbguest`

2. Inicializar nosso projeto
   *Crie uma pasta aonde ficarão os arquivos de configuração e inicialize o vagrant nesta pasta:
      * mkdir ~/lab-nexus
      * vagrant init
      (O comando acima apenas cria um arquivo **Vagrantfile** padrão dentro da pasta)

3. Criação do Vagrantfile
   * Agora vamos criar o arquivo para a nossa máquina do Nexus (Você pode editar o arquivo padrão gerado pelo vagrant init ou apagar ele e criar um novo).
   * Crie o arquivo Vagrantfile como mostrado abaixo:

```ruby
Vagrant.configure("2") do |config|
    # imagem a ser usada no setup
    config.vm.box = "ubuntu/focal64"
    # hostname da máquina a ser criada
    config.vm.hostname = "nexus-srv"
    # configuração do shape da máquina virtual
    config.vm.provider "virtualbox" do |vb|
        vb.cpus = "2"
        vb.memory = "2048"
    end
    # configuração do endereço de rede na sub-rede do virtualbox
    config.vm.network "private_network", ip: "192.168.56.3"
    # porta usada para acesso ao Nexus Web
    config.vm.network "forwarded_port", guest: 8081, host: 8081
    # porta usada para acesso ao repositório docker local
    config.vm.network "forwarded_port", guest: 8181, host: 8181
    # compartilhamento via NFS - pode ser compartilhado com outras máquinas 
    # config.vm.synced_folder ".", "/vagrant", type: "nfs"
    # script para configuração do ambiente
    config.vm.provision "shell", path: "provision.sh"
end
```
4. Criar scripts auxiliares
   * Com o descritivo da nossa máquina criado, vamos criar os scripts auxiliares que irão fazer o trabalho de instalar e configurar o Nexus
   * O primeiro script será o **provision.sh**. Ele será o responsável por instalar os pacotes necessários para o Nexus rodar em nosso ambiente.

```bash
#!/bin/bash

# Ajuste as variáveis abaixo conforme sua necessidade e versão do Nexus que será usada
NEXUS_INST_DIR="/opt"
NEXUS_INST_FOLDER="nexus"
NEXUS_USER="nexus"
NEXUS_USER_PWD="n3x02"
NEXUS_DWLD_URL="https://download.sonatype.com/nexus/3"
NEXUS_INST_FILE="nexus-3.60.0-02-unix.tar.gz"
NEXUS_VERSION="nexus-3.60.0-02"
NEXUS_INIT_PWD="$NEXUS_INST_DIR/sonatype-work/nexus3/admin.password"

#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#
######### Somente mexa nas linhas abaixo se você souber o que está fazendo ##########
#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#--#

# roda um update, caso necessário
sudo apt-get update

# verificar se o nexus já está instalado, caso sim, aborta o processo de instalação
# (checagem precisa de melhorias para evitar casos de instalações quebradas por conta de erros)
if [ -d "$NEXUS_INST_DIR/$NEXUS_INST_FOLDER" ]; then
    echo "Provisionamento ja realizado..."
else
    # instala o JDK - dependencia do nexus
    sudo apt-get install openjdk-8-jdk --yes
    sudo apt-get install wget --yes
    # instala o expect para uso configurar o usuário/senha do nexus
    sudo apt-get install expect --yes
    sudo useradd -d $NEXUS_INST_DIR/$NEXUS_INST_FOLDER -s /bin/bash $NEXUS_USER
    sudo /usr/bin/expect /vagrant/expect-nexus $NEXUS_USER $NEXUS_USER_PWD
    echo "Baixando pacote do Nexus..."
    if [ ! -f "/vagrant/$NEXUS_INST_FILE" ]; then
        wget $NEXUS_DWLD_URL/$NEXUS_INST_FILE -O /vagrant/$NEXUS_INST_FILE
    fi
    if [ -f "/vagrant/$NEXUS_INST_FILE" ]; then
        cd $NEXUS_INST_DIR ; sudo tar xzvf /vagrant/$NEXUS_INST_FILE ; cd -
        sudo mv $NEXUS_INST_DIR/$NEXUS_VERSION $NEXUS_INST_DIR/$NEXUS_INST_FOLDER

        sudo cat <<EOT >> /etc/security/limits.d/$NEXUS_USER.conf
$NEXUS_USER - nofile 65536
EOT

        # ajusta parametros de memória que a JVM para o nexus irá usar
        sudo sed -i '/\-Xms/c\\-Xms512m' $NEXUS_INST_DIR/$NEXUS_INST_FOLDER/bin/nexus.vmoptions
        sudo sed -i '/\-Xmx/c\\-Xmx512m' $NEXUS_INST_DIR/$NEXUS_INST_FOLDER/bin/nexus.vmoptions
        sudo sed -i '/MaxDirectMemorySize/c\-XX:MaxDirectMemorySize=512m' $NEXUS_INST_DIR/$NEXUS_INST_FOLDER/bin/nexus.vmoptions
        # ajusta o usuário em arquivo de conf do nexus
        sudo echo "run_as_user=\"$NEXUS_USER\"" > /opt/nexus/bin/nexus.rc
        # ajusta a permissão dos diretórios dop nexus
        sudo chown -R $NEXUS_USER:$NEXUS_USER $NEXUS_INST_DIR/$NEXUS_INST_FOLDER $NEXUS_INST_DIR/sonatype-work
        # cria o arquivo para gerenciar o start/stop do serviço via systemd
        sudo cat <<EOT >> /etc/systemd/system/nexus.service
[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=$NEXUS_INST_DIR/$NEXUS_INST_FOLDER/bin/nexus start
ExecStop=$NEXUS_INST_DIR/$NEXUS_INST_FOLDER/bin/nexus stop
User=$NEXUS_USER
Restart=on-abort
TimeoutSec=600

[Install]
WantedBy=multi-user.target
EOT

        # carrega, habilita e ativa o serviço do nexus via systemd
        sudo systemctl daemon-reload
        sudo systemctl enable nexus.service
        sudo systemctl start nexus.service

        # Finalização
        echo "Instalação finalizada. Aguardando finalizar processo de inicialização..."
        sleep 10
        echo "Instalação finalizada. Status dos serviços:"
        echo "==========================================="
        sudo systemctl status nexus.service
        echo "==========================================="
        echo "Caso a saída acima apresente erro, reuna o máximo de informações"
        echo "(cópia de logs, prints de tela dos erros, cópia do seus arquivos de config)"
        echo "e entre contato com o autor em ftvieira@pobox.com"

        cat<<EOT
A Senha inicial para acesso ao Nexus pode ser obtida da seguinte forma:

    vagrant ssh
    cat $NEXUS_INIT_PWD

Copie o conteúdo listado na tela e acesse a interface do Nexus.
O acesso deve ser feito com o usuário admin e a senha obtida acima.
Depois é só seguir as instruções para finalizar o processo e alterar a senha.

----------------------------------------------------------------------
Para acessar a interface, use:
    http://<ip_da_sua_maquina>:8081
    Ex.:
        http://192.168.0.10:8081

Para usar o repositório em cache com o docker cli (linha de comando):
    Ex.:
        docker pull 192.168.0.10:8181/ubuntu
        docker run --rm -it 10.0.2.60:8181/debian /bin/bash
    
EOT
    else
        echo "Problema no download do pacote do Nexus!"
    fi
fi
```

   * O segundo script é um auxiliar para o primeiro script. Ele faz uso do **expect** para atribuir a senha ao usuário criado sem que o usuário precise digitar a senha de forma interativa.
   * Crie o arquivo **expect-nexus** com o conteúdo abaixo:

```bash
#!/usr/bin/expect
spawn passwd [lindex $argv 0]
set senha [lindex $argv 1]
expect "ssword:"
send "$senha\r"
expect "ssword:"
send "$senha\r"
expect eof
```

5. Acessando e configurando o docker proxy no Nexus

   * Usando as instruções dadas no final do processo de instalação da máquina, acesse a interface do Nexus:

   * Tela do Nexus no primeiro acesso. Clique em Sign in no canto superior direito

![My Image](imgs/acessando_nexus_01.jpg)

   * No pop-up que aparecer, você deve informar o usuário e senha para o primeiro acesso (use as informações coletadas durante o processo de instalação)

![My Image](imgs/acessando_nexus_02.jpg)

   * Ao clicar em Sign in no pop-up, o wizard de configuração inicial iniciará para troca de senha e alguns ajustes básicos. Basta seguir as orientações das imagens.

![My Image](imgs/acessando_nexus_03.jpg)

![My Image](imgs/acessando_nexus_04.jpg)

![My Image](imgs/acessando_nexus_05.jpg)

![My Image](imgs/acessando_nexus_06.jpg)

   * Agora vamos para a configuração clicando na engrenagem próxima ao campo de pesquisa no topo da tela.

![My Image](imgs/configurando_nexus_01.jpg)

   * Primeiro vamos configurar a área de armazenagem para o nosso proxy. Como aqui é apenas um lab,vamos usar a mesma área já existente, apenas criando um novo apontamento. Para um ambiente produtivo a recomendação seria colocar estar área em uma partição, disco ou mesmo storage.

![My Image](imgs/configurando_nexus_02.jpg)

   * Clique no botão Create Blob Store

![My Image](imgs/configurando_nexus_03.jpg)

   * Na caixa de seleção escolha a opção File e depois clique em Save

![My Image](imgs/acessando_nexus_04.jpg)

   * Ao selecionar o tipo, as demais opções serão exibidas. Ajuste como na imagem e clique em Save. (nesta tela poderia ser configurada uma outra área para armazenar o cache, como em ambientes de produção por exemplo).

![My Image](imgs/acessando_nexus_05.jpg)

![My Image](imgs/acessando_nexus_06.jpg)

   * É preciso também configurar o Docker Bearer Token. Para isso, no menu do lado esquerdo, vá em Security -> Realms, como mostrado abaixo:

![My Image](imgs/configurando_nexus_06a.jpg)

   * No lado esquerdo, clique no símbolo de **+** que está no item Docker Bearer Token. Ao fazer isso, ele passará para a caixa da direita. Para finalizar, clique em Save.

![My Image](imgs/configurando_nexus_06b.jpg)

   * No menu a esquerda, clique em Repositories e depois em Create repository.

![My Image](imgs/configurando_nexus_07.jpg)

   * Na tela que abre escolha a Recipe (Receita) ***docker (proxy)***

![My Image](imgs/configurando_nexus_08.jpg)

   * Configure os dados do repositório como nas imagens abaixo

![My Image](imgs/configurando_nexus_10.jpg)

![My Image](imgs/configurando_nexus_11.jpg)

   * Selecione o Blob Storage que criamos anteriormente

![My Image](imgs/configurando_nexus_12.jpg)

![My Image](imgs/configurando_nexus_13.jpg)

   * Para finalizar, clique no botão Create repository

![My Image](imgs/configurando_nexus_14.jpg)

























