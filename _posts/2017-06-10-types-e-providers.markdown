---
layout: post
title:  "Types e Providers Explicado"
date:   2017-06-10 19:00:00 -0300
categories: puppet dev
---

Se você trabalha com Puppet há um certo tempo **TEM** que entender como *Types* e *Providers* funcionam.

Não apenas para decidir quando deve utilizá-lo em vez de um exec que executa um script - ou dois, mas também porque é essencial para entender o funcionamento do Puppet e dos *types* e *providers* nativos e/ou entregues por módulos de terceiros utilizados.

# Defined Types x Custom Types

Não confundir *custom types* com *defined types*, que são apenas uma forma de criar um *wrapper* em cima de/reutilizando outro(s) recurso(s) existente(s). Neste artigo falaremos de *custom resource types*, mas iremos nos referir a estes apenas como *types.*

# Types e Providers: o que são?

De fora o Puppet permite utilizar uma abordagem declarativa para gerenciar recursos que nativamente são gerenciados de forma imperativa. Mágica? Embora pareça a primeira vista, a técnica utilizada para resolver este problema é conhecida como *abstração*, que dentre outras dezenas de exemplos em computação permite que: exista um protocolo confiável (TCP) em cima de outro não confiável (IP); conexão persistente (HTTP *Keep-Alive*) em cima de uma pilha que não entende esse conceito, ou ainda um framework Web para manter estado em cima de um protocolo *stateless* como o HTTP.

Então alguém cria essas *abstrações* e só preciso me preocupar em resolver meus problemas? Errrr...

Há 15 anos, *Joel Spolsky* - criador do StackOverflow - apresentou um conceito interessante: [*The Law of Leaky Abstraction*](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/), que dentre outros pontos afirma que:

- *"All non-trivial abstractions, to some degree, are leaky."*

As abstrações vão falhar, pouco ou muito, mas em algum momento você será surpreendido com um erro vindo diretamente da camada inferior. O que você faz? Alguém precisa "descer o nível" e entender o que está acontecendo...

- *"So the abstractions save us time working, but they don’t save us time learning."*

As abstrações nos poupam tempo de trabalho (i.e. nos tornam mais produtivos), mas não nos isentam de aprender, *au contraire*, agora precisamos entender também a abstração. :)


Da mesma forma que não é uma boa ideia [*automatizar o desconhecido*](https://ma.ttias.be/automating-unknown/), também não é uma boa ideia utilizar uma ferramenta de gerência de configuração sem entender seu funcionamento ou os recursos sendo *abstraídos* (`package`, `user`, `file`, `concat`, `file_line`, `augeas`, etc).

Recursos nativos são utilizados o tempo todo e muitos módulos também se beneficiam de types e providers para entregar funcionalidades. (e.g. `java_ks`, `file_line`, `concat`, `archive`, etc)

Cedo ou tarde você precisará entender a abstração do Puppet, ou melhor, qual a "mágica" empregada para ser *declarativo* e *convergir* para o estado desejado de maneira *idempotente*. **Entender isto aumentará consideravelmente sua produtividade e capacidade de depurar problemas.**

## Declarativo:

*Estado* de um *recurso*.

{% highlight puppet %}
user { 'jjunior':
	gid    => 1001,
	shell  => '/usr/bin/bash',
	groups => ['admin'],
}
{% endhighlight %}

## Convergência:

Tratar *transições* de *estado*.

De:

{% highlight puppet %}
user { 'jjunior':
	gid    => 1001,
	shell  => '/usr/bin/bash',
	groups => ['admin'],
}
{% endhighlight %}

Para:

{% highlight puppet %}
user { 'jjunior':
	gid    => 1001,
	shell  => '/usr/bin/zsh',
	groups => ['admin'],
}
{% endhighlight %}

Diff:

- Alterar o shell de bash para zsh.

## Idempotência

Aplicado múltiplas vezes sem alterar o *estado* final.

De:

{% highlight puppet %}
user { 'jjunior':
	gid    => 1001,
	shell  => '/usr/bin/zsh',
	groups => ['admin'],
}
{% endhighlight %}

Para:

{% highlight puppet %}
user { 'jjunior':
	gid    => 1001,
	shell  => '/usr/bin/zsh',
	groups => ['admin'],
}
{% endhighlight %}

Diff:

- Não houve mudança de *estado*, tudo certo por aqui. :)

