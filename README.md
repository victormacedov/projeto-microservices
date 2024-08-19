# Projeto Microservice

## Tecnologias utilizadas:

**Linguagem:** Java 17.

**Gerenciador de dependências:** Maven.

**Ecossistema/Framework:** Spring Framework.

\- **Spring Boot:** Iniciar os dois Microservices de negócio.

\- **Spring Web:** Para criar os endpoints.

\- **Spring Data JPA:** Responsável por fazer a modelagem e as ligações com
as base de dados.

\- **Spring Validation:** Para fazermos validações iniciais na entrada da
API.

\- **Spring AMQP:** Para trabalharmos com o protocolo de mensageria,
realizar essa comunicação assíncrona entre eles.

\- **Spring Mail:** Para enviarmos o e-mail para os usuários que forem se
cadastrar na plataforma.

**Banco de dados (Base de dados por microservices):** PostgreSQL.

**Broker:** RabbitMQ.

**Cloud:** Cloud AMQP.

**SMTP:** SMTP Gmail.

#### Para colocarmos o nosso conhecimento teórico na prática, vamos implementar dois microserviços de negócio (User Microservice e Email Microservice).

Mesmo os microserviços sendo independentes entre si e possuir bases de
dados por microserviço, elas vão precisar se comunicar de alguma forma,
para isso nós vamos implementar também todo um fluxo de comunicação
assíncrona entre eles. Lembrando que tem vários outros modelos e tipos
de comunicações entre microserviços, e quando vamos utilizar essa
comunicação assíncrona não bloqueante via mensageria, também temos
diversas maneiras, utilizando de comandos, eventos, mas neste caso vamos
utilizar comandos, onde o "User Microservice" vai produzir uma mensagem,
enviado para outro microservice que ele faça uma determinada ação.

#### Ou seja, o fluxo do RabbitMQ vai ser o seguinte:

\- Um cliente vai enviar um POST para cadastrar um usuário no sistema,
salvando na sua base de dados, e logo após vai publicar uma mensagem com
a intenção de um comando para um canal de mensagens (Broker), para
realizar então essa comunicação assíncrona e não bloqueante. Vamos
iniciar o "Email Microservice", que vai estar esperando essas mensagens
serem enviadas para que elas então possam ser consumidas, ou seja, ele
também vai estar conectado ao canal de mensagens, e quando essas
mensagens forem chegando no Broker, ele vai fazer o respectivo
roteamento para o "Email Microservice" consumir essas mensagens e
realizar então a ação que o "User Microservice" está enviando. Logo após
o "Email Microservice" consumir essa mensagem, ele vai enviar um email
de boas vindas para o usuário que acabou de ser cadastrado na plataforma
e ele vai salvar o email para que a gente tenha esses dados salvos na
arquitetura.

![](/images-readme/1.png)

#### Para implementarmos e fazemos o Broker, nós vamos utilizar o RabbitMQ.

Já que o "User Microservice" vai gerar/produzir uma mensagem, ele vai
ser denominado "Producer" e vai enviar para o Broker.

O RabbitMQ ele é formato pelas estruturas: Exchange e Queues, que variam
muito de Broker para Broker. Ou seja, o Producer envia/produz uma
mensagem para o Broker e quem recebe essa mensagem vai ser o Exchange,
responsável por analisar a mensagem e todas as suas informações, e a
partir disso vai fazer o devido roteamento para as respectivas Queues,
dessa forma essa mensagem pode ser enviada para uma fila, para mais ou
nenhuma, dependendo do tipo de Exchange que vamos utilizar.

Após isso, o "Email Microservice" vai estar conectado a essa fila e vai
sempre consumir essas mensagens, por isso é denominado "Consumer".

Vamos utilizar do CloudAMQP para realizar toda a configuração e
monitoramento do nosso RabbitMQ/Broker em Cloud.

## Criação dos Microserviços com Spring Boot

## User

Java 17, Spring Boot 3.3.2, Jar, Maven.

**Dependências utilizadas**: Spring Web, Spring Data JPA, PostgreSQL
Driver, Validation, Spring for RabbitMQ.

**Base de dados no PostgreSQL**: ms-user

**Configuração e conexão com a base de dados:**

**OBS: Vamos utilizar bases de dados por microservices**
~~~java
server.port=8081

