# README

Pequeno laboratório de estudos para aprimorar conhecimentos de Vagrant e estudar a ferramenta Nexus como proxy de imagens para o Docker Hub, entre outros recursos.

Foi utilizado o Vagrant com Virtualbox para subir a máquina virtual e foi criado um script para instalar e configurar o Nexus na mesma.

### **Procedimentos**

1. Instalação do Vagrant
   * Acesse [Download Vagrant](https://developer.hashicorp.com/vagrant/downloads?product_intent=vagrant) e escolha o seu Sistema operacional. Aqui vamos usar o ambiente **Linux** como base.
   
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
   * Crie uma pasta aonde ficarão os arquivos de configuração e inicialize o vagrant nesta pasta:
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

5. Acessando o Nexus e configurando o docker proxy

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

   * Agora precisamos criar uma Role para o acesso ao repositório. Afinal não podemos dar acesso de admin para todos que precisarem utilizar o repositório. Basta seguir os passos abaixo:

   * No menu a esquerda, selecione Roles

![My Image](imgs/configurando_nexus_14a.jpg)

   * Depois clique no botão Create role

![My Image](imgs/configurando_nexus_14b.jpg)

   * Preencha os campos indicados. O ***Type*** será **Nexus Role**. Os demais campos podem ter os nomes que desejar.

![My Image](imgs/configurando_nexus_14c.jpg)

   * Na parte de Privileges, no lado esquerdo, coloque o filtro com o nome do repositório criado. No nosso caso, foi docker-proxy.

![My Image](imgs/configurando_nexus_14d.jpg)

   * Procure na lista o privilégio adequado, conforme a imagem e clique no sinal de **+** para jogá-lo para a janela da direita.

![My Image](imgs/configurando_nexus_14e.jpg)

   * Para finalizar, vai para o final da página e clique no botão Save.

![My Image](imgs/configurando_nexus_14f.jpg)

   * Com a Role criada, vamos criar nosso usuário (vamos criar um usuário genérico, mas podem ser criados mais usuários, dependendo das necessidades do ambiente)

   * No menu do lado esquerdo, selecione Users, como mostrado na imagem

![My Image](imgs/configurando_nexus_15.jpg)

   * Selecione Create local user

![My Image](imgs/configurando_nexus_16.jpg)

   * Siga as orientações das imagens abaixo e preencha os campos necessários para a criação do usuário no Nexus.

![My Image](imgs/configurando_nexus_17.jpg)

   * Associe a role que foi criada com o usuário. Caso seu ambiente já possua um servidor Nexus,podem existir muitas roles e, neste caso, você deverá pesquisar a role criada para o repositório. Clique no **>** para mover a role para a janela da direita. Para finalizar, clique em Create local user.

![My Image](imgs/configurando_nexus_18.jpg)

![My Image](imgs/configurando_nexus_19.jpg)


6. Configurando e testando o docker cli com nosso repositório (Linux)

   * Para fazer com que o docker cli no Linux possa usar nosso cache e não o Docker Hub diretamente, devemos fazer uma pequena configuração.
   * Devemos editar ou criar o arquivo ***/etc/docker/daemon.json***, como mostrado abaixo:

```json
{
        "insecure-registries": ["<ip_da_sua_maquina>:8181"],
        "registry-mirrors": ["http://<ip_da_sua_maquina>:8181"]
}
```
   * Após editar ou criar este arquivo, o daemon do docker deve ser reiniciado. No nosso laboratório a máquina cliente roda o Ubuntu 20.04, logo o comando será:

```bash
sudo systemctl restart docker
```
   * Para confirmar que as configurações foram reconhecidas, execute o comando:

```bash
docker info
```

   * Nas informações geradas deverá ter um trecho semelhante a este:

```bash
 Insecure Registries:
  192.168.1.129:8181
  127.0.0.0/8
 Registry Mirrors:
  http://192.168.1.129:8181/
```

   * Agora podemos testar nosso repositório. Para isso, teremos que nos autenticar, pois não deixamos o repositório com acesso anônimo. Para isso, vamos executar:

```bash
docker login -u dockerlogin 192.168.1.129:8181
```

   * Será pedida a senha que definimos para o usuário e a resposta deverá ser algo semelhante a isso:

![My Image](imgs/configurando_nexus_20.jpg)

   * Após fazer o login, podemos começar a usar nosso repositório.
   * Vamos começar simplesmente baixando a imagem do ubuntu:

```bash
docker pull 192.168.1.129:8181/ubuntu
```
   * Deverá produzir uma saída semelhante a esta:

![My Image](imgs/configurando_nexus_21.jpg)

   * Para validar que o download da imagem foi realizado pelo nosso cache e que uma cópia da mesma está disponível, vamos ver no Nexus se a imagem do ubuntu está mesmo lá.
   * Para isso, vamos na interface e clicamos na opção Browse.

![My Image](imgs/configurando_nexus_22.jpg)

   * No menu a esquerda, selecione Browse

![My Image](imgs/configurando_nexus_23.jpg)

   * Na tela que se abre, selecionamos o nosso repositório

![My Image](imgs/configurando_nexus_24.jpg)

   * Será exibida uma estrutura similar a mostrada abaixo

![My Image](imgs/configurando_nexus_25.jpg)

   * Se clicarmos nos botões de **+** ao lado das pastas, veremos a estrutura mostrando que a imagem do ubuntu agora está no nosso cache local

![My Image](imgs/configurando_nexus_26.jpg)












   













