## Mais...

- Outro motivo para aprender? - Enriquecer seu *toolbox* de desenvolvedor de infraestrutura para:

	- ter a oportunidade de refatorar *aquele* exec/script para um *custom type*
	- poder dizer com segurança que algo poderia ser melhor escrito como um custom type
	- ou que algo poderia fazer parte de um custom type existente
	- ou que poderia ser um novo *provider* para aquele *type*

Em resumo: *habilidade de gerenciar qualquer coisa que possua uma interface de comunicação*: servidores de aplicação, ferramentas de monitoramento, cofre de senha, nuvem privada e etc.

# Types e providers: quando utilizar?

- Preciso *gerenciar* o *estado* de um *recurso* e os meios convencionais [1] não puderem me ajudar.

[1] `file` com template, `augeas`, `concat` e `file_line` são recursos poderosos para gerenciar arquivos de configuração, mas não resolvem todos os problemas...

Às vezes:
 - a única forma de gerenciar um recurso é via API ou ferramenta CLI - que pode retornar dados estruturados ou não.
 - arquivos de configuração muito complexos (XML, XSD, wtf?)

Já sei! Um `exec` com `sed`, `awk`, `curl`, `xmlstarlet` e `jq` e *nós vamos voar alto!* Ou ainda, uma outra alternativa - igualmente inapropriada - uma `function` que "gera" a configuração que preciso.

Para começar, essas abordagens estão longe de serem *declarativas* e precisarão de uma *abstração* própria para realizar *transições de estado* e serem *idempotente*.

Cuidado com síndrome de `exec`. Começa com: "`exec` é poderoso, `exec` pode fazer qualquer coisa MWUAHAHAAHHA (risada maléfica)", passando por "Pra que template se tenho sed -i?" e termina de forma trágica seu manifesto sendo apenas um instrumento para declarar ações *imperativas* (scripts/comandos) com grandes chances de não ser *idempotente*. Não utilize Puppet como um meio de executar shell script, se esta for sua necessidade, existem ferramentas mais apropriadas para isso.

O Puppet possui um modelo bem definido para tratar esse problema, e funciona muito bem, afinal, é utilizado por todos `types` e `providers` que nós conhecemos e utilizamos.

>**NOTA:** Trecho da documentação oficial do exec: *"A caution: There’s a widespread tendency to use collections of execs to manage resources that aren’t covered by an existing resource type. This works fine for simple tasks, but once your exec pile gets complex enough that you really have to think to understand what’s happening, you should consider developing a custom resource type instead, as it will be much more predictable and maintainable."*

# Construindo Types e Providers