spring.datasource.url= jdbc:postgresql://localhost:5432/ms-user
spring.datasource.username=postgres
spring.datasource.password=victormacedo
spring.jpa.hibernate.ddl-auto=update
~~~

\- Definir qual vai ser a porta que esse microserviço vai estar
disponível, para que não haja conflitos, pois se não definirmos, ele vai
levar em consideração a porta default 8080.

\- Em relação ao Spring DataSource, é para realizarmos a conexão com a
base de dados, passando sua URL, o username do admin e a senha para
conexão.

\- E o **ddl-auto** é para que todos os mapeamentos que a gente for
fazer, eles sejam convertidos em tabelas e colunas, tanto para a criação
quanto para a remoção dos mesmos, isso reflita automaticamente na base
de dados.

Podemos notar que não fizemos nenhuma configuração em relação ao nosso
Servlet Container (Tomcat), pois quando utilizamos do Spring Boot, ele
já traz embutido um Servlet Container embutido, o que facilita a o Start
dessa aplicação.

**Fazemos a mesma coisa com o nosso microserviço Email e sua base de
dados ms-email.\
OBS: As configurações continuam as mesmas, mudando somente sua base de
dados, a server port e implementando mais uma dependência (Java Mail
Sender, para realizar toda a conexão do SMTP do Gmail).**

~~~java
server.port=8082

spring.datasource.url= jdbc:postgresql://localhost:5432/ms-email
spring.datasource.username=postgres
spring.datasource.password=victormacedo
spring.jpa.hibernate.ddl-auto=update
~~~

![](/images-readme/2.png)

Explicando o fluxo do microserviços que vamos fazer até agora:

Um cliente ele vai enviar um POST para uma API no User Microservice, ele
vai receber esse novo usuário e vai salvar na base de dados.

## Save User

**- Criamos o package Models e iniciamos a classe UserModel.**

Declaramos que ele é uma Entidade e que a sua tabela se chama
"TB_USERS", aonde vai ser armazenado todos os válidos usuários
cadastrados.

Implementamos o **Serializable** para termos o controle de versões.

Passamos seus atributos, como: **ID, nome e email**. O ID vai ser do
**tipo UUID**, vai ser gerado automaticamente e vai ser a chave primária
da entidade.\
Utilizamos sempre as notations necessárias para cada um dos seus
atributos.

Geramos também o Getters e Setters de cada atributo.

**- Criamos o package Repositories e iniciamos a interface
UserRepository.**

Ele nos ajuda a utilizarmos e acessarmos todas as facilidade e métodos
que o Spring Data JPA nos oferece, passando somente a Entidade/Model que
vamos utilizar e o identificador da entidade.

~~~java
public interface UserRepository extends JpaRepository<UserModel, UUID> {}
~~~

**- Criamos o package Controllers e iniciamos a classe UserController.**

É aonde nós vamos criar o endpoint para que o cliente envie um POST
passando um novo usuário e esse microservice vai receber o usuário vai
salvar na sua base de dados.

O nosso UserController vai ser um Bean do Spring do tipo
**\@RestController**, para mostrar ao Spring que ele vai ser um Bean
gerenciado por ele.

Iniciamos o nosso POST. O cliente vai acessar para enviar um novo
usuário para ser cadastrado.

~~~java
@PostMapping("/users")
public ResponseEntity<UserModel> saveUser(@RequestBody @Valid UserRecordDto userRecordDto){
    var userModel = new UserModel();
    BeanUtils.copyProperties(userRecordDto, userModel);
    return ResponseEntity.status(HttpStatus.CREATED).body(userService.save(userModel));
}
~~~

\- Utilizamos a notation **\@PostMapping** com a URI **"/users"**.

\- O método **saveUser** é do tipo ResponseEntity, onde vamos retornar
um UserModel, contendo todos os atributos (UUID, nome, email).

[Como parâmetros, ele vai receber no corpo da requisição os dados que
vierem do userRecordDto, que vai ser um JSON do POST. E para garantir as
validações feitas no Record, é utilizado essa notation **\@Valid** no
Controller.]{.mark}

\- Recebemos um userRecordDto da classe UserRecordDto e iniciamos um
objeto da classe UserModel para fazermos a conversão do Record para
Model, para que o Model/Entidade seja salva na tabela de usuários.

