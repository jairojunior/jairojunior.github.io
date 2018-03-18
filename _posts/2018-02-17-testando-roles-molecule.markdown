---
layout: post
title:  "Testando Roles com Molecule"
date:   2018-02-17 12:00:00 -0300
categories: ansible modules dev testes
---

1. [Ansible Roles](#ansible-roles)
2. [Testes?](#testes)
3. [Molecule](#molecule)
4. [Passo a Passo](#passo-a-passo)
    * [virutalenv](#virtualenv)
    * [Molecule](#molecule)
    * [TravisCI](#travisci)

# Ansible Roles

Você começa a utilizar Ansible para automatizar uma tarefa, muitas vezes com simples um playbook *ad-hoc*, pois o escopo da sua automação é pequeno, mas em outros casos o seu trabalho poderia ser organizado em um *role*.

Tecnicamente um *role* pode ser definido como um conjunto de *vars_files*, *tasks*, *handlers*, *files*, e *templates* que servem a um propósito específico (common, hardening, network e etc) ou tecnologia específica (apache, jenkins, gitlab e etc).

Ser coeso é muito importante na hora de construir um  *role*, pois é uma das principais unidades de reúso em Ansible, você não cria um role para OpenStack, por exemplo, e sim para seus componentes isoladamente (Keystone, Swift, Nova, Cinder e etc).

Um servidor pode ter N *roles* e dessa forma podemos fazer composições, o que é bastante útil para testes ou para combinar *roles* de diferentes camadas (os, network, hardening, middleware, app) da sua stack, mas em produção, idealmente, seus servidores deverão ter um único *papel*: banco de dados, balanceador de carga, aplicação, DNS, DHCP, NFS e etc.

Garantir que seu servidor possui um único papel facilita bastante o gerenciamento e reduz o risco ao realizar mudanças. Em [Infrastructure as Code](https://www.amazon.com/Infrastructure-Code-Managing-Servers-Cloud/dp/1491924357), a utilização de infraestrutura compartilhada alimenta um ciclo vicioso, onde a compartilhamos pois é caro provisionar novos servidores e a mudança é muito arriscada, pelo alto impacto de negócio e por restrições técnicas, já que não há isolamento algum.

Esse tipo de cenário leva organizações a manter um comitê para aprovar mudanças (i.e. *CAB* - *Change Advisory Board*, mas bem que poderia ser *Change Avoidance Board*). Mas o que é mais caro no fim das contas? O custo de hardware para ter isolamento ou manter um comitê e um ritmo de mudança em que consigo entregar apenas a cada X meses?

>**NOTA:** Sobre os benefícios de entregar com frequência, assistam [este trecho](https://youtu.be/OZnxSEXD3Tk?t=20m17s) da minha apresentação sobre Testes de Infraestrutura no DevOpsDays BSB '17 ou leia basicamente qualquer livro [dessa](https://jairojunior.github.io/jekyll/update/2017/06/16/lista-leitura-devops.html) lista.

Se o Ansible estiver sendo utilizado da forma correta, provisionar novos servidores - da VM até a camada de aplicação - deve ser algo barato, e isolar meus serviços (i.e. não compartilhar infraestrutura) provavelmente valerá a pena, principalmente em um cenário *on-premise* com um ambiente tradicional virtualizado onde há pouca ou nenhuma elasticidade.

>**NOTA:** Falando de provedores de Cloud - onde as faturas podem ser exorbitantes - este cenário pode ser um pouco diferente e é preciso avaliar bem as perdas e ganhos. Temos ainda o cenário onde parte do *workload* está preparado para executar em containers de processo, e neste caso, orquestradores de container (i.e. Kubernetes) provêm um excelente isolamento com um *overhead* bastante inferior a VM's.

# Testes?

Testes são uma ferramenta para ajudar o desenvolvimento, você está sempre testando e precisa de feedback constante para direcionar seus próximos passos.

Passos grandes tornam difíceis de identificar erros (assim como grandes entregas) e essa é uma das essências do TDD (*Test Driven Development*), o uso de *baby steps*, passos pequenos, apoiados pelo feedback dos testes.

**Como aplicar esta técnica no desenvolvimento de playbooks *ad-hoc* e *roles*?**

Em um playbook *ad-hoc*, um fluxo razoável começaria com:

`vagrant init centos/7 # ou qualquer outra imagem`

Introduziríamos o trecho abaixo no Vagrantfile para utilizar o Ansible da sua máquina (host) para executar um playbook como alvo o (guest):

```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
  ansible.verbose = true
end
```

E ao final ficaríamos em um ciclo com os comandos abaixo: 

```shell
vagrant up
vagrant provision
# editar playbook
vagrant ssh # para verificar algum estado
# provision e ssh ad infinitum
```

Ocasionalmente as máquinas seria destruídas e criadas novamente para aumentar a confiabilidade do seu script. (i.e. `vagrant destroy -f; vagrant up`)

>**NOTA:** Aliases ajudam bastante aqui, e o projeto bash-it no GitHub [ajuda bastante](https://github.com/Bash-it/bash-it/blob/master/aliases/available/vagrant.aliases.bash) com aliases para diversas ferramentas, como o Vagrant - caso você seja uma pessoa sem criatividade para dar nomes, como eu. (:

Esse ciclo é bem parecido com o [Red-Green-Refactor](http://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html) do TDD, com a grande diferença que em vez de deixar o computador realizar testes/verificar o estado (i.e. testes automatizados), esse papel é realizado por você.

Existem dois problemas quando *você* (i.e. *homo sapiens* ou algo mais evoluído) decide fazer o trabalho do computador neste caso:

- Computador é muito mais eficiente para tarefas repetitivas. 
- O tempo dele é mais barato que o seu.

Existem algumas ferramentas que podem te ajudar nesta tarefa, como o [serverspec](http://serverspec.org/), [goss](https://github.com/aelsabbahy/goss) e [testinfra](https://github.com/philpep/testinfra).

Embora você provavelmente esteja se coçando para utilizar o *goss*, já que é feito em Go, falarei do *testinfra* (Python), pois acredito ser mais popular no mundo Ansible e é a opção padrão para o *Molecule*.

A ideia é simples, programar as verificações - antes realizadas manualmente - em um script python:

```python
def test_nginx_is_installed(host):
    nginx = host.package("nginx")
    assert nginx.is_installed
    assert nginx.version.startswith("1.2")


def test_nginx_running_and_enabled(host):
    nginx = host.service("nginx")
    assert nginx.is_running
    assert nginx.is_enabled
```

Em um ambiente de desenvolvimento, como o nosso até aqui, o script executa no *host* conectando-se no *guest* via ssh:

`py.test --connection=ssh --hosts=server`

>**NOTA:** Combinação de `py.test --ssh-config=/path/to/ssh_config --hosts=server` e `vagrant ssh-config` podem ser utilizados para simplificar a execução caso esteja utilizando o Vagrant.

O que acabamos de fazer foi construir um teste de aceitação automatizado, que tem como principal característica o fato de ser caixa-preta, ou seja, preocupa-se apenas com o funcionamento do ponto de vista externo. 

Em resumo, nosso desafio é: **Provisionar servidores, executar Playbooks Ansible tendo estes como alvo e verificar que a configuração aplicada reflete o estado desejado.**

- **O que verificamos?** - *qualquer verificação em tempo de execução*:

  - *Infraestrutura está operacional do ponto de vista do usuário.*

  - Serviço está em execução ou habilitado.

  - Serviço está escutando em uma porta específica.

  - Serviço está respondendo a requisições de forma adequada.

  - Usuário foi criado.

  - Arquivos de configuração foram copiados ou gerados corretamente a partir de templates.

  - E virtualmente qualquer verificação programável.

- **Qual segurança essa base de testes nos provê?**

  - Realizar mudanças complexas ou introduzir features sem quebrar o comportamento existente. (e.g. Continua funcionando em RedHat-based após adicionar suporte ao Ubuntu)

Se olharmos para o que fizemos até agora, podemos facilmente mapear todos estes passos para o padrão [Four-Phase Test](http://xunitpatterns.com/Four%20Phase%20Test.html) - uma forma de estruturar testes que deixa claro o seu objetivo - e temos as seguintes fases: *setup*, *exercise*, *verify* e *teardown*:

  - **setup**: Preparação dos testes, tudo que acontece **antes** da execução do caso de testes.

    *vagrant up*

  - **exercise**: Execução efetiva do código que está sendo testado.

    *vagrant provision*

  - **verify**: Verificar a saída do passo anterior.

    *py.test*

  - **teardown**: Voltar para o estado anterior ao setup.

    *vagrant destroy*

Poderíamos utilizar os mesmos conceitos para testar Roles definidos em um master playbook, mas será que terei que ter este trabalho repetitivo toda vez que iniciar o desenvolvimento? E se eu quiser utilizar Docker em vez de Vagrant ou provisionar uma instância no OpenStack da minha organização? E se eu quiser utilizar o *goss* em vez do *infratest*? Como rodar isso no meu *CI Server*?

Existe um jeito mais rápido e simples de fazer tudo isso?

# Molecule

O *Molecule* é uma ferramenta que auxilia bastante no desenvolvimento com testes, e sua instalação é bem simples, podendo ser feita via pip, como o próprio Ansible.

A primeira coisa sobre o *Molecule* é que ele possui uma operação para criação de um *role* já com tudo pré-definido para execução de testes de aceitação: `molecule init role –role-name foo` e também é possível facilmente adicionar as configurações do molecule a um role existente.

Um outro aspecto importante é sua flexbilidade, permitindo utilizar diferentes *drivers* para infraestrutura: Docker\*, Vagrant, OpenStack, GCE, EC2, Azure e para bibliotecas de verificação de servidores: testinfra\* e goss. Os itens marcados com asterisco são as opções *default* e utilizadas em um *init* sem parâmetros.

Por último, a utilização dos seus comandos facilitam a execução de tarefas comumente necessárias durante o fluxo de desenvolvimento:

- *lint* - Executa yaml-lint, ansible-lint, flake8 e falha caso problemas sejam apontados
- *syntax* - Verifica o role para erros de sintaxe.
- *create* - Cria instância(s) com o driver configurado.
- *prepare* - Configura as instâncias com playbook para preparação.
- *converge* - Executa playbook no(s) host(s) alvo.
- *idempotence* - Executa playbook duas vezes e falha caso existam mudanças na segunda execução.
- *verify* - Executa biblioteca de verificação. (testinfra ou goss)
- *destroy* - Destroí instância(s).
- **test** - Executa todos os passos anteriores.

Adicionalmente, o comando *login* pode ser utilizado para conectar em servidores provisionados para fins de *troubleshooting*. 

# Passo a Passo

Como vou do zero teste a testes executando continuamente no meu CI Server com o Molecule?

## virtualenv

*virtualenv* é uma ferramenta para criar ambientes isolados de Python enquanto o *virtualenvwrapper* é um conjunto de extensões para facilitar o uso do virtualenv.

O uso dessas ferramentas evita problemas como conflitos entre dependências do molecule e demais pacotes python instalados na sua máquina.

```sh
sudo pip install virtualenvwrapper
export WORKON_HOME=~/envs
source /usr/local/bin/virtualenvwrapper.sh
mkvirtualenv mocule
```

## Molecule

Instalar o molecule com driver para docker:

`pip install molecule ansible docker-py`

Gerar um novo role já com testes:

`molecule init role -r role_name`

Ou para roles existentes:

`molecule init scenario -r jboss`


Com isso toda configuração necessária para subir um ambiente configurado com o seu role é gerada e preciso me preocupar apenas em construir meus casos de teste em *molecule/default/tests/*:

```python
import os

import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']).get_hosts('all')


def test_jboss_running_and_enabled(host):
    jboss = host.service('wildfly')

    assert jboss.is_enabled


def test_jboss_listening_http(host):
    socket = host.socket('tcp://0.0.0.0:8080')

    assert socket.is_listening


def test_mgmt_user_authentication(host):
    command = """curl --digest -L -D - http://localhost:9990/management \
                -u ansible:ansible"""

    cmd = host.run(command)

    assert 'HTTP/1.1 200 OK' in cmd.stdout
```

Os testes do exemplo acima verificam basicamente se o serviço no SO está habilitado, se há um processo escutando na porta HTTP e se a autenticação do Wildfly foi configurada corretamente.

Estes testes são bastante simples de serem construídos, você precisa, basicamente, pensar em uma forma de verificação caixa-preta para o que o seu role se propõe a fazer. Se está acostumado a construir verificações para seu sistema de monitoramento/alerta, será bem mais fácil reutilizar esse conhecimento para fazer algo com a API do [testinfra](https://testinfra.readthedocs.io/en/latest/) ou utilizando um comando.

## TravisCI

Executar seus testes continuamente com o Molecule é bastante simples. O exemplo abaixo funciona para o TravisCI com Docker, mas pode facilmente ser adaptado para qualquer CI Server e qualquer um dos drivers suportados pelo Molecule. 

```yaml
---
sudo: required
language: python
services:
  - docker
before_install:
  - sudo apt-get -qq update
  - pip install molecule
  - pip install docker-py
script:
  - molecule test
```
[Resultado](https://travis-ci.org/jairojunior/ansible-role-jboss/builds/345731738)