Em breve a construção de types e providers será facilitada com o [pdk - Puppet Development Kit](https://github.com/puppetlabs/pdk) - que também irá permitir que parâmetros e propriedades sejam tipados, eliminando a necessidade de validação/coerção de tipos.

Apesar destas mudanças, os conceitos fundamentais aqui explicados não mudarão.

*Requisitos:* Ruby

Qual o nome do seu type? Qual recurso ele abstrai? Vamos criar um *custom resource type* que gerencia um repositório no GitHub.

{% highlight ruby %}
# lib/puppet/type/github_repository.rb
Puppet::Type.newtype(:github_repository) do
	ensurable

	newparameter(:name) do
	end

	newparameter(:access_token) do
	end

	newproperty(:description) do
	end

	newproperty(:private) do
	end

	newproperty(:has_issues) do
	end
end
{% endhighlight %}

Ciclo de vida de um recurso `ensurable`:

{% highlight yaml %}
present:
  exists?:
    Sim:
      para_cada_propriedade_gerenciada:
        insync?:
          Sim: Não faz nada.
          Não: Atualiza propriedades necessárias.
    Não: create

absent:
  exists?:
    Sim: destroy
    Não: Não faz nada.
{% endhighlight %}

`exists?`, `create`, `destroy` são métodos que devem ser implementados para, respectivamente: verificar se um recurso existe, criá-lo com o estado desejado e destruí-lo.

Veremos mais detalhes sobre o `insync?` e o gerenciamento de uma propriedade na seção específica para este propósito.

**NOTA:** Na arquitetura Master/Agent, Types, providers e facts dentro de `/lib` são distribuídos automaticamente via [pluginsync](https://docs.puppet.com/puppet/4.10/plugins_in_modules.html#auto-download-of-agent-side-plugins-pluginsync).

## Parameters

Os parâmetros mudam a forma como o Puppet gerencia um recurso, mas não necessariamente são mapeados para o estado de um recurso.

No nosso exemplo, temos `name` - um tipo especial de parâmetro - e o `access_token`, nossa credencial utilizada para gerenciar o repositório *GitHub*.

Um outro exemplo de parâmetro bastante comum é o `ensure`, que possui uma forma simplificada de ser definido: chamada a `ensurable`. Este parâmetro tem como objetivo determinar a ausência/presença de um recurso, mas em casos especiais, como do `package` podem estar relacionado com uma propriedade de um recurso (i.e. versão).

*O que torna o `name` especial?*

Para responder essa questão precisamos primeiro entender o que são recursos *isomórficos*, e a definição é simples: *"Um recurso cujo nome corresponde a sua identidade (i.e. torna único)"*. Exemplos: usuários, grupos, arquivos e basicamente qualquer outra coisa exceto o `exec`.

Não posso gerenciar, em um mesmo sistema, dois usuários ou grupos com mesmo nome, ou arquivos com mesmo path, mas posso ter o mesmo comando `exec` quantas vezes quiser.

Dessa forma, o `name` é especial pois por padrão ele é o `namevar` - o que identifica um recurso. Embora possamos escolher outro `namevar` para nosso recurso, já que `path` é o `namevar` de `file`, e o `command` do `exec`, diferente de `user` que utiliza o o padrão (`name`).

> **NOTA:** Existem ainda casos especiais de recursos que precisam de mais de um parâmetro para identificá-los, como o `package`, já que posso ter o mesmo pacote (`name`) em `providers` distintos (e.g. docker-yum e docker-pip). Esta técnica é conhecida como *composite namevar* e seu comportamento está atrelado ao comportamento do `title_patterns`.

### `namevar`, `name`, `title`, e agora?

O `name` assume o valor do `title` por padrão, exceto quando o parâmetro marcado como `namevar` (que por padrão é `name`) for passado.

Por essa razão que você pode "brincar" com o title quando o `namevar` for definido:

{% highlight puppet %}
file { 'tantofaz':
	path => '/tmp/foo'
}

file { 'qualquercoisa':
	path => '/tmp/bar'
}
{% endhighlight %}

Mas ainda assim **não** poderiam existir recursos com mesmo `title` em um catálogo, já que ele é utilizado para outros propósitos - como ordenação. Um bom exemplo de uso do `title` em conjunto com `namevar` é justamente como uma forma de referenciar um recurso se seu `namevar` for dinâmico, como no exemplo abaixo:

{% highlight puppet %}
$service_name = $facts['osfamily'] ? {
  'Debian' => 'apache2',
  default  => 'httpd',
}

service { 'apache':
	name => $service_name,
}

resource { 'xpto':
	require => Service['apache'],
}
{% endhighlight %}

**NOTA:** Existem também [metaparameters](https://docs.puppet.com/puppet/latest/metaparameter.html) disponíveis para serem utilizados em todos recursos, *defined types* ou *custom types*.

## Properties

As propriedades podem ser mapeadas diretamente para o estado de um recurso. Podendo ser consultadas ou modificadas, como o caso da *description* e o *toggles* para *private* e *has_issues*  do nosso repositório *GitHub*.

Diferente dos parâmetros, as propriedades são gerenciadas - possuem um ciclo de vida bem definido, controlado basicamente por três métodos: `insync?(is)`, `property_name`, `property_name=(value)`

>**NOTA:** Se a propriedade não for de nenhum tipo complexo ou possuir alguma lógica especial para verificação, a implementação padrão de `insync?(is)` (compara o valor dos objetos) será suficiente e não haverá necessidade de sobreeescrevê-la.`

{% highlight ruby %}
# lib/puppet/provider/github_repository/http_api.rb
Puppet::Type.type(:github_repository).provide :http_api do

	# exists?, create, destroy

	def description
		# Lê description
	end

	def description=(value)
		# Atualiza description
	end

	def private
		# Verifica se é private
	end

	def private=(value)
		# Atualiza private
	end

	def has_issues
		# Verifica se *has_issues*
	end

	def has_issues=(value)
		# Atualiza *has_issues*
	end
end
{% endhighlight %}

No exemplo acima temos todos os métodos necessários para ler o estado atual das propriedades e atualizá-las caso necessário. Essa lógica é específica de *Provider*.

Na prática, como essas leituras e modificações acessariam a API HTTP do GitHub teríamos que implementar de uma forma mais eficiente para minimizar os acessos.

>**NOTA:** Alguns métodos adicionais de `Puppet::Property` podem ser sobreescritos para exibir de forma expressiva as mudanças em uma property. `is_to_s`: a forma como o valor atual será exibido, `should_to_s`: a forma como o valor desejado será exibido, `change_to_s`: mensagem exibida quando uma propriedade é alterada.

## Provider(s)

Como vimos acima, nossos *types* interagem diretamente com nossos *providers*, pois a forma como as propriedades são gerenciadas irá variar de acordo com o *provider*.

No nosso exemplo, não precisamos nos preocupar com diferentes sistemas operacionais e distribuições, já que a "única forma" de gerenciar nosso repositório é através da API HTTP, e essa integração irá funcionar da mesma forma em qualquer lugar utilizando bibliotecas nativas do Ruby.

No mundo real encontramos cenários mais complexos, como os providers `yum`, `puppet_gem`, `pip`, `apt` para o type `package`, ou `posix` e `windows` para o `file`.

Como restringir um provider a uma característica de um sistema? Como tratar as diferenças de funcionalidades entre diferentes providers?

`confine` e `feature` são os recursos para resolver este problemas e mais detalhes podem ser encontrados na [documentação oficial](https://docs.puppet.com/puppet/4.10/provider_development.html#suitability).

## autorequire

O `autorequire` é outro recurso interessante utilizado na construção de `types`, pois permite que um auto-relacionamento seja configurado, ou seja, caso um recurso específico seja encontrado no catálogo, um recurso daquele `type` será executado apenas após aquele recurso.

Ele é o que permite, dentre outros exemplos, que um arquivo e um diretório pai não precisem ser explicitamente relacionados.

{% highlight puppet %}
file { '/opt/app':
	ensure => 'directory',
}

file { '/opt/app/foo': }
{% endhighlight %}

## Mais...

Leia código: Tipos nativos ([lib/puppet/type](https://github.com/puppetlabs/puppet/tree/master/lib/puppet/type) e [lib/puppet/provider](https://github.com/puppetlabs/puppet/tree/master/lib/puppet/provider) em *puppetlabs-puppet*) e diversos outros exemplos: *puppetlabs-aws, puppetlabs-ciscopuppet, puppetlabs-postgresql, puppetlabs-mysql, puppetlabs-java_ks, , puppet-zabbix, puppet-archive*