Como esse userRecordDto é um JSON, seus atributos precisam se
transformar em atributos de objeto Java, ou seja, nós copiamos as
propriedades dele e passamos para esse objeto da classe UserModel que
criamos logo acima, assim nós conseguimos manipular essas informações.

Ou seja, quem vai ser salvo na base de dados vai ser nosso UserModel e
não userRecordDto/JSON que vem do POST, porém pegamos essas
informações/propriedades que vem do POST e transformamos em objeto Java,
em um userModel por exemplo, e utilizamos ela para subirmos na base de
dados.

Tendo um Model, nós precisamos de um Service para salvar esse userModel
na base de dados.

Para utilizarmos nossos métodos feitos no Service, vamos criar um ponto
de injeção para ele.

~~~java
final UserService userService;

public UserController(UserService userService) {
    this.userService = userService;
}
~~~

**- Criamos o package dtos e iniciamos o UserRecordDto do tipo Record.**

**São registros imutáveis que podemos utilizar nessas transferências de
dados.**

Aqui nós vamos mapear os atributos que recebemos do nosso POST, como nós
queremos receber ele.

Para nós fazermos o mapeamento desses atributos que recebemos do POST,
no Record todos os atributos vão estar presentes como parâmetros.
Incluímos também a validação.

~~~java
public record UserRecordDto(@NotBlank String name,
                            @NotBlank @Email String email) {

}
~~~

\- Aqui nós vamos receber o name só se ele não estiver vazio/nulo,
recebemos também só o e-mail se ele não estiver vazio/nulo e validado como um e-mail. Essas notations são do pacote jakarta.validation, que vem da dependência Validation.

**- Criamos o package Services e iniciamos a classe UserService.**

O Service interage com o Repository para acessar e manipular dados
persistidos. Ele pode executar operações de leitura e escrita no banco
de dados através do Repository. O Service contém a lógica de negócios da
aplicação. Ele é responsável por processar as requisições recebidas do
Controller, aplicar regras de negócios, realizar validações, e
orquestrar operações complexas.

Ele vai também vai ser um Bean do Spring do tipo **\@Service.**

Vamos conectar o Service ao nosso Repository, que tem acesso a toda a
estrutura do JPA. Para isso, nós vamos um ponto de injeção
userRepository dentro do Service.

Podemos fazer das seguintes formas:

\- Utilizando o **\@Autowired** e iniciamos o userRepository ou via
construtor, que foi o que eu utilizei e que a IDE que utilizei sugere
também.

~~~java
final UserRepository userRepository;

public UserService(UserRepository userRepository) {
    this.userRepository = userRepository;
}
~~~

**Após isso, nós criamos o NOSSO método save(), para que ele seja
utilizado no nosso Controller.**

~~~java
@Transactional
public UserModel save(UserModel userModel) {
    return userRepository.save(userModel);
}
~~~

\- Ele vai ter como retorno um UserModel, por isso que ele é desse tipo.

Ele vai receber o userModel e vai retornar já utilizando o
userRepository com o save do JPA passando o userModel. 
Aqui nós utilizamos o save do JPA, mas podemos utilizar outros métodos prontos para nos auxiliar na implementação.

Ou seja, o save() que utilizamos quando vamos salvar o userModel que era um userRecordDto, nós utilizamos o NOSSO save() do Service.

~~~java
return ResponseEntity.status(HttpStatus.CREATED).body(userService.save(userModel));
~~~

Por enquanto nosso save() vai fazer somente isso, porém posteriormente
ele vai publicar uma mensagem no nosso canal de mensagens via comandos
para o Broker. Por isso vamos deixar a notation **\@Transactional**,
para garantirmos o rollback, pois se algum das operações falha ou dá
algum tipo de erro, ele realiza o rollback e tudo volta ao normal.

## Criando conexão com RabbitMQ na CloudMQP e configurações na CloudMQP.

Para não precisarmos baixar nem nada, vamos utilizar do CloudMQP onde
vamos ter todo o gerenciamento do Broker.

Após fazer todo o login na plataforma, na tela principal vai ter as
instâncias dos Brokers que a gente for criando e utilizando. Iniciamos
uma instância Free com o nome "ms" e as seguintes configurações:

![](/images-readme/3.png)

Depois de criar, ao clicar na dependência, nós vamos ter a URL que vamos
inserir nos arquivos de propriedade (application.properties) para fazer
a conexão dela com os microservices já vai estar disponível.

