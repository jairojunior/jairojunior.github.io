---
layout: post
title:  "Lista de Leitura para DevOps"
date:   2017-06-16 20:00:00 -0300
categories: jekyll update
---

Quem me conhece um pouco melhor sabe o quanto gosto de recomendar a leitura de livros. 

Após ler um livro muito bom que me faz repensar o que sei ou apresentar algo completamente novo irei chatear as pessoas mais próximas até que elas o comprem e possam entender melhor aquela linha de pensamento ou mesmo ajudar a ajustar/melhorar algo que interpretei de uma forma diferente - basicamente forço a pessoa a entrar em um *"Clube do Livro"* comigo (:

Há também o caso de livros que compilam boa parte do que estudei no passado recente e me dão um feedback que estou no caminho certo - ou próximo dele.

Costumo dizer que se você está fazendo algo inédito em computação é um forte candidato ao próximo [Turing Award](http://amturing.acm.org/) ou *idiota do ano*. Minha auto-crítica sempre leva a crer na segunda opção e me obriga a constantemente a encontrar referências ou construí-las a partir de connhecimento existinte, afinal, pouquíssimas vezes na vida fui *realmente* pioneiro.

# Livros?

*"There are three kinds of men. The one that learns by reading. The few who learn by observation. The rest of them have to pee on the electric fence for themselves."*

*― Will Rogers*

Eventualmente você aprenderá algumas coisas apenas do jeito difícil. Mas, e se houvesse uma forma de evitar isso? Acredito que os livros são uma excelente ferramenta para este propósito - além de ter um excelente custo benefício.

Existe uma infinidade de [livros gratuitos](https://github.com/EbookFoundation/free-programming-books) e novos sendo disponibilizados [a cada dia](https://www.packtpub.com/packt/offers/free-learning), além de [outras iniciativas](https://leanpub.com/dockerparadesenvolvedores) e [meios](https://www.goodreads.com/) de acompanhar sua lista de leitura.

Não tem desculpa para não começar /agora/. Não estou aqui para vender uma metodologia milagrosa que fará você ler *30 livros em 30 dias* ou algo do tipo. Se planeje da forma que achar adequado, mas não deixe de ler, trace objetivos ambiciosos e alcance-os, em pouco tempo os resultados irão aparecer.

Sim, existem outros meios sensacionais para aprender: cursos, vídeo aulas, artigos, projetos open source e etc. Ler livros não elimina esses outros meios, pelo contrário, é uma grande combinação.

# Fundamentos

*"An investment in knowledge always pays the best interest."*

*― Benjamin Franklin*

É comum olhar para vagas de emprego, analisar os requisitos e estudar aquelas tecnologias - deixando todos fundamentos para trás. Cuidado! Este pode ser um caminho perigoso...

*At some point I made the decision to focus on foundational concepts; not features of a particular implementation; my tech career took off.*

*― [Kelsey Hightower](https://twitter.com/kelseyhightower/status/826528907381739520)*

Além do tweet acima, outros textos também apontam para a [importância de aprender fundamentos antes de tecnologias](https://disi.unitn.it/~lissandrini/notes/learn-the-fundamentals.html):

> I graduated in Computer Science in the early 2000s.
>
> When I took a Databases class, NoSQL didn't exist.  
> When I took a Computer Graphics class, OpenGL didn't support shaders.  
> When I took a Computer Security class, no one knew about botnets yet.  
> When I took an Artificial Intelligence class, deep learning didn't exist.  
> When I took a Programming Languages class, reactive programming wasn't a «thing».  
> When I took a Distributed Systems class, there was no Big Data or cloud computing.  
> When I took an Operating Systems class, hypervisors didn't exist (in PCs at least).  
> When I took a Networking class, there was no wifi in my laptop or internet in my phone.  
>
> Learn the fundamentals. The rest will change anyway.
> 
*— "Learn the Fundamentals" by Hisham H. Muhammad*

A ideia desta lista de leitura é te ajudar a contruir fundamentos sobre o que é considerado importante em DevOps. Não importa se trabalha com Docker, Puppet, Ansible, Jenkins ou TravisCI, afinal, são "apenas" ferramentas. Fundamentos irão te ajudar independente de tecnologia e aumentar o seu potencial de aprender novas tecnologias e analisá-las de forma científica.

>**NOTA:** Esta lista foi publicada inicialmente em [awesome-devops-br](https://github.com/devops-br/awesome-devops-br) e pretendo mantê-la atualizada em ambos canais. A versão do blog terá um pouco mais de conteúdo.

### Formato

**Título do Livro** (*Autor1, Autor2, AutorN*): Pequena resenha sobre o conteúdo do livro.

> **Citações**:

* *"Citação 1"*
* *"Citação 2"*

### Lista

**The Phoenix Project: A Novel about IT, DevOps, and Helping Your Business Win** (*Kevin Behr, George Spafford, Gene Kim*): Uma narrativa sobre a introdução de DevOps em uma empresa fícticia - que em certos momentos farão você cogitar a possibilidade do Gene Kim ser um espião trabalhando ao seu lado devido as grandes semelhanças com _qualquer empresa de TI_. Será impossível não se identificar de forma assustadora com os personagens do livro e dar pequenos sorrisos ao encontrar versões "da vida real" dos mesmos.

**The DevOps Handbook: How to Create World-Class Agility, Reliability, and Security in Technology Organizations** (*Gene Kim, Jez Humble, Patrick Debois, and John Willis*): Está com a sensação de que o projeto unicórnio do Phoenix Project é pura ficcão? Não sabe como colocar o "Three Ways" em prática ou por onde começar? Este livro vai te ajudar a entender DevOps e ilustrar o case de grandes organizações completamente transformadas ou impulsionadas por DevOps, Lean, Agile, TPS e etc.

> **Citações**:

* *"In DevOps, we typically define our technology value stream as the process required to convert a business hypothesis into a technology-enabled service that delivers value to the customer."*

* *"Agile often serves as an effective enabler of DevOps, because of its focus on small teams continually delivering high quality code to customers."*

* *"DevOps isn’t about automation, just as astronomy isn’t about telescopes."*

* *"Because value is created only when our services are running in production, we must ensure that we are not only delivering fast flow, but that our deployments can also be performed without causing chaos and disruptions such as service outages, service impairments, or security or compliance failures."*

* *"Instead of a culture of fear, we have a high-trust, collaborative culture, where people are rewarded for taking risks."* 

**Release It!: Design and Deploy Production-Ready Software** (*Michael T. Nygard*): Porque a Amazon não fica indisponível na Black Friday e meu site não fica em pé com um aumento de 20% na carga por causa de um cupom novo? Qual o custo dessa indisponibilidade para a minha empresa? Como reverter essa situação? Porque a distância entre `feature-complete` e `production-ready` é tão grande? Se você *não* estiver familiarizado com os termos "Inversão de SLA", "Circuit Breaker", "Zero downtime deployments", "Falhas em cascata", "Bulkheads", "Unbalanced Capacities", "Unbounded Result Sets", leia-este-livro-agora.

> **Citações**:

* *"First, you need to accept that fact that despite your best laid plans, bad things will still happen. Second, realize that “Release 1.0” is not the end of the development project but the beginning of the system’s life on its own.*"

* *"Software design today resembles automobile design in the early 90s: disconnected from the real world."*

* *"Systems spend much more of their life in operation than in development—at least, the ones that don’t get canceled or scrapped do."*

* *"Don't avoid one-time development expenses at the cost of recurring operational expenses."*

* *"Team assignments are the first draft of the architecture. (See ​Conway’s Law​.) It’s a terrible irony that these very early decisions are also the least informed."*

* "*Denying the inevitability of failures robs you of your power to control and contain them."*

**Continuous Delivery: Reliable Software Releases Through Build, Test, and Deployment Automation** (*Jez Humble, David Farley*): Todas as funcionalidades foram implementadas, mas ainda serão necessárias semanas ou meses para seu software ser entregue. Como manter meu software sempre pronto para produção? Quais práticas utilizar? Quais não utilizar? Quais os benefícios? Embutir qualidade no processo de desenvolvimento e antecipar riscos é potencialmente o melhor investimento a ser feito no seu software!

> **Citações**:

* *"A working software application can be usefully decomposed into four components: executable code, configuration, host environment, and data."*

* *"Configuration management refers to the process by which all artifacts relevant to your project, and the relationships between them, are stored, retrieved, uniquely identified, and modified."*

* *"In software, when something is painful, the way to reduce the pain is to do it more frequently, not less."*

* *"It should always be cheaper to create a new environment than to repair an old one"*

* *"The repeatability and reliability derive from two principles: automate almost everything, and keep everything you need to build, deploy, test, and release your application in version control."*

**Site Reliability Engineering: How Google Runs Production Systems** (*Betsy Beyer, Chris Jones, Jennifer Petoff, Niall Richard Murphy*): Coletanêa de artigos do time de SRE do Google, ilustrando a origem do termo, cultura, princípios e práticas internas, da formação de time até valiosas lições de como potencializar o feedback de sistemas em produção para o desenvolvimento - e sem deixar de lado conceitos como gerenciamento de mudança, monitoramento, planejamento de capacidade e resposta a incidentes.

> **Citações**:

* *"Software engineering has this in common with having children: the labor before the birth is painful and difficult, but the labor after the birth is where you actually spend most of your effort."*

* *"SRE is what happens when you ask a software engineer to design an operations team."*

* *"One could equivalently view SRE as a specific implementation of DevOps with some idiosyncratic extensions."*

* *"The most relevant metric in evaluating the effectiveness of emergency response is how quickly the response team can bring the system back to health — that is, the MTTR."* 

> **Notas**:

- Disponível gratuitamente [online](https://landing.google.com/sre/book/index.html)

- [Review completo por Fernando Ike](https://medium.com/@fernandoike/site-reliability-engineer-sre-b9440f02ca26)
