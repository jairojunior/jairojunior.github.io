---
layout: post
title:  "Desenvolvimento de Módulos Ansible"
date:   2017-12-03 13:00:00 -0300
categories: ansible modules dev
---

O Ansible funciona conectando em nós e enviando pequenos programas, chamados de "módulos" para execução. Dessa forma, temos uma arquitetura *push* (configuração é "empurrada") - sem agentes - ao contrário do modelo *pull* (configuração é "puxada") - dos sistemas baseado em agentes (Puppet, SaltStack e etc).

Esses **módulos** são mapeados para **recursos** e seu **estado** (representado em arquivos YAML), com eles podemos **gerenciar** virtualmente *qualquer coisa* que possua uma API, CLI ou arquivo de configuração. Exemplos incluem dispositivos de redes, como load balancers e firewalls, passando por orquestradores de containers, os próprios containers, VM's em um hypervisor ou mesmo instâncias em nuvens públicas (AWS, GCE, Azure e etc) e/ou privadas (OpenStack e CloudStack), equipamentos de storage e configurações de sistema operacional (arquivos, services, usuários, grupos e etc).

Com a filosofia "baterias inclusas" do Ansible, [centenas de módulos (mais de 1250)](http://docs.ansible.com/ansible/latest/list_of_all_modules.html) já estão disponíveis com a instalação. Qualquer task no seu playbook possui um **module** por trás: `copy`, `debug`, `command`, `file`, `user`, `group`, `cron`, `service`, `ec2`, `aws_s3`, `mysql_db` e etc.

>**NOTA:** Não confundir *modules* com *roles*, disponíveis na [Galaxy](http://galaxy.ansible.com)), e definido como um conjunto de *tasks* e *vars* para uma tecnologia específica (e.g. apache, jenkins, kibana) e que pode ser adicionada a um servidor. *Roles* podem ser utilizado para entregar *modules*.

O Ansible define um contrato muito simples - *JSON* na stdout - para construção desses módulos, de forma que configuração declarada em playbooks (*YAML*) possa ser convertida em pequenos programas e entregue via *SSH/WinRM* - ou qualquer outro [connection plugin](https://docs.ansible.com/ansible/devel/plugins/connection.html) - e sejam executados em servidores remotos. Dessa forma, modules podem ser escritos em qualquer linguagem que seja capaz de retornar *JSON*, embora todos os modules entregues com Ansible sejam escritos em *Python* utilizando a *API* do Ansible para este propósito, o que facilita bastante as coisas.

*Modules* são uma das formas de *ampliar* as capacidades do Ansible. Outras formas, como [*Dynamic inventories*](http://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.html) e [*plugins*](http://docs.ansible.com/ansible/latest/dev_guide/developing_plugins.htm) também se enquadram nesta categoria e é interessante conhecer sua aplicabilidade, pois podem ser mais apropriados para o seu problema. (:

- *Dynamic Inventories*: Script para obter informações de inventário de fontes externas.

*Plugins* são divididos em diversas categorias com propósitos específicos, como *Action*, *Cache*,*Callback*, *Connection*, *Filters*, *Lookup* e *Vars*. Dentre estes, podemos descrever os mais populares da seguinte forma:

- *Connection plugins* implementam uma forma de comunicação servidores do inventário (e.g. SSH, WinRM), ou seja, como os programas são transportados pela rede e executados em um servidor remoto.
- *Filters plugins* permitem manipular dados dentro de um playbook ou template Ansible (e.g. ). Está é uma funcionalidade do Jinja2, e ampliada pelo Ansible, com plugins mais voltados para problemas de infraestrutura como código.
- *Lookup plugins* são utilizados para trazer dados de uma fonte externa (e.g. env, file, hiera, database). São implementados como uma função *Jinja2*.

# Quando desenvolver um módulo?

Embora muitos módulos sejam entregues com o Ansible, existe a possibilidade do seu problema de automação ainda não fazer parte da base de código do Ansible, seja por não ser popular, ou por tratar-se de um problema específico da sua organização (serviço/tecnologia interna).

>**NOTA:** Antes de começar a trabalhar em algo, sempre verifique os PR's abertos, consulte no canal do IRC (#ansible-devel) e/ou pesquise na [lista de desenvolvimento](https://groups.google.com/forum/#!forum/ansible-devel) e em [working groups](https://github.com/ansible/community/) existentes.

Como identificar que preciso de um module em vez de utilizar os existentes?

- Meios convencionais (e.g. templates, `file`, `get_url`, `lineinfile` e etc) de gerenciamento de recursos não atendem.
- Sou "obrigado" a recorrer a combinação de *commands*, *shell*, *filters*, processamento de texto com *regexes* "mágicas", chamadas a API utilizando curl…

**Resultado**: Playbooks complexos, imperativos, não idempotentes e não determinísticos.

**Cenário ideal**: Existe uma API ou CLI para gerenciamento - e esta retorna dados estruturados (JSON, XML, YAML e etc).

- *“Faça amor, não faça shell script em YAML.”*

**Exemplo**: Playbook Raiz

```yaml
- name: Lê recurso remoto
   command: "curl -v http://xpto/resource/abc"
 register: resource
 changed_when: False

 - name: Cria recurso caso nao exista
   command: "curl -X POST http://xpto/resource/abc -d '{ config:{ client: xyz, url: http://beta, pattern: *.* } }'"
   when: "resource.stdout | 404"

 # Deixar aqui para quando precisar deletar hehehe
 #- name: Remove recurso
 #  command: "curl -X DELETE http://xpto/resource/abc"
 #  when: resource.stdout == 1
```

Além de bastante frágil (e se o nome do recurso fosse 404, ele seria criado?), este recurso seria apenas criado de forma aparentemente idempotente, mas não seria capaz de atualizar um recurso existente (*convergir* para o estado desejado).

Playbooks escritos desta maneira desrespeitam vários princípios da infraestrutura como código. Não é *legível para seres humanos*, é *difícil de reusar ou parametrizar*, além de não seguir o paradigma declarativo das ferramentas de gerência de configuração e consequentemente falhar em ter um comportamento *idempotente* e em *convergir* para o estado declarado.

Abusar de playbooks desta natureza pode minar a adoção de automação. Em vez de explorar as capacidades de uma ferramenta de gerencia de configuração, trazemos os mesmos problemas de uma abordagem imperativa baseada em scripts e execução de comandos. Levando a observações como: *"Estou apenas copiar meus scripts para YAML."*.

>**NOTA:** Entender conceitos como o paradigma declarativo, idempotência e convergência são essenciais, mesmo para quem é apenas usuário de ferramentas de gerência de configuração. Existe bastante discussão online sobre o assunto, a começar pelo [excelente artigo](https://groups.google.com/forum/#!topic/ansible-project/WpRblldA2PQ) do Michael DeHaan - criador do Ansible, assim como a [revisão histórica](https://ttboj.wordpress.com/2016/11/30/a-revisionist-history-of-configuration-management/) do James, que trabalha no [mgmt](https://ttboj.wordpress.com/2016/01/18/next-generation-configuration-mgmt/) - a próxima geração de ferramentas de gerência de configuração. Em português, O artigo sobre [types e providers do Puppet](https://jairojunior.github.io/puppet/dev/2017/06/10/types-e-providers.html) publicado neste blog, possui uma descrição bem resumida desses conceitos.

**Exemplo**: Module

```yaml
- name: XPTO
  xpto:
    name: abc
    state: present
    config:
      client: xyz
      url: http://beta
      pattern: "*.*"
```

Se implementado corretamente, a abordagem baseada em *module* apresentará as seguintes características:

- Declarativo - recurso é representado como YAML
- Idempotente
- Capaz de convergir para o *estado desejado* independente do *estado atual*.
- Legível para seres humanos.
- Facilmente parametrizável e reusado.

>**NOTA:** Código não é interpretado apenas por computadores, mas também por seres humanos. 80/20 desenvolvimento/manutenção.

## Case JBoss

**Antes**: Playbook Raiz

```yaml
 - name: Read datasource
   command: "jboss-cli.sh -c '/subsystem=datasources/data-source=DemoDS:read-resource()'"
   register: datasource

 - name: Create datasource
   command: "jboss-cli.sh -c '/subsystem=datasources/data-source=DemoDS:add(driver-name=h2, user-name=sa, password=sa, min-pool-size=20, max-pool-size=40, connection-url=.jdbc:h2:mem:demo;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE..)'"
   when: 'datasource.stdout | outcome => failed'
```

*Problemas:*

- JBoss-CLI retorna texto puro em uma sintaxe JSON-like, logo, esta abordagem é bastante frágil, pois estamos fazendo uma espécie de parser para essa notação. Mesmo um parser para algo simples, como JSON, pode ser algo [complexo](https://tools.ietf.org/html/rfc7159). 
- JBoss-CLI é apenas uma interface CLI para falar com a Management API (porta 9990).
- Executar algo nativamente é mais eficiente do que abrir uma sessão de terminal e executar um novo processo.
- Não é declarativo.
- Não é capaz de convergir para o estado desejado.

**Depois**: Module ([PR #25422](https://github.com/ansible/ansible/pull/25422))

```yaml
- name: Configure datasource
      jboss_resource:
        name: "/subsystem=datasources/data-source=DemoDS"
        state: present
        attributes:
          driver-name: h2
          connection-url: "jdbc:h2:mem:demo;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE"
          jndi-name: "java:jboss/datasources/DemoDS"
          user-name: sa
          password: sa
          min-pool-size: 20
          max-pool-size: 40
```

*Resultado:* Declarativo, idempotente, legível para seres humanos e capaz convergir de qualquer estado declardo para o estado desejado, 

## Razões para aprender

- Contribuir com módulos existentes.
- Tenho playbooks *"raiz"* e gostaria de melhorá-los...
- Ou não tenho, mas gostaria de evitar.
- Melhorar consideravelmente minha capacidade de depurar problemas em Playbooks e aumentar consideravelmente sua produtividade.

- “...abstractions save us time working, but they don’t save us time learning.”

# Custom Ansible Modules 101

- *JSON* (JavaScript Object Notation) no *stdout* - esse é o contrato.
- Pode ser escrito em qualquer linguagem - desde que retorne JSON (duh).
- Desenvolver em Python utilizando a API do Ansible costuma ser a melhor opção.
- Modules entregues com o Ansible (lib/ansible/modules/) devem ser escritos em Python e suportar as versões compatíveis (>= 2.6 <= 3.5)

## Ansible-way

*Primeiro passo*: `git clone https://github.com/ansible/ansible.git`

Navegar em lib/ansible/modules/ e conhecer *modules* existentes.

*Ferramentas:* git, python, virtualenv, pdb (debugging)

## Alternativa

```
library/                  # se houverem custom modules, coloque-os aqui (opcional)
module_utils/             # se houverem custom modules, coloque-os aqui (opcional)

site.yml                  # Playbook principal

roles/
    common/               # esta hierarquia representa um "role"
        tasks/            #
            main.yml      #
        library/          # roles podem incluir custom modules
        module_utils/     # roles podem incluir custom module_utils
```

- Mais fácil para começar.
- Não precisa de nada além do Ansible já instalado e sua IDE/editor de texto preferido.
- É o melhor caminho caso o module tenha utilidade apenas na sua organização.

*Primeiro passo:* Criar diretório library dentro do diretório onde está seu playbook ou em roles/<name>/.

## Primeiros passos

Você poderia fazer tudo por conta própria - inclusive em outra linguagem - ou utilizar a classe  AnsibleModule (Python)...

Vantagens: Jeito mais fácil de colocar JSON na stdout (`exit_json()`, `fail_json()`) da forma esperada pelo Ansible (msg, meta, has_changed, result), acessar parâmetros (`params[]`) e logar a execução (`log()`, `debug()`).

### Código

```python
def main():

  arguments = dict(name=dict(required=True, type='str'),
                  state=dict(choices=['present', 'absent'], default='present'),
                  config=dict(required=False, type='dict'))

  module = AnsibleModule(argument_spec=arguments, supports_check_mode=True)
  try:
      if module.check_mode:
          # Não realiza operações, apenas verifica o estado e reporta
          module.exit_json(changed=has_changed, meta=result, msg='Fez alguma coisa ou não...')

      if module.params['state'] == 'present':
          # Verifica se recurso existe
          # Estado desejado `module.params['param_name'] é igual ao estado consultado?
          module.exit_json(changed=has_changed, meta=result)

      if module.params['state'] == 'absent':
          # Remove o recurso caso exista
          module.exit_json(changed=has_changed, meta=result)

  except Error as err:
      module.fail_json(msg=str(err))
```

>**NOTA:** O *check_mode* ("dry run") permite que um playbook seja executado e apenas verifique se mudanças são necessárias, mas não as realiza.

>**NOTA:** O diretório *module_utils* pode ser utilizado para entregar código compartilhado entre diferentes modules.

## Testes

### Ansible-way

Existe um [working group](https://github.com/ansible/community/tree/master/group-testing) para tratar exclusivamente de testes na base de código do Ansible. Atualmente o número de testes unitários é baixo no Ansible e a maioria dos testes são de integração. 

A [estratégia de testes](https://jairojunior.github.io/jekyll/update/2017/06/07/teste-infraestrutura-puppet-pt1.html) varia consideravelmente de acordo com a ferramenta atualizada e testes unitários nem sempre são possíveis em Ansible, dessa forma, há um investimento grande em análise estática (lint) e testes de integração. Essas verificações são executadas continuamente no [CI Server do projeto no GitHub](https://app.shippable.com/github/ansible/ansible/dashboard), o *Shippable*.

*Testes de integração no Ansible*: Utiliza *containers* e o próprio Ansible para *setup* e *verify* dos testes:

```yaml
- name: Configure datasource
 jboss_resource:
   name: "/subsystem=datasources/data-source=DemoDS"
   state: present
   attributes:
     connection-url: "jdbc:h2:mem:demo;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE"
     ...
 register: result

- name: assert output message that datasource was created
 assert:
   that:
      - "result.changed == true"
      - "'Added /subsystem=datasources/data-source=DemoDS' in result.msg"
```

### Alternativa

[*Molecule*]() + [*Vagrant*]() + [*pytest*](): `molecule init` (dentro de roles/<name>)

Maior flexibilidade para escolher:

- Como subir sua infraestrutura: Vagrant, Docker, OpenStack, EC2
- Como verificar o estado da sua infraestrutura: testinfra e goss

>**NOTA:** Começar com VM's facilita bastante a reprodução de problemas no ambiente real. *Containers* para este propósito são interessantes por serem mais rápidos e "baratos" (computacionalmente), mas [containers não são VM's](*Falta link*) e exigem esforço para termos imagens de *containers* análogas a imagens de *VM's*, isto ocasionalmente leva a falsos positivos - testes que passam em *VM's*, mas falham em *containers*.

>**NOTA:** Utilizo principalmente VM's para testes durante o ciclo de desenvolvimento e containers apenas quando quero feedback rápido ou onde possuo essa limitação, como no TravisCI, embora seja possível provisionar instâncias em uma nuvem pública para este propósito.