---
layout: post
title:  "Teste de Infraestrutura com Puppet: Parte I"
date:   2017-06-07 19:00:00 -0300
categories: jekyll update
---

Testes de infraestrutura aos poucos vem ganhando espaço e novas ferramentas vem surgindo (molecule e goss) enquanto as mais maduras (beaker, serverspec, rspec-puppet) tem sido ativamente mantidas e constantemente melhoradas.

Se você não está testando seu código está optando por passar algo adiante no pipeline com um baixo grau de confiança e consequentemente maior chance de falhar.

Porque escrever especificamente para Puppet? Toda a parte de fundamentos de testes pode ser reusada e aplicada para as demais tecnologias, mas hoje o conjunto de ferramentas utilizados para testar, bem como a **estratégia**, variam consideravelmente, visto que cada tecnologia optou por uma [abordagem diferente](https://ttboj.wordpress.com/2016/11/30/a-revisionist-history-of-configuration-management/) para resolver o problema de gerenciamento de configuração.

Podemos encontrar mais similaridades em nível de aceitação, onde meus testes são totalmente caixa preta, mas mesmo neste caso, o *toolset* preferido de cada tecnologia não é o mesmo.

## Níveis de teste

Uma estratégia eficaz para testar um sistema é composta de vários níveis: unitário, integração, aceitação, além de testes para aspectos mais transversais, como desempenho e segurança.

Essa decomposição em níveis tem como objetivo explorar o potencial e contornar limitações de cada nível e o termo *Pirâmide de Testes* é comumente empregado em discussões sobre o assunto: [Martin Fowler](https://martinfowler.com/bliki/TestPyramid.html) e [Google Testing Blog](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)

Embora a ideia tenha surgido para o desenvolvimento de sistemas, é possível utilizar os mesmos princípios e até mesmo criar uma pirâmide de maneira análoga para testes de infraestrutura, como em [Infrastructure as Code](http://infrastructure-as-code.com/).

É comum também o assunto cair em [discussões](https://martinfowler.com/articles/is-tdd-dead/) sobre o que exatamente se encaixa em cada nível e se um teste é de integração ou unitário, ou mesmo quando optar por um em detrimento de outro.

Ao longo dessa série, focarei apenas nas perguntas que cada um tenta responder e suas principais características. Começando por:

- **Unitário**: *Unidade* faz a *coisa certa* e possui uma *interface fácil* de ser utilizada?
  - *Unidade*: Depende da tecnologia sendo testada, e pode ser tanto um objeto quanto um *resource* ou uma *function* Puppet.

  - *Coisa certa*: Comportamento esperado é igual ao observado.

  - *Interface fácil de ser utilizada*: Ao construir um teste unitário você será colocado no papel de cliente daquele código e terá um feedback da interface do mesmo.

**Características:** Rápido, baixo custo para construção[1], determínistico e provê feedback sobre sua interface em um nível de granularidade bem baixo.

Mais em: https://pragprog.com/magazines/2012-01/unit-tests-are-first

Uma outra característica do teste unitário é ser caixa cinza, ou seja, você se preocupa com o comportamento externo (caixa preta), mas se aproveita de conhecer detalhes da implementação (caixa branca) na hora de construir os testes.

[1] Esse custo está relacionado com a qualidade do seu código. Quanto mais monolítico ou acoplado seu código, maior a dificuldade de construir testes unitários. A dificuldade para testar é apenas um reflexo da qualidade do seu código. (:

## Puppet

O Puppet possui uma abstração bem clara em sua abordagem para gerenciamento de configuração - o catálogo - definido como uma coleção de recursos com seus respectivos estados.

O catálogo é o resultado da compilação de manifestos - arquivos contendo código na DSL do Puppet. Logo, o que você declara no manifesto estará no seu catálogo e será gerenciado e tudo aquilo fora dele está fora de escopo.

> **NOTA:** Sempre busque o mínimo de estado gerenciado. Não adicione funcionalidade para o caso de precisar. Isto acarretará em aumento de complexidade, logo, prefira sempre adicionar testes e aumentar a sua capacidade de mudança para quando realmente for necessário introduzir novas features.

## Testes unitários, Puppet e você

Sabendo as características de testes unitários e do funcionamento do Puppet podemos decompor o nosso problema em: (1) compilação do catálogo e (2) aplicação do catálogo.

Compilar um catálogo é um processo muito mais *rápido* e *determínistico* do que aplicá-lo. Um catálogo é caracterizado como conjunto de recursos e sub-conjuntos deste catálogo são as menores *unidades* que podemos testar. Dessa forma, podemos traçar nosso objetivo:

**Como garantir que os recursos declarados utilizando a DSL do Puppet serão compilados em um catálogo que inclui todos os recursos necessários com o estado desejado?**

- **Escopo:** - *tudo que acontece em tempo de compilação de catálogo*:

  - Lógica baseada em parâmetros ou fatos. Recursos e classes adicionados dinamicamente.

  - Lambdas/iteração (code blocks) funcionam de forma apropriada (i.e. tem o resultado esperado em seu catálogo).

  - Chamadas a funções nativas ou de outros módulos (e.g. stdlib).

  - Restrição de unicidade para recursos isomórficos (nome = identidade) não será violada.

  - Comportamento de suas funções (Ruby ou DSL Puppet).

  - Conteúdo de arquivos a partir templates. (EPP ou ERB)

- **Provê segurança para:**

  - Migrar versões do Puppet/Ruby.

  - Realizar mudanças em nível de catálogo sem introduzir bugs. (e.g. Uma nova classe para suportar systemd sem alterar o catálogo para sysvinit)

- **Não provê nenhuma garantia:**

  - Em tempo de aplicação do catálogo.
    - O pacote desejado irá existir em algum repositório?
    - Todas as configurações necessárias foram realizadas?
    - O serviço irá iniciar com sucesso?
    - O binário executável está no PATH?
    - O uso das expressões augeas tiveram o resultado esperado?


Apenas ter um catálogo compilado com sucesso e da forma esperada não é garantia de uma aplicação bem sucedida, certo? Sim, mas é metade do caminho e te ajuda a prever facilmente o impacto de uma mudança em outras partes do seu código.

Parece promissor, mas... Como eu compilo meu catálogo? Como forjo fatos? Simulo diferentes parâmetros para minhas classes? Como verifico o catálogo compilado?

O [rspec-puppet](https://github.com/rodjek/rspec-puppet) é a resposta para estas perguntas, mas antes de entrar em detalhes, dois pontos precisam ser melhor esclarecidos:

### 1. [Four-Phase Test](http://xunitpatterns.com/Four%20Phase%20Test.html)

**Como estruturar o teste de forma que fique óbvio o seu objetivo?**

O padrão xUnit de quatro fases define as seguintes fases para um teste: setup, exercise, verify e teardown, mapearemos a seguir estas fases para o nosso problema:

  - **setup**: Preparação dos testes, tudo que acontece **antes** da execução do caso de testes.

    *Puppet:* Definição dos fatos, paramêtros, pré-condições e o manifesto a ser aplicado.

  - **exercise**: Execução efetiva do código que está sendo testado.

    *Puppet:* Compilação do catálogo.

  - **verify**: Verificar a saída do passo anterior.

    *Puppet:* O catálogo contem os recursos desejados com o estado correto.

  - **teardown**: Voltar para o estado anterior ao setup.

Outras terminologias que expressam a mesma ideia: *arrange-act-assert* e *given-when-then*

### 2. rspec

DSL escrita em Ruby que permite escrever testes de forma expressiva utilizando a terminologia do BDD (*GivenWhenThen*).

> **NOTA:** Algumas pessoas tem a crença que o simples fato de utilizar o estilo do BDD implica em BDD, *specification by example*, documentação como código e a paz mundial. Geralmente estes são os maiores produtores de contra-exemplos em http://www.betterspecs.org/

### rspec-puppet

O rspec-puppet utiliza o estilo do BDD para descrever seu teste de forma que fique claro que o estado está sendo preparado (fatos, parâmetros, pré-condições) para compilar o catálogo e que ao final este será verificado para garantir que o resultado observado é o esperado.

#### Dissecando um caso de teste

{% highlight ruby %}
# spec/classes/init_spec.rb

# 1
describe 'mymodule' do
  # 2
  context 'when using default parameters' do
    context 'on a RedHat-based OS' do
      # 3
      let :pre_condition { 'include epel '}
      # 3
      let :facts do
        {
          :kernel                 => 'Linux',
          :osfamily               => 'RedHat',
          :operatingsystem        => 'RedHat',
          :operatingsystemrelease => '6',
        }
      end

      # 4
      it { is.expected_to contain_class('mymodule::install') }
      it { is.expected_to contain_class('mymodule::repo::el') }
      it { is.expected_to contain_class('mymodule::config') }
      it { is.expected_to contain_class('mymodule::service') }
    end
    # 2
    context 'on a Debian-based OS' do
      # 3
      let :facts do
        {
          :kernel                 => 'Linux',
          :osfamily               => 'Debian',
          :operatingsystem        => 'Ubuntu',
          :operatingsystemrelease => '14.04',
        }
      end

      #4
      it { is.expected_to contain_class('mymodule::install') }
      it { is.expected_to contain_class('mymodule::repo::debian') }
      it { is.expected_to contain_class('mymodule::config') }
      it { is.expected_to contain_class('mymodule::service') }
    end
  end
  # 2
  context 'when using a tarball source' do
    # 3
    let :params { :install_source => 'http://acme.org/my.tar.gz'}
    # não faz diferença qual o SO, pois não precisamos configurar repos (:

    # 4
    it { is.expected_to contain_class('mymodule::install') }
    it { is.expected_to contain_class('mymodule::config') }
    it { is.expected_to contain_class('mymodule::service') }
  end

end
{% endhighlight %}

1. O bloco `describe` <class, define, function> irá determinar qual é o elemento sendo testado e incluí-lo no catálogo. No exemplo acima teremos o equivalente a `include myclass`.

2. O bloco `context` define um cenário de teste sob determinadas condições e pode ser aninhado para expressar casos mais complexos, como no exemplo acima, onde temos: *"mymodule when using default parameters on a RedHat-based OS"*, combinando todas as condições que expressam esse cenário: `include myclass` e fatos que identificam um SO baseado no RedHat.

3. É válido ressaltar que o `context` tem caráter apenas de documentação e demarcação de escopo, deixando para os blocos `let` e `before` o papel de definir de fato o estado. O *rspec-puppet* provê algumas facilidades nesse sentido, como o `:params`, `:facts` e `:pre_condition`, que permitem, respectivamente, definir parâmetros para o elemento sendo testado, definir fatos e pré-condições (código Puppet).

4. Por último precisamos realizar as verificações, com os blocos it (`it { is.expected_to ... }`), que em conjunto com os matchers do *rspec-puppet* podem nos ajudar a fazer asserções sobre o catálogo gerado.

Por fim, a segurança dos seus testes vai depender de quão bem os cenários foram montados e quão boa são suas asserções. Dessa forma o *rspec-puppet* provê uma gama de matchers, do mais simples, como `it { is_expected.to compile }`. (Lê-se: "É esperado que o catálogo compile")

Até outros mais elaboradas derivados de `contain_<resource type>(<resource name>)`, como:
`contain_class('mymodule::install')` ou `contain_file('/etc/yum.repos.d/mymodule.repo')`

Podemos ir mais longe e verificarmos o estado dos recursos:
{% highlight ruby %}
it { is_expected.to contain_file('/etc/yum/repos.d/mymodule.repo').with(:content => 'a=b') }
{% endhighlight %}

E seu relacionamentos:

{% highlight ruby %}
it { is_expected.to contain_class('mymodule::service').that_requires('mymodule::config') }
{% endhighlight %}

Outros matchers e exemplos podem ser encontrados no [GitHub](https://github.com/rodjek/rspec-puppet) do projeto.

#### Testando de forma inteligente em vez de testar mais, TDD e goiabada para sobremesa

Verificar todos os recursos e seus respectivos parâmetros e propriedades em um catálogo para todos os cenários possíveis é inviável tecnicamente e pouco efetivo. Para contornar este problema utilizamos o **particionamento em classes de equivalência**.

Na prática, definimos uma condição que representa todos elementos de um conjunto e utilizamos apenas **um** elemento que satisfaça nossa condição como entrada no nosso teste, visto que os demais *deverão* ter comportamento similar. Nesta abordagem o maior desafio é encontrar as classes de equivalência que precisamos testar, e no exemplo acima pudemos identificar três: RedHat-based, Debian-based, Uso de tarball.

Encontrar essas classes de equivalência de forma a produzir testes relevantes requer prática, e uma técnica que pode auxiliar neste processo de descoberta é o **TDD** (Test-driven development), devido a prática de *test-first* e *baby steps*.

#### Exemplo de relatório de execução do rspec para o módulo *puppetlabs-apache*:

```
apache::mod::auth_cas
  default params
    behaves like a mod class, without including apache
      should compile into a catalogue without dependency cycles
  default configuration with parameters
    on a Debian OS
      should compile into a catalogue without dependency cycles
      should contain Class[apache::params]
      should contain Apache::Mod[auth_cas]
      should contain Package[libapache2-mod-auth-cas]
      should contain File[auth_cas.conf] with path => "/etc/apache2/mods-available/auth_cas.conf"
      should contain File[/var/cache/apache2/mod_auth_cas/] with owner => "www-data"
    on a RedHat OS
      should compile into a catalogue without dependency cycles
      should contain Class[apache::params]
      should contain Apache::Mod[auth_cas]
      should contain Package[mod_auth_cas]
      should contain File[auth_cas.conf] with path => "/etc/httpd/conf.d/auth_cas.conf"
      should contain File[/var/cache/mod_auth_cas/] with owner => "apache"
```

Mais exemplos de código: https://github.com/puppetlabs/puppetlabs-apache/tree/master/spec e https://github.com/puppetlabs/puppetlabs-ntp/tree/master/spec

### Passo a passo

Como começar com testes com rspec-puppet em projeto existente?

>**NOTA:** Requer Ruby instalado. Caso não possua RVM, rbenv, asdf ou qualquer ferramenta análoga. Recomendo que escolha uma destas e instale antes de iniciar.

{% highlight bash %}
cd mymodule
gem install bundler --no-ri --no-rdoc
bundle init
bundle add rspec-puppet
bundle install
mkdir spec/{classes, defines} # spec/{classes, defines, applications, functions, types, types_aliases, hosts}
{% endhighlight %}

{% highlight ruby %}
# spec/spec_helper.rb
require 'rspec-puppet'
require 'puppetlabs_spec_helper/module_spec_helper'


base_dir = File.dirname(File.expand_path(__FILE__))

RSpec.configure do |c|
  c.module_path     = File.join(base_dir, 'fixtures', 'modules')
  c.manifest_dir    = File.join(base_dir, 'fixtures', 'manifests')

  c.after(:suite) do
    RSpec::Puppet::Coverage.report!
  end
end
{% endhighlight %}

{% highlight ruby %}
# spec/classes/init_spec.rb

require 'spec_helper'

describe 'mymodule' do
  context 'when using default parameters' do
    # ...
  end
end
{% endhighlight %}


## Extras

Código de testes também deve ter qualidade, afinal, você precisará mantê-lo junto com o seu código. Procure remover código de testes desnecessário e utilizar ferramentas como: rubocop-rspec (melhores práticas rspec, tamanho dos testes e etc), codeclimate (duplicação, acompanhamento via interface) e consultar exemplos em betterspecs.org

Um outro aspecto importante é implementar a cultura de retroalimentar sua base de testes. Sempre que encontrar um novo bug, verifique se não seria possível adicionar um teste para prevenir que este aconteça novamente. Adicione este teste, que irá falhar, e corrijá o problema após.
