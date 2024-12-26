---
title: Mocks não são Stubs
description: Um guia completo sobre teste doubles
---
# Mocks não são Stubs (Mocks Aren't Stubs)

O termo 'Mock Objects' tornou-se popular para descrever objetos de caso especial
que imitam objetos reais para teste. A maioria dos ambientes de linguagem agora
tem frameworks que facilitam a criação de mock objects. No entanto, o que muitas
vezes não é percebido é que os mock objects são apenas uma forma de objeto de
teste de caso especial, que permite um estilo diferente de teste. Neste artigo,
explicarei como os mock objects funcionam, como eles incentivam o teste baseado
na verificação de comportamento e como a comunidade ao seu redor os usa para
desenvolver um estilo diferente de teste.

02 de janeiro de 2007

Foto de Martin Fowler Martin Fowler popular

teste

## Conteúdo

- [Mocks não são Stubs (Mocks Aren't
  Stubs)](#mocks-não-são-stubs-mocks-arent-stubs)
  - [Conteúdo](#conteúdo)
  - [Testes Regulares](#testes-regulares)
  - [Testes com Mock Objects](#testes-com-mock-objects)
  - [Usando EasyMock](#usando-easymock)
  - [A Diferença Entre Mocks e Stubs](#a-diferença-entre-mocks-e-stubs)
  - [Teste Clássico e Mockista](#teste-clássico-e-mockista)
  - [Escolhendo Entre as Diferenças](#escolhendo-entre-as-diferenças)
  - [Conduzindo TDD](#conduzindo-tdd)
  - [Configuração de Fixture](#configuração-de-fixture)
  - [Isolamento de Teste](#isolamento-de-teste)
  - [Acoplando Testes a Implementações](#acoplando-testes-a-implementações)
  - [Estilo de Design](#estilo-de-design)
  - [Então, devo ser um classicista ou um
    mockista?](#então-devo-ser-um-classicista-ou-um-mockista)
  - [Considerações Finais](#considerações-finais)
  - [Leitura adicional](#leitura-adicional)

Eu encontrei o termo “mock object” pela primeira vez há alguns anos na
comunidade Extreme Programming (XP). Desde então, tenho me deparado com mock
objects cada vez mais. Em parte, isso ocorre porque muitos dos principais
desenvolvedores de mock objects foram meus colegas na Thoughtworks em vários
momentos. Em parte, é porque eu os vejo cada vez mais na literatura de testes
influenciada pelo XP.

Mas, com frequência, vejo mock objects descritos de forma inadequada. Em
particular, eu os vejo frequentemente confundidos com stubs - um auxiliar comum
para ambientes de teste. Eu entendo essa confusão - eu os via como semelhantes
por um tempo também, mas as conversas com os desenvolvedores de mock permitiram
que um pequeno entendimento de mock penetrasse em meu crânio de casca de
tartaruga.

Essa diferença é, na verdade, duas diferenças separadas. Por um lado, há uma
diferença em como os resultados dos testes são verificados: uma distinção entre
verificação de estado e verificação de comportamento. Por outro lado, há uma
filosofia totalmente diferente sobre a maneira como os testes e o design
interagem, que eu chamo aqui de estilos clássico e mockista de Desenvolvimento
Orientado a Testes (Test Driven Development).

## Testes Regulares

Vou começar ilustrando os dois estilos com um exemplo simples. (O exemplo está
em Java, mas os princípios fazem sentido com qualquer linguagem orientada a
objetos.) Queremos pegar um objeto de pedido (order) e preenchê-lo a partir de
um objeto de depósito (warehouse). O pedido é muito simples, com apenas um
produto e uma quantidade. O depósito armazena estoques de diferentes produtos.
Quando pedimos a um pedido para se preencher a partir de um depósito, há duas
respostas possíveis. Se houver produto suficiente no depósito para atender ao
pedido, o pedido será preenchido e a quantidade do produto no depósito será
reduzida na quantidade apropriada. Se não houver produto suficiente no depósito,
o pedido não será preenchido e nada acontecerá no depósito.

Esses dois comportamentos implicam alguns testes, que se parecem com testes
JUnit bastante convencionais.

```java
public class OrderStateTester extends TestCase {
  private static String TALISKER = "Talisker";
  private static String HIGHLAND_PARK = "Highland Park";
  private Warehouse warehouse = new WarehouseImpl();

  protected void setUp() throws Exception {
    warehouse.add(TALISKER, 50);
    warehouse.add(HIGHLAND_PARK, 25);
  }
  public void testOrderIsFilledIfEnoughInWarehouse() {
    Order order = new Order(TALISKER, 50);
    order.fill(warehouse);
    assertTrue(order.isFilled());
    assertEquals(0, warehouse.getInventory(TALISKER));
  }
  public void testOrderDoesNotRemoveIfNotEnough() {
    Order order = new Order(TALISKER, 51);
    order.fill(warehouse);
    assertFalse(order.isFilled());
    assertEquals(50, warehouse.getInventory(TALISKER));
  }
}
```

Os testes xUnit seguem uma sequência típica de quatro fases: configuração
(setup), exercício (exercise), verificação (verify) e desmontagem (teardown).
Neste caso, a fase de configuração é feita em parte no método setUp
(configurando o depósito) e em parte no método de teste (configurando o pedido).
A chamada para order.fill é a fase de exercício. É aqui que o objeto é
estimulado a fazer a coisa que queremos testar. As instruções assert são então o
estágio de verificação, verificando se o método exercido executou sua tarefa
corretamente. Neste caso, não há fase explícita de desmontagem, o coletor de
lixo faz isso por nós implicitamente.

Durante a configuração, existem dois tipos de objetos que estamos montando.
Order é a classe que estamos testando, mas para que Order.fill funcione, também
precisamos de uma instância de Warehouse. Nessa situação, Order é o objeto em
que estamos focados em testar. As pessoas orientadas a testes gostam de usar
termos como objeto sob teste (object-under-test) ou sistema sob teste
(system-under-test) para nomear tal coisa. Qualquer um dos termos é um palavrão
feio de dizer, mas como é um termo amplamente aceito, vou tapar o nariz e
usá-lo. Seguindo Meszaros, usarei Sistema Sob Teste (System Under Test), ou
melhor, a abreviação SUT.

Portanto, para este teste, preciso do SUT (Order) e de um colaborador
(warehouse). Preciso do depósito por dois motivos: um é para fazer o
comportamento testado funcionar (já que Order.fill chama os métodos do
warehouse) e, em segundo lugar, preciso dele para verificação (já que um dos
resultados de Order.fill é uma mudança potencial no estado do warehouse). À
medida que exploramos este tópico mais a fundo, você verá que faremos muita
distinção entre SUT e colaboradores. (Na versão anterior deste artigo, eu me
referia ao SUT como o “objeto primário” e aos colaboradores como “objetos
secundários”)

Este estilo de teste usa verificação de estado: o que significa que determinamos
se o método exercido funcionou corretamente examinando o estado do SUT e de seus
colaboradores após o método ter sido exercido. Como veremos, os mock objects
permitem uma abordagem diferente para a verificação.

## Testes com Mock Objects

Agora vou pegar o mesmo comportamento e usar mock objects. Para este código,
estou usando a biblioteca jMock para definir mocks. jMock é uma biblioteca de
mock objects Java. Existem outras bibliotecas de mock objects por aí, mas esta é
uma biblioteca atualizada escrita pelos criadores da técnica, então é uma boa
para começar.

```java
public class OrderInteractionTester extends MockObjectTestCase {
  private static String TALISKER = "Talisker";

  public void testFillingRemovesInventoryIfInStock() {
    //setup - data
    Order order = new Order(TALISKER, 50);
    Mock warehouseMock = new Mock(Warehouse.class);
    
    //setup - expectations
    warehouseMock.expects(once()).method("hasInventory")
      .with(eq(TALISKER),eq(50))
      .will(returnValue(true));
    warehouseMock.expects(once()).method("remove")
      .with(eq(TALISKER), eq(50))
      .after("hasInventory");

    //exercise
    order.fill((Warehouse) warehouseMock.proxy());
    
    //verify
    warehouseMock.verify();
    assertTrue(order.isFilled());
  }

  public void testFillingDoesNotRemoveIfNotEnoughInStock() {
    Order order = new Order(TALISKER, 51);    
    Mock warehouse = mock(Warehouse.class);
      
    warehouse.expects(once()).method("hasInventory")
      .withAnyArguments()
      .will(returnValue(false));

    order.fill((Warehouse) warehouse.proxy());

    assertFalse(order.isFilled());
  }
}
```

Concentre-se em testFillingRemovesInventoryIfInStock primeiro, pois eu usei
alguns atalhos com o teste posterior.

Para começar, a fase de configuração é muito diferente. Para começar, ela é
dividida em duas partes: dados e expectativas. A parte de dados configura os
objetos com os quais estamos interessados em trabalhar, nesse sentido é
semelhante à configuração tradicional. A diferença está nos objetos que são
criados. O SUT é o mesmo - um pedido. No entanto, o colaborador não é um objeto
de depósito, mas sim um depósito mock - tecnicamente uma instância da classe
Mock.

A segunda parte da configuração cria expectativas no mock object. As
expectativas indicam quais métodos devem ser chamados nos mocks quando o SUT é
exercido.

Depois que todas as expectativas estão em vigor, eu exerço o SUT. Após o
exercício, faço a verificação, que tem dois aspectos. Eu executo asserts no SUT
- como antes. No entanto, também verifico os mocks - verificando se eles foram
chamados de acordo com suas expectativas.

A principal diferença aqui é como verificamos se o pedido fez a coisa certa em
sua interação com o depósito. Com a verificação de estado, fazemos isso por meio
de asserts no estado do depósito. Os mocks usam verificação de comportamento,
onde, em vez disso, verificamos se o pedido fez as chamadas corretas no
depósito. Fazemos essa verificação dizendo ao mock o que esperar durante a
configuração e pedindo ao mock para se verificar durante a verificação. Apenas o
pedido é verificado usando asserts, e se o método não alterar o estado do
pedido, não haverá asserts.

No segundo teste, faço algumas coisas diferentes. Primeiro, crio o mock de forma
diferente, usando o método mock em MockObjectTestCase em vez do construtor. Este
é um método de conveniência na biblioteca jMock que significa que não preciso
chamar explicitamente verify mais tarde, qualquer mock criado com o método de
conveniência é verificado automaticamente no final do teste. Eu poderia ter
feito isso no primeiro teste também, mas queria mostrar a verificação mais
explicitamente para mostrar como funciona o teste com mocks.

A segunda coisa diferente no segundo caso de teste é que relaxei as restrições
na expectativa usando withAnyArguments. A razão para isso é que o primeiro teste
verifica se o número é passado para o depósito, então o segundo teste não
precisa repetir esse elemento do teste. Se a lógica do pedido precisar ser
alterada posteriormente, apenas um teste falhará, facilitando o esforço de
migração dos testes. Acontece que eu poderia ter deixado withAnyArguments de
fora inteiramente, pois esse é o padrão.

## Usando EasyMock

Existem várias bibliotecas de mock objects por aí. Uma com a qual me deparo
bastante é o EasyMock, tanto em suas versões Java quanto .NET. O EasyMock também
permite a verificação de comportamento, mas tem algumas diferenças de estilo com
o jMock que valem a pena discutir. Aqui estão os testes familiares novamente:

```java
public class OrderEasyTester extends TestCase {
  private static String TALISKER = "Talisker";
  
  private MockControl warehouseControl;
  private Warehouse warehouseMock;
  
  public void setUp() {
    warehouseControl = MockControl.createControl(Warehouse.class);
    warehouseMock = (Warehouse) warehouseControl.getMock();    
  }

  public void testFillingRemovesInventoryIfInStock() {
    //setup - data
    Order order = new Order(TALISKER, 50);
    
    //setup - expectations
    warehouseMock.hasInventory(TALISKER, 50);
    warehouseControl.setReturnValue(true);
    warehouseMock.remove(TALISKER, 50);
    warehouseControl.replay();

    //exercise
    order.fill(warehouseMock);
    
    //verify
    warehouseControl.verify();
    assertTrue(order.isFilled());
  }

  public void testFillingDoesNotRemoveIfNotEnoughInStock() {
    Order order = new Order(TALISKER, 51);    

    warehouseMock.hasInventory(TALISKER, 51);
    warehouseControl.setReturnValue(false);
    warehouseControl.replay();

    order.fill((Warehouse) warehouseMock);

    assertFalse(order.isFilled());
    warehouseControl.verify();
  }
}
```

O EasyMock usa uma metáfora de gravar/reproduzir (record/replay) para definir as
expectativas. Para cada objeto que você deseja mockar, você cria um controle e
um mock object. O mock satisfaz a interface do objeto secundário, o controle
oferece recursos adicionais. Para indicar uma expectativa, você chama o método,
com os argumentos que espera no mock. Você segue isso com uma chamada para o
controle se quiser um valor de retorno. Depois de terminar de definir as
expectativas, você chama replay no controle - momento em que o mock termina a
gravação e está pronto para responder ao objeto primário. Depois de pronto, você
chama verify no controle.

Parece que, embora as pessoas muitas vezes fiquem confusas à primeira vista com
a metáfora de gravar/reproduzir, elas rapidamente se acostumam com ela. Ela tem
uma vantagem sobre as restrições do jMock, pois você está fazendo chamadas de
método reais para o mock em vez de especificar nomes de métodos em strings. Isso
significa que você pode usar o preenchimento de código em seu IDE e qualquer
refatoração de nomes de métodos atualizará automaticamente os testes. A
desvantagem é que você não pode ter as restrições mais flexíveis.

Os desenvolvedores do jMock estão trabalhando em uma nova versão que usará
outras técnicas para permitir que você use chamadas de método reais.

## A Diferença Entre Mocks e Stubs

Quando foram introduzidos pela primeira vez, muitas pessoas confundiram
facilmente os mock objects com a noção comum de teste de usar stubs. Desde
então, parece que as pessoas entenderam melhor as diferenças (e espero que a
versão anterior deste artigo tenha ajudado). No entanto, para entender
completamente a maneira como as pessoas usam os mocks, é importante entender os
mocks e outros tipos de dublês de teste. (“dublês”? Não se preocupe se este for
um termo novo para você, espere alguns parágrafos e tudo ficará claro.)

Quando você está fazendo testes como este, você está se concentrando em um
elemento do software por vez - daí o termo comum teste de unidade. O problema é
que, para fazer uma única unidade funcionar, muitas vezes você precisa de outras
unidades - daí a necessidade de algum tipo de depósito em nosso exemplo.

Nos dois estilos de teste que mostrei acima, o primeiro caso usa um objeto de
depósito real e o segundo caso usa um depósito mock, que obviamente não é um
objeto de depósito real. Usar mocks é uma maneira de não usar um depósito real
no teste, mas existem outras formas de objetos irreais usados em testes como
este.

O vocabulário para falar sobre isso logo fica confuso - todos os tipos de
palavras são usados: stub, mock, fake, dummy. Para este artigo, vou seguir o
vocabulário do livro de Gerard Meszaros. Não é o que todo mundo usa, mas acho
que é um bom vocabulário e, como é meu ensaio, posso escolher quais palavras
usar.

Meszaros usa o termo Dublê de Teste (Test Double) como o termo genérico para
qualquer tipo de objeto de simulação usado no lugar de um objeto real para fins
de teste. O nome vem da noção de Dublê de Corpo (Stunt Double) em filmes. (Um de
seus objetivos era evitar o uso de qualquer nome que já fosse amplamente
utilizado.) Meszaros então definiu cinco tipos particulares de dublês:

-   Objetos **Dummy** são passados, mas nunca realmente usados. Normalmente,
    eles são usados apenas para preencher listas de parâmetros.
-   Objetos **Fake** realmente têm implementações funcionais, mas geralmente
    usam algum atalho que os torna inadequados para produção (um banco de dados
    na memória é um bom exemplo).
-   **Stubs** fornecem respostas prontas para chamadas feitas durante o teste,
    geralmente não respondendo a nada fora do que está programado para o teste.
-   **Spies** são stubs que também registram algumas informações com base em
    como foram chamados. Uma forma disso pode ser um serviço de e-mail que
    registra quantas mensagens foram enviadas.
-   **Mocks** são o que estamos falando aqui: objetos pré-programados com
    expectativas que formam uma especificação das chamadas que se espera que
    recebam.

Desses tipos de dublês, apenas os mocks insistem na verificação de
comportamento. Os outros dublês podem, e geralmente usam, verificação de estado.
Na verdade, os mocks se comportam como outros dublês durante a fase de
exercício, pois precisam fazer o SUT acreditar que está conversando com seus
colaboradores reais - mas os mocks diferem nas fases de configuração e
verificação.

Para explorar um pouco mais os dublês de teste, precisamos estender nosso
exemplo. Muitas pessoas só usam um dublê de teste se o objeto real for difícil
de trabalhar. Um caso mais comum para um dublê de teste seria se disséssemos que
queríamos enviar uma mensagem de e-mail se não conseguíssemos preencher um
pedido. O problema é que não queremos enviar mensagens de e-mail reais para os
clientes durante o teste. Então, em vez disso, criamos um dublê de teste do
nosso sistema de e-mail, um que podemos controlar e manipular.

Aqui podemos começar a ver a diferença entre mocks e stubs. Se estivéssemos
escrevendo um teste para esse comportamento de envio de e-mail, poderíamos
escrever um stub simples como este.

```java
public interface MailService {
  public void send (Message msg);
}
public class MailServiceStub implements MailService {
  private List<Message> messages = new ArrayList<Message>();
  public void send (Message msg) {
    messages.add(msg);
  }
  public int numberSent() {
    return messages.size();
  }
}       
```

Podemos então usar a verificação de estado no stub como este.

```java
class OrderStateTester...

  public void testOrderSendsMailIfUnfilled() {
    Order order = new Order(TALISKER, 51);
    MailServiceStub mailer = new MailServiceStub();
    order.setMailer(mailer);
    order.fill(warehouse);
    assertEquals(1, mailer.numberSent());
  }
```

Claro, este é um teste muito simples - apenas que uma mensagem foi enviada. Não
testamos se foi enviada para a pessoa certa ou com o conteúdo certo, mas servirá
para ilustrar o ponto.

Usando mocks, este teste ficaria bem diferente.

```java
class OrderInteractionTester...

  public void testOrderSendsMailIfUnfilled() {
    Order order = new Order(TALISKER, 51);
    Mock warehouse = mock(Warehouse.class);
    Mock mailer = mock(MailService.class);
    order.setMailer((MailService) mailer.proxy());

    mailer.expects(once()).method("send");
    warehouse.expects(once()).method("hasInventory")
      .withAnyArguments()
      .will(returnValue(false));

    order.fill((Warehouse) warehouse.proxy());
  }
}
```

Em ambos os casos, estou usando um dublê de teste em vez do serviço de e-mail
real. Há uma diferença em que o stub usa verificação de estado enquanto o mock
usa verificação de comportamento.

Para usar a verificação de estado no stub, preciso fazer alguns métodos extras
no stub para ajudar na verificação. Como resultado, o stub implementa
MailService, mas adiciona métodos de teste extras.

Mock objects sempre usam verificação de comportamento, um stub pode ir para
qualquer lado. Meszaros se refere a stubs que usam verificação de comportamento
como um Espião de Teste (Test Spy). A diferença está em como exatamente o dublê
executa e verifica e deixarei isso para você explorar por conta própria.

## Teste Clássico e Mockista

Agora estou no ponto em que posso explorar a segunda dicotomia: aquela entre TDD
clássico e mockista. O grande problema aqui é quando usar um mock (ou outro
dublê).

O estilo clássico de TDD é usar objetos reais, se possível, e um dublê se for
difícil usar a coisa real. Portanto, um TDDer clássico usaria um depósito real e
um dublê para o serviço de e-mail. O tipo de dublê realmente não importa muito.

Um praticante de TDD mockista, no entanto, sempre usará um mock para qualquer
objeto com comportamento interessante. Neste caso, tanto para o depósito quanto
para o serviço de e-mail.

Embora os vários frameworks de mock tenham sido projetados com o teste mockista
em mente, muitos classicistas os consideram úteis para criar dublês.

Um desdobramento importante do estilo mockista é o Desenvolvimento Orientado a
Comportamento (Behavior Driven Development - BDD). O BDD foi originalmente
desenvolvido por meu colega Daniel Terhorst-North como uma técnica para ajudar
melhor as pessoas a aprender Desenvolvimento Orientado a Testes, concentrando-se
em como o TDD opera como uma técnica de design. Isso levou à renomeação de
testes como comportamentos para explorar melhor onde o TDD ajuda a pensar sobre
o que um objeto precisa fazer. O BDD adota uma abordagem mockista, mas se
expande sobre isso, tanto com seus estilos de nomeação quanto com seu desejo de
integrar a análise em sua técnica. Não vou entrar nisso mais aqui, pois a única
relevância para este artigo é que o BDD é outra variação do TDD que tende a usar
testes mockistas. Deixarei para você seguir o link para obter mais informações.

Às vezes, você vê o estilo “Detroit” usado para “clássico” e “Londres” para
“mockista”. Isso alude ao fato de que o XP foi originalmente desenvolvido com o
projeto C3 em Detroit e o estilo mockista foi desenvolvido pelos primeiros
adeptos do XP em Londres. Devo também mencionar que muitos TDDers mockistas não
gostam desse termo e, na verdade, de qualquer terminologia que implique um
estilo diferente entre os testes clássicos e mockistas. Eles não consideram que
haja uma distinção útil a ser feita entre os dois estilos.

## Escolhendo Entre as Diferenças

Neste artigo, expliquei um par de diferenças: verificação de estado ou
comportamento / TDD clássico ou mockista. Quais são os argumentos a ter em mente
ao fazer as escolhas entre eles? Vou começar com a escolha da verificação de
estado versus comportamento.

A primeira coisa a considerar é o contexto. Estamos pensando em uma colaboração
fácil, como pedido e depósito, ou uma difícil, como pedido e serviço de e-mail?

Se for uma colaboração fácil, a escolha é simples. Se eu for um TDDer clássico,
não uso um mock, stub ou qualquer tipo de dublê. Eu uso um objeto real e
verificação de estado. Se eu for um TDDer mockista, uso um mock e verificação de
comportamento. Nenhuma decisão.

Se for uma colaboração difícil, então não há decisão se eu for um mockista - eu
apenas uso mocks e verificação de comportamento. Se eu for um classicista, então
tenho uma escolha, mas não é grande coisa qual usar. Normalmente, os
classicistas decidem caso a caso, usando a rota mais fácil para cada situação.

Então, como vemos, a verificação de estado versus comportamento não é uma grande
decisão. O verdadeiro problema é entre TDD clássico e mockista. Acontece que as
características da verificação de estado e comportamento afetam essa discussão,
e é aí que vou concentrar a maior parte da minha energia.

Mas antes de fazer isso, deixe-me apresentar um caso extremo. Ocasionalmente,
você se depara com coisas que são realmente difíceis de usar a verificação de
estado, mesmo que não sejam colaborações difíceis. Um ótimo exemplo disso é um
cache. O objetivo de um cache é que você não pode dizer pelo seu estado se o
cache acertou ou errou - este é um caso em que a verificação de comportamento
seria a escolha sábia até mesmo para um TDDer clássico obstinado. Tenho certeza
de que existem outras exceções em ambas as direções.

À medida que nos aprofundamos na escolha clássico/mockista, há muitos fatores a
considerar, então eu os dividi em grupos aproximados.

## Conduzindo TDD

Os mock objects surgiram da comunidade XP, e uma das principais características
do XP é sua ênfase no Desenvolvimento Orientado a Testes - onde o design de um
sistema é desenvolvido por meio de iteração conduzida pela escrita de testes.

Portanto, não é surpresa que os mockistas falem particularmente sobre o efeito
dos testes mockistas em um design. Em particular, eles defendem um estilo
chamado desenvolvimento orientado a necessidades (need-driven development). Com
esse estilo, você começa a desenvolver uma história de usuário escrevendo seu
primeiro teste para o exterior do seu sistema, tornando algum objeto de
interface seu SUT. Ao refletir sobre as expectativas dos colaboradores, você
explora a interação entre o SUT e seus vizinhos - efetivamente projetando a
interface de saída do SUT.

Depois de executar seu primeiro teste, as expectativas nos mocks fornecem uma
especificação para a próxima etapa e um ponto de partida para os testes. Você
transforma cada expectativa em um teste em um colaborador e repete o processo
trabalhando em seu caminho para o sistema, um SUT por vez. Esse estilo também é
conhecido como outside-in, que é um nome muito descritivo para ele. Funciona bem
com sistemas em camadas. Você primeiro começa programando a UI usando camadas
mock por baixo. Em seguida, você escreve testes para a camada inferior,
percorrendo gradualmente o sistema uma camada por vez. Essa é uma abordagem
muito estruturada e controlada, que muitas pessoas acreditam ser útil para
orientar os recém-chegados a OO e TDD.

O TDD clássico não fornece exatamente a mesma orientação. Você pode fazer uma
abordagem de etapas semelhante, usando métodos com stubs em vez de mocks. Para
fazer isso, sempre que precisar de algo de um colaborador, basta codificar
exatamente a resposta que o teste requer para fazer o SUT funcionar. Então,
quando você estiver verde com isso, você substitui a resposta codificada por um
código adequado.

Mas o TDD clássico também pode fazer outras coisas. Um estilo comum é o
middle-out. Nesse estilo, você pega um recurso e decide o que precisa no domínio
para que esse recurso funcione. Você faz com que os objetos de domínio façam o
que você precisa e, depois que eles estiverem funcionando, você coloca a UI em
camadas por cima. Fazendo isso, você pode nunca precisar fingir nada. Muitas
pessoas gostam disso porque concentra a atenção no modelo de domínio primeiro, o
que ajuda a evitar que a lógica de domínio vaze para a UI.

Devo enfatizar que tanto os mockistas quanto os classicistas fazem isso uma
história por vez. Há uma escola de pensamento que constrói aplicativos camada
por camada, não iniciando uma camada até que outra esteja concluída. Tanto os
classicistas quanto os mockistas tendem a ter um histórico ágil e preferem
iterações refinadas. Como resultado, eles trabalham recurso por recurso, em vez
de camada por camada.

## Configuração de Fixture

Com o TDD clássico, você deve criar não apenas o SUT, mas também todos os
colaboradores de que o SUT precisa em resposta ao teste. Embora o exemplo tenha
apenas alguns objetos, os testes reais geralmente envolvem uma grande quantidade
de objetos secundários. Normalmente, esses objetos são criados e desmontados a
cada execução dos testes.

Os testes mockistas, no entanto, só precisam criar o SUT e os mocks para seus
vizinhos imediatos. Isso pode evitar parte do trabalho envolvido na construção
de fixtures complexas (pelo menos em teoria. Já me deparei com histórias de
configurações de mock bastante complexas, mas isso pode ser devido ao fato de
não usar bem as ferramentas).

Na prática, os testadores clássicos tendem a reutilizar fixtures complexas o
máximo possível. Da maneira mais simples, você faz isso colocando o código de
configuração da fixture no método de configuração xUnit. Fixtures mais
complicadas precisam ser usadas por várias classes de teste, então, neste caso,
você cria classes especiais de geração de fixtures. Eu costumo chamá-las de
Object Mothers, com base em uma convenção de nomenclatura usada em um projeto XP
inicial da Thoughtworks. Usar mothers é essencial em testes clássicos maiores,
mas as mothers são código adicional que precisa ser mantido e qualquer alteração
nas mothers pode ter efeitos cascata significativos nos testes. Também pode
haver um custo de desempenho na configuração da fixture - embora eu não tenha
ouvido falar que isso seja um problema sério quando feito corretamente. A
maioria dos objetos de fixture são baratos para criar, aqueles que não são
geralmente são duplicados.

Como resultado, ouvi ambos os estilos acusarem o outro de dar muito trabalho. Os
mockistas dizem que criar as fixtures dá muito trabalho, mas os classicistas
dizem que isso é reutilizado, mas você precisa criar mocks a cada teste.

## Isolamento de Teste

Se você introduzir um bug em um sistema com testes mockistas, geralmente fará
com que apenas os testes cujo SUT contenha o bug falhem. Com a abordagem
clássica, no entanto, quaisquer testes de objetos de cliente também podem
falhar, o que leva a falhas em que o objeto com bugs é usado como colaborador no
teste de outro objeto. Como resultado, uma falha em um objeto muito usado causa
uma onda de testes com falha em todo o sistema.

Os testadores mockistas consideram isso um grande problema; isso resulta em
muita depuração para encontrar a raiz do erro e corrigi-lo. No entanto, os
classicistas não expressam isso como uma fonte de problemas. Normalmente, o
culpado é relativamente fácil de identificar observando quais testes falham e os
desenvolvedores podem dizer que outras falhas são derivadas da falha raiz. Além
disso, se você estiver testando regularmente (como deveria), saberá que a quebra
foi causada pelo que você editou por último, então não é difícil encontrar a
falha.

Um fator que pode ser significativo aqui é a granularidade dos testes. Como os
testes clássicos exercitam vários objetos reais, muitas vezes você encontra um
único teste como o teste principal para um cluster de objetos, em vez de apenas
um. Se esse cluster abranger muitos objetos, pode ser muito mais difícil
encontrar a verdadeira fonte de um bug. O que está acontecendo aqui é que os
testes são muito granulares.

É bem provável que os testes mockistas tenham menos probabilidade de sofrer com
esse problema, porque a convenção é mockar todos os objetos além do primário, o
que deixa claro que testes mais granulares são necessários para os
colaboradores. Dito isso, também é verdade que usar testes excessivamente
granulares não é necessariamente uma falha do teste clássico como uma técnica,
mas sim uma falha em fazer o teste clássico corretamente. Uma boa regra é
garantir que você separe testes granulares para cada classe. Embora os clusters
às vezes sejam razoáveis, eles devem ser limitados a apenas alguns objetos - não
mais do que meia dúzia. Além disso, se você se deparar com um problema de
depuração devido a testes excessivamente granulares, você deve depurar de forma
orientada a testes, criando testes mais granulares conforme avança.

Em essência, os testes xUnit clássicos não são apenas testes de unidade, mas
também minitestes de integração. Como resultado, muitas pessoas gostam do fato
de que os testes de cliente podem detectar erros que os testes principais de um
objeto podem ter perdido, principalmente investigando áreas onde as classes
interagem. Os testes mockistas perdem essa qualidade. Além disso, você também
corre o risco de que as expectativas nos testes mockistas possam estar
incorretas, resultando em testes de unidade que ficam verdes, mas mascaram erros
inerentes.

É neste ponto que devo enfatizar que, independentemente do estilo de teste que
você usa, você deve combiná-lo com testes de aceitação mais granulares que
operam em todo o sistema como um todo. Muitas vezes me deparei com projetos que
demoraram a usar testes de aceitação e se arrependeram.

## Acoplando Testes a Implementações

Quando você escreve um teste mockista, você está testando as chamadas de saída
do SUT para garantir que ele converse adequadamente com seus fornecedores. Um
teste clássico se preocupa apenas com o estado final - não como esse estado foi
derivado. Os testes mockistas são, portanto, mais acoplados à implementação de
um método. Alterar a natureza das chamadas para os colaboradores geralmente faz
com que um teste mockista quebre.

Esse acoplamento leva a algumas preocupações. A mais importante é o efeito no
Desenvolvimento Orientado a Testes. Com os testes mockistas, escrever o teste
faz você pensar sobre a implementação do comportamento - na verdade, os
testadores mockistas veem isso como uma vantagem. Os classicistas, no entanto,
acham que é importante pensar apenas no que acontece na interface externa e
deixar toda a consideração da implementação para depois que você terminar de
escrever o teste.

O acoplamento à implementação também interfere na refatoração, uma vez que as
mudanças na implementação têm muito mais probabilidade de quebrar os testes do
que nos testes clássicos.

Isso pode ser agravado pela natureza dos kits de ferramentas de mock. Muitas
vezes, as ferramentas de mock especificam chamadas de método e correspondências
de parâmetros muito específicos, mesmo quando eles não são relevantes para este
teste específico. Um dos objetivos do kit de ferramentas jMock é ser mais
flexível em sua especificação das expectativas para permitir que as expectativas
sejam mais flexíveis em áreas onde não importa, ao custo de usar strings que
podem tornar a refatoração mais complicada.

## Estilo de Design

Um dos aspectos mais fascinantes desses estilos de teste para mim é como eles
afetam as decisões de design. Ao conversar com os dois tipos de testadores,
tomei conhecimento de algumas diferenças entre os designs que os estilos
incentivam, mas tenho certeza de que estou apenas arranhando a superfície.

Já mencionei uma diferença no tratamento de camadas. O teste mockista suporta
uma abordagem outside-in, enquanto os desenvolvedores que preferem um estilo de
modelo de domínio tendem a preferir o teste clássico.

Em um nível menor, notei que os testadores mockistas tendem a se afastar de
métodos que retornam valores, em favor de métodos que atuam em um objeto de
coleta. Pegue o exemplo do comportamento de coletar informações de um grupo de
objetos para criar uma string de relatório. Uma maneira comum de fazer isso é
fazer com que o método de relatório chame métodos de retorno de string nos
vários objetos e monte a string resultante em uma variável temporária. Um
testador mockista provavelmente passaria um buffer de string para os vários
objetos e faria com que eles adicionassem as várias strings ao buffer - tratando
o buffer de string como um parâmetro de coleta.

Os testadores mockistas falam mais sobre evitar 'acidentes de trem' - cadeias de
métodos do estilo getThis().getThat().getTheOther(). Evitar cadeias de métodos
também é conhecido como seguir a Lei de Demeter. Embora as cadeias de métodos
sejam um cheiro, o problema oposto de objetos intermediários inchados com
métodos de encaminhamento também é um cheiro. (Sempre achei que ficaria mais
confortável com a Lei de Demeter se ela fosse chamada de Sugestão de Demeter.)

Uma das coisas mais difíceis para as pessoas entenderem no design OO é o
princípio “Diga, não pergunte” (Tell Don't Ask), que o encoraja a dizer a um
objeto para fazer algo em vez de extrair dados de um objeto para fazê-lo no
código do cliente. Os mockistas dizem que usar testes mockistas ajuda a promover
isso e evitar o confete getter que permeia muito do código hoje em dia. Os
classicistas argumentam que existem muitas outras maneiras de fazer isso.

Um problema reconhecido com a verificação baseada em estado é que ela pode levar
à criação de métodos de consulta apenas para suportar a verificação. Nunca é
confortável adicionar métodos à API de um objeto puramente para teste, usar a
verificação de comportamento evita esse problema. O contra-argumento para isso é
que tais modificações geralmente são pequenas na prática.

Os mockistas favorecem interfaces de função e afirmam que usar esse estilo de
teste incentiva mais interfaces de função, uma vez que cada colaboração é
mockada separadamente e, portanto, é mais provável que seja transformada em uma
interface de função. Portanto, em meu exemplo acima, usando um buffer de string
para gerar um relatório, um mockista teria maior probabilidade de inventar uma
função específica que fizesse sentido nesse domínio, que pode ser implementada
por um buffer de string.

É importante lembrar que essa diferença no estilo de design é um motivador chave
para a maioria dos mockistas. As origens do TDD foram um desejo de obter testes
de regressão automáticos fortes que suportassem o design evolutivo. Ao longo do
caminho, seus praticantes descobriram que escrever testes primeiro melhorou
significativamente o processo de design. Os mockistas têm uma forte ideia de que
tipo de design é um bom design e desenvolveram bibliotecas de mock
principalmente para ajudar as pessoas a desenvolver esse estilo de design.

## Então, devo ser um classicista ou um mockista?

Acho esta uma pergunta difícil de responder com confiança. Pessoalmente, sempre
fui um TDDer clássico à moda antiga e, até agora, não vejo nenhuma razão para
mudar. Não vejo nenhum benefício convincente para o TDD mockista e estou
preocupado com as consequências de acoplar testes à implementação.

Isso me impressionou particularmente quando observei um programador mockista. Eu
realmente gosto do fato de que, ao escrever o teste, você se concentra no
resultado do comportamento, não em como ele é feito. Um mockista está
constantemente pensando em como o SUT será implementado para escrever as
expectativas. Isso parece realmente antinatural para mim.

Também sofro com a desvantagem de não tentar o TDD mockista em nada mais do que
brinquedos. Como aprendi com o próprio Desenvolvimento Orientado a Testes,
muitas vezes é difícil julgar uma técnica sem experimentá-la seriamente. Eu
conheço muitos bons desenvolvedores que são mockistas muito felizes e convictos.
Portanto, embora eu ainda seja um classicista convicto, prefiro apresentar os
dois argumentos da forma mais justa possível para que você possa decidir por si
mesmo.

Portanto, se o teste mockista parece atraente para você, sugiro que experimente.
Vale a pena tentar, principalmente se você estiver tendo problemas em algumas
das áreas que o TDD mockista pretende melhorar. Vejo duas áreas principais aqui.
Uma é se você está gastando muito tempo depurando quando os testes falham porque
eles não estão quebrando de forma limpa e informando onde está o problema. (Você
também pode melhorar isso usando o TDD clássico em clusters mais refinados.) A
segunda área é se seus objetos não contêm comportamento suficiente, o teste
mockista pode encorajar a equipe de desenvolvimento a criar objetos mais ricos
em comportamento.

## Considerações Finais

À medida que o interesse em testes de unidade, os frameworks xUnit e o
Desenvolvimento Orientado a Testes cresceu, mais e mais pessoas estão se
deparando com mock objects. Na maioria das vezes, as pessoas aprendem um pouco
sobre os frameworks de mock objects, sem entender completamente a divisão
mockista/clássica que os sustenta. Qualquer que seja o lado dessa divisão em que
você se incline, acho útil entender essa diferença de pontos de vista. Embora
você não precise ser um mockista para achar os frameworks de mock úteis, é útil
entender o pensamento que orienta muitas das decisões de design do software.

O objetivo deste artigo era, e é, apontar essas diferenças e apresentar as
compensações entre elas. Há mais no pensamento mockista do que tive tempo para
entrar, particularmente suas consequências no estilo de design. Espero que nos
próximos anos vejamos mais material escrito sobre isso e que isso aprofunde
nossa compreensão das fascinantes consequências de escrever testes antes do
código.

## Leitura adicional

Para uma visão geral completa da prática de teste xUnit, fique de olho no
próximo livro de Gerard Meszaros (aviso: está na minha série). Ele também mantém
um site com os padrões do livro.

Para saber mais sobre TDD, o primeiro lugar a procurar é o livro de Kent.

Para saber mais sobre o estilo mockista de teste, o melhor recurso geral é
Freeman & Pryce. Os autores cuidam do mockobjects.com. Em particular, leia o
excelente artigo da OOPSLA. Para saber mais sobre o Desenvolvimento Orientado a
Comportamento (Behavior Driven Development), uma ramificação diferente do TDD
que é muito mockista em estilo, comece com a introdução de Daniel
Terhorst-North.

Você também pode descobrir mais sobre essas técnicas consultando os sites de
ferramentas para jMock, nMock, EasyMock e o .NET EasyMock. (Existem outras
ferramentas de mock por aí, não considere esta lista completa.)

XP2000 viu o artigo original sobre mock objects, mas agora está bastante
desatualizado.

[Link Original em
inglês](https://martinfowler.com/articles/mocksArentStubs.html#RegularTests)