O RabbitMQ Manager, interface onde vamos conseguir visualizar nossos
Exchanges, Queues (que ainda vamos criar e ter todo o controle de
mensagens que vão estar chegando e vão estar sendo consumidas). Vamos
ter alguns Exchanges defaults já prontos e que vamos utilizar algum
deles.

**- Vamos fazer essa configuração no Email Microservice.**

Voltando ao nosso Email Microservice, vamos iniciar a nossa conexão com
o RabbitMQ/Broker, utilizando a URL citada acima.\
Dessa forma, nós já vamos poder fazer a conexão com o nosso Broker e
posteriormente receber, enviar mensagens, etc.

Vamos iniciar também uma classe de configuração do RabbitMQ, pois vamos
ter que iniciar uma fila/queue, que vai receber essas mensagens e o
Email Microservice vai estar conectado nessa fila para consumir essas
mensagens.

Para isso vamos no "application.properties" do "Email Microservice" e
adicionamos as seguintes linhas:

~~~java
spring.rabbitmq.addresses=URL do Broker.

broker.queue.email.name=default.email
~~~

\- Adicionamos a URL do Broker\
- Definimos o nome da fila/queue que o Email Microservice vai consumir
as mensagens. Para a gente já olhar para o nome da fila e saber ao qual
Exchange pertence, podemos colocar o nome do Exchange juntamente com
Routing Key que vamos utilizar para rotear essa mensagem para uma
determinada fila.

Com isso podemos implementar nossas Configs, nossos Consumers e como
vamos receber essa mensagem via RecordDto/JSON.

**- Criamos o package Configs e iniciamos a classe RabbitMQConfig**

Aqui vai ser responsável por criarmos classes de configuração do
RabbitMQ via Spring. Utilizamos a notation **\@Configuration**.

~~~java
public class RabbitMQConfig {

    @Value("${broker.queue.email.name}")
    private String queue;

    @Bean
    public Queue queue(){
        return new Queue(queue, true);
    }
}
~~~

\- Vamos utilizar do **\@Value** passando exatamente a propriedade que
definimos no application.properties, sendo essa propriedade um atributo
(queue).\
- Assim, nós podemos iniciar essa nova fila em nosso Broker, assim vamos
utilizar de um método produtor com o **\@Bean**, conseguindo deixar ele
explícito para o Spring que Queue será criado somente quando for
necessário.\
Utilizando de new Queue, passando o nome dessa fila que será criada e se
ela é durável ou não, ou seja, se o Broker cair por um determinado
momento e voltar, essa fila vai ser preservada.

Então, quando iniciarmos a aplicação, nós vamos ter essa fila já criada
no nosso Broker/CloudMQP.

Com a fila já declarada, o Email Microservice ele vai precisar consumir
as mensagens que forem chegando em tal fila e para isso vamos ter que
criar/implementar o Consumer.

## Criando o Consumer

**- Criando package Consumer e iniciando a classe EmailConsumer.**
Ele vai ser responsável por consumir as mensagens que forem chegando na fila.

Ele também vai ser um Bean do Spring, porém vai ser mais genérico,
utilizando somente a notation **\@Component**.

~~~java
@Component
public class EmailConsumer {
    @RabbitListener(queues = "${broker.queue.email.name}")
    public void listenEmailQueue(@Payload EmailRecordDto emailRecordDto){
        System.out.println(emailRecordDto.emailTo());
    }
}
~~~

\- Iniciamos um método chamado "**listenEmailQueue**", que para que ele
seja realmente esse ouvinte, nós utilizamos a notation
**\@RabbitListener** e passando a fila que vai ser "escutada" via
properties.

\- Utilizamos também a notation **\@Payload**, pois ele vai receber o corpo dessa mensagem (JSON) que vai estar sendo enviada nessa
comunicação assíncrona.

\- E no final, printamos o email que está chegando, como uma forma para
ver se está funcionando a aplicação.

Mas o que vamos receber no Consumer vai ser um RecordDto, ou seja, temos
que configurar ele também, para posteriormente enviar e salvar todo esse
email.

**- Criando o package dtos e iniciando o record EmailRecordDto.**

Passamos os seguintes atributos para enviarmos esse email em si, sendo
eles:
~~~java
public record EmailRecordDto(UUID userId,
                             String emailTo,
                             String subject,
                             String text) {
}
~~~

