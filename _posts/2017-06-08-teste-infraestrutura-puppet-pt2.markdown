---
layout: post
title:  "Teste de Infraestrutura com Puppet: Parte II"
date:   2017-06-08 19:00:00 -0300
categories: jekyll update
---

Na continuação desta série o foco será em testes de aceitação, partindo da seguinte definição:

- **Aceitação:** O sistema como um todo funciona?

  **Características:** Lento, não-determinístico (frágil), alto custo para construção, porém com alto valor de negócio (provê feedback do funcionamento da perspectiva do usuário final).

Enquanto testes unitários provêem feedback quase imediato, executando em poucos segundos, os testes de aceitação precisam de um pouco mais de tempo para executar, de poucos minutos a horas.

No caso de testes de infraestrutura, este tempo é gasto para provisionar máquinas virtuais ou iniciar containers, baixar e instalar pacotes, iniciar ou reiniciar serviços.

Estas características são as mesmas que também o tornam não determínisticos, uma pequena variação na disponibilidade de recursos no servidor que executa os testes e podemos não conseguir subir uma VM, ter um timeout para iniciar um serviço ou executar um comando, um download corrompido, uma dependência que não pode ser baixada do Forge.

Logo, testes de aceitação são bastante frágeis e podem facilmente sinalizar um problema na sua mudança, mesmo que a causa real esteja em outro lugar que não seu código.

# Puppet

Na parte anterior explicamos em alto nível sobre a abstração do catálogo no Puppet, mas nos preocupamos apenas com a compilação do mesmo e deixamos de lado a sua aplicação - o que realmente realiza mudanças e garante o estado.

A aplicação do catálogo se comporta de forma diferente de acordo com o modo de execução do Puppet:

- **Masterless:** Manifesto é aplicado diretamente, ou seja, o catálogo existe apenas em memória e os insumos (fatos, dados do hiera, módulos) são locais.

- **Master/Agent:** Agente envia fatos para um Master e solicita seu catálogo, este por sua vez, irá consultar a classificação deste servidor, fazer lookup dos seus dados, carregar os módulos utilizados, compilar o catálogo e enviar ao agente, que por fim irá aplicá-lo e retornar o relatório de execução para o Master.

Embora a arquitetura *Master/Agent* seja significativamente diferente da *Masterless*, dificilmente um código funciona apenas em uma das arquiteturas, com raríssimas excecões, como carregamento de bibliotecas em código nativo dentro de `lib/` (i.e. o que é entregue pelo `pluginsync`), além das `functions`, que são executadas no Master na arquitetura *Master/Agent*.

Por esta razão, é comum realizar testes de aceitação com Puppet utilizando apenas modo Masterless (i.e. `puppet apply`), visto que a complexidade de construir/executar testes multi-node geralmente não compensa os benefícios para grande parte dos módulos.

Em todo caso, mesmo quando seu módulo se encaixar cenários que se beneficiam de testes com a arquitetura *Master/Agent*, os testes com a arqutetura *Masterless* ainda serão bastante úteis para prever impacto e cobrir boa parte das suas mudanças.

# Testes de aceitação, Puppet e você

Neste nível o desafio é aplicar o catálogo e verificar a configuração de um servidor. Poderíamos fazer isso localmente, mas teríamos dificuldade em testar diferentes configurações e um trabalho gigantesco para limpar o estado após a execução, por sorte podemos trabalhar facilmente com máquinas virtuais ou containers de processo para esse propósito.

**Como provisionar servidores, aplicar manifestos Puppet e verificar que a configuração reflete o estado desejado?**

- **Escopo** - *qualquer verificação após aplicação de catálogo*:

  - *Infraestrutura está operacional do ponto de vista do usuário.*

  - Serviço está em execução ou habilitado.

  - Serviço está escutando em uma porta específica.

  - Serviço está respondendo a requisições de forma adequada.

  - Usuário foi criado.

  - Virtualmente qualquer verificação programável.

- **Provê segurança para:**

  - Realizar mudanças complexas sem quebrar o comportamento existente. (e.g. Continua funcionando em RedHat-based após adicionar suporte ao Ubuntu)