Agora como vamos receber um JSON, nos precisamos transformar ele em um
"objeto Java", para que possamos manipular e salvar os dados na base de
dados.

Por isso que dentro do package Configs, nós vamos implementar o seguinte
método de conversão:

~~~java
@Bean
public Jackson2JsonMessageConverter messageConverter(){
    ObjectMapper objectMapper = new ObjectMapper();
    return new Jackson2JsonMessageConverter(objectMapper);
}
~~~

Vamos estar recebendo uma mensagem com o Payload, ou seja, o corpo da
mensagem como JSON e depois vamos estar fazendo a conversão para o
EmailRecordDto receber esse tipo em Java.\
Após isso, o nosso Consumer está finalizado.

Para o User Microservice enviar a mensagem, produzida no UserService, na
fila do Broker, nós precisamos conectar ele no RabbitMQ e passar o
endereço que está no RabbitMQ e o nome da fila, ou seja, fazer o mesmo
passo que fizemos no properties.application do Email Microservice só que
no User Microservice**.**

## Criando o Producer

**Depois de fazermos essas configurações no Email Microservice, voltamos
para o User Microservice para configurarmos nosso Producer.**

**- Criando o package Configs e iniciando a classe RabbitMQConfig.**

Já no User Microservice, nós iniciamos o package **configs**,
responsável pelas configurações do RabbitMQ, ou seja, iniciamos uma
classe chamada **RabbitMQConfig** e passamos somente a configuração do
MessageConverter que também implementamos no package configs do Email
Microservice, para que quando ele produzir a mensagem que será publicada
na fila e que será enviada para o Broker e já fique tudo pronto.

**- Criando o package Producers e iniciando a classe UserProducer.**

Como fizemos com o EmailConsumer, agora vamos fazer o UserProducer.

Ele vai ser um Bean do Spring do tipo **\@Component** e é dentro dessa
classe que vai ser feito o envio dessas mensagens para a sua respectiva
fila/Broker.

Primeiro de tudo, vamos precisar de um ponto de injeção/instância do
RabbitTemplate via construtor, é uma classe da biblioteca Spring AMQP,
que fornece uma maneira simplificada de enviar e receber mensagens em
sistemas que usam o RabbitMQ como middleware de mensagens.

~~~java
final RabbitTemplate rabbitTemplate;

public UserProducer(RabbitTemplate rabbitTemplate) {
    this.rabbitTemplate = rabbitTemplate;
}
~~~

Vamos precisar também do nome da fila que estamos utilizando, para isso
vamos utilizar novamente da notação **\@Value** passando o todo o
caminho da propriedade para acessar esse valor.\
A routingKey que vamos ter que passar para enviar essas mensagens quando
estamos utilizando Exchange do tipo default, ela tem que ser do mesmo
nome da fila que foi criada.

~~~java
@Value(value = "${broker.queue.email.name}")
private String routingKey;
~~~

E logo após isso, temos o método que convertermos e enviamos essa
mensagem, por isso que temos que ter o MessageConverter, presente no
package configs.

~~~java
public void publishMessageEmail(UserModel userModel){
    var emailDto = new EmailDto();
    emailDto.setUserId(userModel.getUserId());
    emailDto.setEmailTo(userModel.getEmail());
    emailDto.setSubject("Cadastro realizado com sucesso!");
    emailDto.setText(userModel.getName() + ", seja bem-vindo(a)! \nAgradecemos o seu cadastro, aproveite agora todos os recursos da nossa plataforma!");
    rabbitTemplate.convertAndSend("", routingKey, emailDto);
}
~~~

Para ele enviar essa mensagem, ele precisar enviar o emailDto que já
configuramos para receber no Consumer, ou seja, vamos criar ele aqui no
Producer também, pois vai ser ele que vai ser recebido do outro lado.

Para receber esse emailDto, nós vamos precisar das informações do
userModel, por isso passamos ele como parâmetro do método.

\- Ao entrar no método, nós criamos um objeto Java/uma instância do
EmailDto.

E depois que eu crio o emailDto dentro do método publishMessageEmail,
nós podemos setar as informações para configurar esse emailDto.