Ao ler sobre testes de aceitação a primeira reação geralmente é: *"Vamos fazer apenas esse tipo de teste e ignorar os demais!"*...

Respire fundo e pense nas limitações do teste de aceitação, fragilidade, lentidão para ter feedback, o custo para construí-los... Se você pode ter o mesmo nível de confiabilidade em um nível inferior na pirâmide de testes, adicione cobertura naquele nível e deixe para aceitação apenas só pode ser coberto em nível de aceitação.

*Exemplo:* Não vale a pena verificar o estado de um arquivo que o conteúdo é gerado a partir de um template em *aceitação* quando poderíamos facilmente testar o template em nível *unitário*.

## Ferramentas

*Quais ferramentas podem me ajudar a construir esse tipo de testes para Puppet?*

- Beaker e serverspec.

*Posso utilizar outras?*

- Sim, mas essas duas são de longe das mais populares na comunidade e consequentemente acarretará em maior facilidade de encontrar suporte e bons exemplos (todos módulos supported e approved utilizam essa combinação).

As fases do padrão [Four-Phase Test](http://xunitpatterns.com/Four%20Phase%20Test.html) também são úteis para entender o ciclo de vida dos testes de aceitação e onde essas ferramentas se encaixam nele:

- **setup:** *(Global)* Preparação para o estado antes da execução de qualquer teste.

  *Beaker:* Provisionar servidor(es) utilizando um dos seus *hypervisors*, instalar agente do Puppet, instalar dependências (módulos, pacotes ou qualquer outra coisa), copiar módulo sendo testado para o servidor.

  - **setup** (Opcional) Preparação antes da execução do caso de teste.

  - **exercise:** Execução efetiva do código que está sendo testado.

    *Beaker:* Aplicação de manifestos.

  - **verify:** Verificar a saída do passo anterior.

    *Beaker:* Manifesto foi aplicado sem erros e manifesto foi aplicado uma segunda vez sem mudanças (i.e. é idempotente).

    *serverspec:* Usuários e grupos existem? Serviço está em execução? Serviço está ouvindo na porta designada? *Qualquer comando/script para verificar uma configuração foi bem sucedido?*

  - **teardown** (Opcional) Limpeza após execução do caso de teste.

- **teardown:** *(Global)* Voltar para o estado anterior ao setup.

  *Beaker:* Destruir servidor(es).

### Beaker e serverspec

**Beaker** é uma ferramenta desenvolvida para construção/execução de testes de aceitação com Puppet e possui as seguintes capacidades:

  - Provisionamento de máquinas virtuais ou containers de processo com suporte a uma grande variedade de providers: vagrant, docker, openstack, gce, ec2, vsphere, vmpooler.
  - Disponibiliza helpers para execução de comandos nos servidores.
  - Interação entre máquinas virtuais.

**serverspec** é uma biblioteca que permite escrever testes rspec com asserções próprias para infraestrutura.

#### Passo a Passo

Como iniciar com beaker e serverspec em um novo projeto?

Dependências:

{% highlight bash %}
cd mymodule
gem install bundler --no-ri --no-rdoc
bundle init
bundle add beaker
bundle add beaker-rspec
bundle add beaker-puppet_install_helper
bundle add serverspec
bundle install
mkdir spec/acceptance
{% endhighlight %}

>**NOTA:** Requer Ruby instalado. Caso não possua RVM, rbenv, asdf ou qualquer ferramenta análoga. Recomendo que escolha uma destas e instale antes de iniciar.

O primeiro passo para iniciar a construção de testes de aceitação é definir quais `nodes` serão utilizados durante a execução dos seus testes e para isso o beaker permite declarar `nodesets` em arquivos YAML:

{% highlight yaml %}
# spec/acceptance/nodesets/default.yml

HOSTS:
  centos-72-x64:
    roles:
      - agent
      - default
    platform: el-7-x86_64
    box: puppetlabs/centos-7.2-64-nocm
    hypervisor: vagrant

CONFIG:
  type: aio
{% endhighlight %}

>**NOTA:** Para configurações mais especficas consulte a documentação do [Vagrant como Hypervisor do Beaker](https://github.com/puppetlabs/beaker/blob/master/docs/how_to/hypervisors/vagrant.md).

Posteriormente iremos integrar o beaker ao *rspec* , para que nossos *nodesets* sejam provisionados antes da execução dos casos de teste e para prepararmos nossos *nodes* com tudo necessário para aplicação dos manifestos Puppet. Caso seja necessário configurações adicionais para o seu módulo, consulte a documentação do [Beaker::DSL](http://www.rubydoc.info/github/puppetlabs/beaker/Beaker/DSL)

{% highlight ruby %}
# spec/spec_helper_acceptance.rb

require 'beaker-rspec/spec_helper'
require 'beaker-rspec/helpers/serverspec'
require 'beaker/puppet_install_helper'

# Utilitário para instalar agente do Puppet
run_puppet_install_helper

PROJECT_ROOT = File.expand_path(File.join(File.dirname(__FILE__), '..'))

RSpec.configure do |c|

  c.before :suite do
    hosts.each do |host|
      # Instala dependências utilizando Puppet
      on host, puppet('module', 'install', 'puppetlabs-apache')

      # Copia módulo sendo testado para servidor provisionado
      copy_module_to(host, source: PROJECT_ROOT, module_name: 'example')
    end
  end

end
{% endhighlight %}

Finalmente poderemos escrever nossos casos de teste:

{% highlight ruby %}
# spec/acceptance/config_abc_spec.rb

require 'spec_helper_acceptance'

describe 'my case xpto' do
  context 'config abc' do
    it 'applies manifest without error' do
      pp = <<-EOS
        include xpto
      EOS

      apply_manifest(pp, :catch_failures => true, :acceptable_exit_codes => [0, 2])

      # Novamente para verificar que não houve mudanças e nosso código é idempotente
      expect(apply_manifest(pp, :catch_failures => true).exit_code).to be_zero
    end

    describe service('xpto') do
      it { should be_enabled }
      it { should be_running }
    end

    describe port(8888) do
      it { should be_listening }
    end

  end
end
{% endhighlight %}

Por fim, executamos com: `PUPPET_INSTALL_TYPE=agent rspec spec/acceptance/config_abc_spec.rb`

[Sobre o PUPPET_INSTALL_TYPE.](https://github.com/puppetlabs/beaker-puppet_install_helper)

Ver projeto [exemplo](https://github.com/jairojunior/exemplo-puppet-beaker).

>**NOTA:** Perceba que não precisamos dizer ao beaker onde estão nossos testes ou nodesets e tão pouco dizer qual nodesets queremos utilizar. No mundo Ruby, Convenção sobre configuração (Convention over Configuration - CoC) reduz o número de decisões que precisamos tomar ao assumir algumas convenções.

Variáveis de ambiente para execução dos testes Beaker:

- `BEAKER_set`: *nodeset* a ser utilizado.
- `BEAKER_debug`: Definir qualquer valor para habilitar o modo debug que mostra detalhes do funcionamento do beaker e aplicação dos manifestos Puppet.
- `BEAKER_destroy`: Por padrão os nodes são destruídos após a execução dos testes. Ao utilizar a opção `no` para esta variável os nodes permanecerão em execução e `onpass` os manterá vivos apenas em caso de falha. Essa opção é muito útil para troubleshooting.
- [Mais](https://github.com/puppetlabs/beaker-rspec)

### Matriz

Uma dúvida comum é como construir uma matriz para execução em diferentes SO's. Com o uso do beaker, atingir este objetivo localmente é tão simples quanto ter múltiplos *nodesets* e um script para iterar sobre eles:

{% highlight bash %}
#!/bin/bash

BOXES=( default centos-66-x64 debian-78-x64 debian-82-x64 ubuntu-1404-x64 ubuntu-1604-x64 )

for BOX in "${BOXES[@]}"
do
  PUPPET_INSTALL_TYPE="agent" BEAKER_set="${BOX}" bundle exec rake beaker
done
{% endhighlight %}

> **NOTA:** Alguns providers como o vagrant com o VirtualBox não suportam o provisionamento paralelo de máquinas virtuais.

> **NOTA:** Servidores de Integração Contínua possuem recursos para execução de comandos com variáveis de ambiente assumindo diferentes valores que permitem facilmente construir matrizes. (e.g. `matrix` em `env` no TravisCI)