Feito isso, nós usamos o método convertAndSend do rabbitTemplate, que
iniciamos o ponto de injeção lá em cima, e passamos 3 informações: qual
que vai ser o Exchange que vai receber essa mensagem dentro do Broker,
qual que vai ser a routingKey que o Exchange vai fazer o
redirecionamento para a respectiva fila e qual que vai ser o corpo dessa
mensagem/qual que vai ser o emailDto.\
\
**OBS**: Como vamos utilizar o Exchange default da troca padrão, basta
passar uma String vazia que ele já automaticamente vai acionar que vamos
utilizar do Exchange default, e como o Exchange é default, a routingKey
tem que ser do mesmo nome da fila que vai receber essa mensagem e vai
rotear para o Consumer.

**- Dentro do package dtos, iniciamos uma nova classe chamada
EmailDto.**

Dentro dela, declaramos os atributos que vão ser recebidos lá no Dto do
Email Microservice e criamos os métodos Getters e Setters de cada um
deles.

Tendo o nosso Producer pronto, quem que vai acionar o método
publishMessageEmail?\
Será o método save do UserService, por isso que utilizamos a notation
**\@Transactional**.

Ou seja, logo após salvarmos o user no banco de dados, ele
automaticamente vai enviar o email.

Só que para isso nós vamos ter que fazer algumas mudanças. Além de
chamarmos o ponto de injeção do UserRepository, vamos chamar também o do
UserProducer, para que possamos utilizar os métodos dessa classe e
dispararmos esse email para o Broker logo quando ele for salvo na base
de dados.

~~~java
final UserRepository userRepository;
final UserProducer userProducer;

public UserService(UserRepository userRepository, UserProducer userProducer) {
    this.userRepository = userRepository;
    this.userProducer = userProducer;
}

@Transactional
public UserModel save(UserModel userModel) {
    userModel = userRepository.save(userModel);
    userProducer.publishMessageEmail(userModel);
    return userModel;
}
~~~

Agora, além de salvar o usuário, "ouvir as mensagens", nós já podemos
publicar essas mensagens no Broker.

## Executando o fluxo comunicação assíncrona entre microservices

No User Microservice:

Simulando o cadastro do usuário, é passado um JSON com o nome e o email.

Ele vai salvo no banco de dados, já com seu UUID, e logo depois vai
publicar essa mensagem no Broker, mas antes disso ele vai montar o Dto
do EmailDto no UserProducer no método publishMessageEmail e enviar essa
mensagem para o Broker passando o Exchange, a routingKey e o emailDto via 
~~~java
rabbitTemplate.convertAndSend("", routingKey, emailDto);
~~~

Agora na fila, lá no RabbitMQ, conseguimos ver que recebemos uma
mensagem e ver essa mensagem pelo **Get Message(s).** Logo, nosso
Producer conseguiu "produzir" e enviar a mensagem para a fila
(default.email).

No Email Microservice:

Por agora, verificamos se ela chegou no Email Microservice:

![](/images-readme/4.png)

Ou seja, o fluxo em resumo funciona da seguinte forma: Quando o User
Microservice ele cadastra um usuário, no final desse processo ele produz
uma mensagem EmailDto com o intuito de um comando, enviando esse comando
para um Broker, que redireciona para o Email Microservice, o
microserviço Consumer que está esperando esses comandos chegarem até ele
para que ele possa executar tal ação.

Esse tipo de comunicação é apenas um dos vários tipos de comunicações
entre microservices, como por exemplo: Mensageria com eventos,
implementação de padrões, modelos de comunicação síncrona e assíncrona,
modelos de comunicação assíncrona vias API's.

## Implementando envio de emails com SMTP Gmail.

**Agora focamos no resto da nossa implementação no Email Microservice,
pois o nosso User Microservice está pronto. Vamos enviar o email e
salvar o conteúdo desse email em nossa base de dados por microserviço.**

**- Criando o package Model e iniciando a classe EmailModel.**

Para que possamos salvar nosso email na base de dados, nós temos que
mapear uma entidade, por isso criamos o EmailModel.\
Sendo o nossos atributos: **emailId** (Identificador do email),
**userId** (Identificador de para quem foi enviado), **emailFrom** (quem
que enviou esse email), **emailTo** (para quem que foi enviado),
**text** (do tipo text para conseguir guardar mais caracteres, usando a
notation **\@Column** e especificando a sua definição),
**sendDateEmail** (data e horário de envio do email) e **statusEmail**
(vai ser um Enum que mostra o status do email).

Iniciamos os Getters e Setters.

**- Criando o package Enum e iniciando a classe Enum StatusEmail.**

Ele vai ter somente dois status: Enviado e Erro.

**- Criando o package Repositories iniciando a interface
EmailRepository.**

Precisamos dele para acessarmos todos os métodos JPA. Lembrando sempre
de passar nosso Model e o tipo do identificador da entidade.

Voltando ao EmailConsumer, que por enquanto estamos recebendo nosso
EmailRecordDto no corpo da mensagem e printando o email, nós vamos
receber esse Dto e transformar ele em um Model, ou seja, transformar o
emailRecordDto em um emailModel para ser salvo na base de dados.

E antes de fazer esse salvamento e prosseguir para o envio da mensagem
no email do usuário, nós precisamos criar a conexão com o Google SMTP.
Para fazer a conexão, assim como fizemos com o RabbitMQ, PostgreSQL, nós
vamos utilizar do application.properties, passando as seguintes
propriedades utilizando do Spring Mail:

~~~java
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=victormacedo1001@gmail.com
spring.mail.password=**** **** **** ****
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
~~~

\- Precisamos passar o host, que vai ser o smtp do Gmail.\
- Passamos a porta, sendo ela a 587, porta default.\
- O email de domínio que vai enviar os outros emails.\
- Precisamos passar uma senha especial chamada "senha para aplicativos",
para que ele tenha acesso a todas as configurações necessárias do email.

**OBS: A conta tem que ter a verificação de duas etapas ativa.**

Tendo feito a conexão, nós voltamos ao Consumer.

Para fazer todos esse envio de email e salvar esse email na base de
dados, nós precisamos utilizar de uma camada Service para colocar toda
essa lógica.

**- Criando o package Services e iniciando a classe EmailService.**

Ele vai ser um Bean do Spring do tipo **\@Service**.

Vamos precisar de pontos de injeções, sendo eles: emailRepository e
emailSender, responsável por fazer todo o envio dos email.\
Eles vão ser feitos via construtor.

~~~java
final EmailRepository emailRepository;
final JavaMailSender emailSender;

public EmailService(EmailRepository emailRepository, JavaMailSender mailSender) {
    this.emailRepository = emailRepository;
    this.emailSender = mailSender;
}
~~~

Vamos precisar também da propriedade que está o email que vamos utilizar
para enviar esses emails, ou seja, utilizamos o **\@Value** para
obtermos esse email e ser chamado via variável.

~~~java
@Value(value = "${spring.mail.username}")
private String emailFrom;
~~~

E finalmente vamos criar o método **sendEmail**, ele que vai ser
responsável por ocorrer toda a lógica do envio e salvamento do email na
base de dados, sendo literalmente consumido/chamado no emailConsumer.\
Ele vai retornar um EmailModel e receber um EmailModel lá do nosso
EmailConsumer.

~~~java
@Transactional
public EmailModel sendEmail(EmailModel emailModel) {
    try{
        emailModel.setSendDateEmail(LocalDateTime.now());
        emailModel.setEmailFrom(emailFrom);

        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(emailModel.getEmailTo());
        message.setSubject(emailModel.getSubject());
        message.setText(emailModel.getText());
        emailSender.send(message);

        emailModel.setStatusEmail(StatusEmail.SENT);
    } catch (MailException e){
        emailModel.setStatusEmail(StatusEmail.ERROR);
    } finally {
        return emailRepository.save(emailModel);
    }
}
~~~

Primeiro, vamos setar algumas informações do emailModel que vão faltar,
como:\
- Horário e Data que esse email foi enviado e quem que enviou esse
email, deixando tudo documentado no banco de dados.

Utilizando do **SimpleMailMessage**, onde vamos criar essa mensagem de
email e ao final vamos utilizar do emailSender para enviar, utilizando o
método send do JPA, e passar essa mensagem que vamos criar.\
Após isso, retornamos o statusEmail (Enum) como enviado.

Utilizamos o Error do statusEmail (Enum) no catch.

E independente, no finally, nós utilizamos o emailRepository com o
método save do JPA para enviar o emailModel para a base de dados do
Email Microservice.

**Voltando ao EmailConsumer.**

Nós criamos o ponto de injeção do EmailService via construtor para
podermos utilizarmos dos métodos e suas lógicas.

~~~java
emailService.sendEmail(emailModel);
~~~

E caso tudo estiver correto, nós finalizamos mais uma aplicação.
