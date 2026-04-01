# 📬 Sistema de Mensageria Completo com JMS e ActiveMQ

## 📋 Sobre o Projeto

Este projeto é uma aplicação Java completa que demonstra o uso avançado de **JMS (Java Message Service)** com **Apache ActiveMQ**, implementando um sistema de mensageria empresarial para gerenciamento de pedidos. O projeto foi desenvolvido com base no curso **"JMS e ActiveMQ: mensageria com Java"** da **Alura**, explorando conceitos fundamentais e avançados de mensageria, incluindo filas, tópicos, assinaturas duráveis, tratamento de erros e padrões de integração.

## 🎯 Objetivo

Implementar um sistema de mensageria empresarial robusto para processamento de pedidos, demonstrando:

- **Filas (Queues)** - Comunicação ponto-a-ponto
- **Tópicos (Topics)** - Padrão Publish/Subscribe
- **Assinaturas Duráveis** - Garantia de entrega mesmo com consumidor offline
- **Seletores de Mensagens** - Filtragem de mensagens
- **Transações JMS** - Controle transacional
- **Dead Letter Queue (DLQ)** - Tratamento de mensagens com erro
- **Serialização de Objetos** - ObjectMessage e JAXB
- **Configurações Avançadas** - Prioridade, TTL, persistência

## 🛠️ Tecnologias Utilizadas

### Core
- **Java** - Linguagem de programação principal
- **JMS 1.1** - Java Message Service API
- **JNDI** - Java Naming and Directory Interface
- **JAXB** - Java Architecture for XML Binding

### Message Broker
- **Apache ActiveMQ** - Broker de mensagens open-source
  - Console de administração web (porta 8161)
  - Protocolo TCP (porta 61616)
  - Suporte a filas e tópicos
  - DLQ integrada

### Dependências
- **activemq-all-5.12.0.jar** - Cliente ActiveMQ completo

## 🏗️ Arquitetura do Projeto

### Estrutura de Diretórios

```
java-jms-activemq-main/
│
├── src/                                              # Código-fonte
│   ├── br/com/caelum/jms/                           # Produtores e consumidores
│   │   ├── TesteProdutorFila.java                   # Produtor para fila
│   │   ├── TesteConsumidorFila.java                 # Consumidor de fila (transacional)
│   │   ├── TesteProdutorTopico.java                 # Produtor para tópico
│   │   ├── TesteConsumidorTopicoEstoque.java        # Consumidor tópico estoque
│   │   ├── TesteConsumidorTopicoComercial.java      # Consumidor tópico comercial
│   │   ├── TesteConsumidorTopicoEstoqueSelector.java # Consumidor com seletor
│   │   └── TesteConsumidorDLQ.java                  # Consumidor DLQ
│   │
│   ├── br/com/caelum/modelo/                        # Modelo de domínio
│   │   ├── Pedido.java                              # Entidade Pedido (Serializable + JAXB)
│   │   ├── Item.java                                # Entidade Item
│   │   └── PedidoFactory.java                       # Factory para geração de pedidos
│   │
│   └── jndi.properties                               # Configuração JNDI
│
├── jms1.1/                                           # Especificação JMS
│   ├── doc/api/                                      # JavaDoc JMS
│   └── src/share/javax/jms/                          # API JMS
│
├── activemq-data/                                    # Dados persistentes ActiveMQ
├── bin/                                              # Classes compiladas
└── README.md                                         # Documentação
```

## 🔑 Recursos e Funcionalidades Implementadas

### 1. **Comunicação via Filas (Point-to-Point)**

#### Produtor de Fila
**Classe:** `TesteProdutorFila.java`

```java
Destination fila = (Destination) context.lookup("financeiro");
MessageProducer producer = session.createProducer(fila);
Message message = session.createTextMessage("<pedido><id>13</id></pedido>");
producer.send(message);
```

**Características:**
- ✅ Envio de mensagens XML para fila financeiro
- ✅ Comunicação assíncrona
- ✅ Garantia de entrega para um único consumidor

#### Consumidor de Fila (Transacional)
**Classe:** `TesteConsumidorFila.java`

```java
Session session = connection.createSession(true, Session.SESSION_TRANSACTED);
MessageConsumer consumer = session.createConsumer(fila);
consumer.setMessageListener(new MessageListener() {
    public void onMessage(Message message) {
        // Processa mensagem
        session.rollback(); // Força reprocessamento
    }
});
```

**Características:**
- ✅ Sessão transacional (`SESSION_TRANSACTED`)
- ✅ Rollback intencional para demonstração
- ✅ MessageListener assíncrono
- ⚠️ Mensagens não confirmadas vão para DLQ após redelivery

### 2. **Comunicação via Tópicos (Publish/Subscribe)**

#### Produtor de Tópico
**Classe:** `TesteProdutorTopico.java`

```java
Destination topico = (Destination) context.lookup("loja");
MessageProducer producer = session.createProducer(topico);

Pedido pedido = new PedidoFactory().geraPedidoComValores();
Message message = session.createObjectMessage(pedido);
producer.send(message);
```

**Características:**
- ✅ Envio de objetos serializados (`ObjectMessage`)
- ✅ Um produtor → Múltiplos consumidores
- ✅ Broadcast de mensagens
- ✅ Suporte a propriedades customizadas

#### Consumidores de Tópico

##### A. Consumidor Estoque (Durable Subscriber)
**Classe:** `TesteConsumidorTopicoEstoque.java`

```java
connection.setClientID("estoque");
MessageConsumer consumer = session.createDurableSubscriber(topico, "assinatura");
```

**Características:**
- ✅ Assinatura durável
- ✅ Recebe mensagens mesmo quando offline
- ✅ ClientID único: "estoque"

##### B. Consumidor Comercial (Transacional)
**Classe:** `TesteConsumidorTopicoComercial.java`

```java
connection.setClientID("comercial");
Session session = connection.createSession(true, Session.SESSION_TRANSACTED);
MessageConsumer consumer = session.createDurableSubscriber(topico, "assinatura");

public void onMessage(Message message) {
    ObjectMessage objectMessage = (ObjectMessage) message;
    Pedido pedido = (Pedido) objectMessage.getObject();
    System.out.println(pedido.getCodigo());
    session.commit();
}
```

**Características:**
- ✅ Processamento transacional
- ✅ Desserialização de objetos Java
- ✅ Commit explícito
- ✅ ClientID único: "comercial"

### 3. **Seletores de Mensagens (Message Selectors)**

**Classe:** `TesteConsumidorTopicoEstoqueSelector.java`

```java
MessageConsumer consumer = session.createDurableSubscriber(
    topico, 
    "assinatura-selector", 
    "ebook is null OR ebook=false", 
    false
);
```

**Funcionalidade:**
- ✅ Filtra mensagens baseado em propriedades
- ✅ Recebe apenas produtos físicos (não ebooks)
- ✅ Sintaxe SQL-like para filtros
- ✅ Reduz processamento desnecessário

**Exemplo de uso no produtor:**
```java
message.setBooleanProperty("ebook", true); // Não será consumido pelo seletor
```

### 4. **Dead Letter Queue (DLQ)**

**Classe:** `TesteConsumidorDLQ.java`

```java
Destination fila = (Destination) context.lookup("DLQ");
MessageConsumer consumer = session.createConsumer(fila);
consumer.setMessageListener(new MessageListener() {
    public void onMessage(Message message) {
        System.out.println(message); // Mensagens problemáticas
    }
});
```

**Características:**
- ✅ Captura mensagens com erro
- ✅ Mensagens após máximo de redelivery
- ✅ Poison messages isoladas
- ✅ Permite análise e correção de erros

### 5. **Modelo de Domínio com Serialização**

#### Classe Pedido
**Características:**
- ✅ `Serializable` - Para ObjectMessage
- ✅ Anotações JAXB - Para XML
- ✅ `@XmlRootElement`, `@XmlAccessorType`
- ✅ Relacionamento com Item (Set)

```java
@XmlRootElement
@XmlAccessorType(XmlAccessType.FIELD)
public class Pedido implements Serializable {
    private Integer codigo;
    private Calendar dataPedido;
    private Calendar dataPagamento;
    private BigDecimal valorTotal;
    
    @XmlElementWrapper(name="itens")
    @XmlElement(name="item")
    private Set<Item> itens = new LinkedHashSet<>();
}
```

#### Classe Item
```java
@XmlAccessorType(XmlAccessType.FIELD)
public class Item implements Serializable {
    private Integer id;
    private String nome;
}
```

#### Factory Pattern
**Classe:** `PedidoFactory.java`
- ✅ Geração de pedidos com valores de teste
- ✅ Criação de itens associados
- ✅ Parsing de datas

### 6. **Configuração JNDI**

**Arquivo:** `jndi.properties`

```properties
# Factory do ActiveMQ
java.naming.factory.initial = org.apache.activemq.jndi.ActiveMQInitialContextFactory

# URL do broker
java.naming.provider.url = tcp://192.168.0.208:61616

# Filas
queue.financeiro = fila.financeiro
queue.DLQ = ActiveMQ.DLQ

# Tópicos
topic.loja = topico.loja
```

**Recursos:**
- ✅ Configuração centralizada
- ✅ Mapeamento de filas e tópicos
- ✅ DLQ configurada
- ✅ Fácil mudança de ambiente

## 💡 Conceitos JMS Implementados

### ✅ Componentes Básicos
- [x] **ConnectionFactory** - Factory de conexões
- [x] **Connection** - Conexão com broker
- [x] **Session** - Contexto de trabalho
- [x] **Destination** - Fila ou Tópico
- [x] **MessageProducer** - Produtor de mensagens
- [x] **MessageConsumer** - Consumidor de mensagens
- [x] **MessageListener** - Callback assíncrono

### ✅ Tipos de Mensagens
- [x] **TextMessage** - Mensagens de texto (XML)
- [x] **ObjectMessage** - Objetos Java serializados

### ✅ Padrões de Mensageria
- [x] **Point-to-Point (Queue)** - Fila financeiro
- [x] **Publish/Subscribe (Topic)** - Tópico loja
- [x] **Durable Subscription** - Assinaturas persistentes

### ✅ Recursos Avançados
- [x] **Message Selectors** - Filtros SQL-like
- [x] **Transações JMS** - SESSION_TRANSACTED
- [x] **Rollback/Commit** - Controle transacional
- [x] **Dead Letter Queue** - Tratamento de erros
- [x] **ClientID** - Identificação única de clientes

### ✅ Modos de Sessão
- [x] **AUTO_ACKNOWLEDGE** - Reconhecimento automático
- [x] **SESSION_TRANSACTED** - Transacional

## 🚀 Como Executar o Projeto

### Pré-requisitos

1. **JDK 8 ou superior**
2. **Apache ActiveMQ** instalado e rodando
3. **activemq-all-5.12.0.jar** no classpath

### Configuração do ActiveMQ

#### 1. Instalar ActiveMQ
```bash
# Download
wget https://archive.apache.org/dist/activemq/5.12.0/apache-activemq-5.12.0-bin.tar.gz

# Extrair
tar -xzf apache-activemq-5.12.0-bin.tar.gz
cd apache-activemq-5.12.0
```

#### 2. Iniciar ActiveMQ
```bash
# Linux/Mac
./bin/activemq start

# Windows
bin\activemq.bat start
```

#### 3. Acessar Console
- **URL:** http://localhost:8161/admin
- **Usuário:** admin
- **Senha:** admin

#### 4. Criar Destinos
No console, criar:
- **Fila:** `fila.financeiro`
- **Tópico:** `topico.loja`

### Configuração do Projeto

#### 1. Ajustar jndi.properties
```properties
java.naming.provider.url = tcp://localhost:61616
```

#### 2. Compilar
```bash
javac -cp ".:activemq-all-5.12.0.jar" src/br/com/caelum/**/*.java
```

### Cenários de Execução

#### Cenário 1: Fila com Transação e DLQ

**Terminal 1 - Consumidor Fila (com rollback):**
```bash
java -cp ".:src:activemq-all-5.12.0.jar" br.com.caelum.jms.TesteConsumidorFila
```

**Terminal 2 - Consumidor DLQ:**
```bash
java -cp ".:src:activemq-all-5.12.0.jar" br.com.caelum.jms.TesteConsumidorDLQ
```

**Terminal 3 - Produtor:**
```bash
java -cp ".:src:activemq-all-5.12.0.jar" br.com.caelum.jms.TesteProdutorFila
```

**Comportamento:**
1. Mensagem enviada para fila.financeiro
2. Consumidor processa e faz rollback
3. Após 6 tentativas, mensagem vai para DLQ
4. Consumidor DLQ captura mensagem problemática

#### Cenário 2: Tópico com Múltiplos Consumidores

**Terminal 1 - Consumidor Estoque:**
```bash
java -cp ".:src:activemq-all-5.12.0.jar" br.com.caelum.jms.TesteConsumidorTopicoEstoque
```

**Terminal 2 - Consumidor Comercial:**
```bash
java -cp ".:src:activemq-all-5.12.0.jar" br.com.caelum.jms.TesteConsumidorTopicoComercial
```

**Terminal 3 - Produtor:**
```bash
java -cp ".:src:activemq-all-5.12.0.jar" br.com.caelum.jms.TesteProdutorTopico
```

**Comportamento:**
1. Mensagem enviada para topico.loja
2. **Ambos** consumidores recebem a mesma mensagem
3. Estoque: recebe TextMessage
4. Comercial: recebe ObjectMessage e faz commit

#### Cenário 3: Seletor de Mensagens

**Terminal 1 - Consumidor com Seletor:**
```bash
java -cp ".:src:activemq-all-5.12.0.jar" br.com.caelum.jms.TesteConsumidorTopicoEstoqueSelector
```

**Terminal 2 - Produtor:**
```bash
java -cp ".:src:activemq-all-5.12.0.jar" br.com.caelum.jms.TesteProdutorTopico
```

**Modificar produtor para testar:**
```java
message.setBooleanProperty("ebook", true);  // Não receberá
message.setBooleanProperty("ebook", false); // Receberá
```

## 📊 Arquitetura de Mensageria

### Fluxo de Fila (Point-to-Point)

```
┌─────────────────┐         ┌──────────────────┐        ┌─────────────────┐
│TesteProdutorFila│  send   │ Apache ActiveMQ  │receive │TesteConsumidor  │
│                 │ ──────> │ fila.financeiro  │ ──────>│    Fila         │
│  <pedido>       │         │                  │        │  (rollback)     │
└─────────────────┘         └──────────────────┘        └─────────────────┘
                                     │
                                     │ após 6 tentativas
                                     ↓
                            ┌──────────────────┐
                            │   ActiveMQ.DLQ   │
                            │ (Dead Letter Q)  │
                            └──────────────────┘
                                     ↓
                            ┌─────────────────┐
                            │ TesteConsumidor │
                            │      DLQ        │
                            └─────────────────┘
```

### Fluxo de Tópico (Pub/Sub)

```
                              ┌────────────────────────────┐
                              │  Apache ActiveMQ           │
                              │  topico.loja               │
                              └────────────────────────────┘
                                     ⬆         ⬇  ⬇  ⬇
                                     │         │  │  │
┌────────────────────┐               │         │  │  │
│TesteProdutorTopico │    publish    │         │  │  │
│                    │ ──────────────┘         │  │  │
│ Pedido (Object)    │                         │  │  │
└────────────────────┘                         │  │  │
                                               │  │  │
                          ┌────────────────────┘  │  │
                          │                       │  │
              ┌───────────▼────────┐   ┌──────────▼─────────┐
              │ Consumidor Estoque │   │Consumidor Comercial│
              │   (ClientID)       │   │    (ClientID)      │
              │ Durable Subscriber │   │ Durable + Transact │
              └────────────────────┘   └────────────────────┘
                                               │
                          ┌────────────────────┘
                          │
              ┌───────────▼────────────┐
              │ Consumidor Estoque     │
              │   com Seletor          │
              │ (ebook=false)          │
              └────────────────────────┘
```

## 🔧 Detalhes Técnicos

### Assinaturas Duráveis

**Características:**
- Requer `setClientID()` único
- Requer nome de assinatura
- Mensagens persistidas mesmo com consumidor offline
- Broker armazena mensagens até consumo

**Código:**
```java
connection.setClientID("estoque"); // Único!
MessageConsumer consumer = session.createDurableSubscriber(
    topico, 
    "assinatura"  // Nome da assinatura
);
```

### Transações JMS

**Sessão Transacional:**
```java
Session session = connection.createSession(true, Session.SESSION_TRANSACTED);
```

**Commit:**
```java
session.commit(); // Confirma recebimento
```

**Rollback:**
```java
session.rollback(); // Mensagem retorna à fila
```

### Dead Letter Queue (DLQ)

**Configuração Padrão ActiveMQ:**
- Máximo de 6 tentativas de redelivery
- Após 6 falhas → DLQ
- Fila especial: `ActiveMQ.DLQ`

**Quando ocorre:**
- Rollback repetido
- Exceções não tratadas
- Timeout de processamento

### Message Selectors

**Sintaxe SQL-like:**
```java
"ebook is null OR ebook=false"
"price > 100 AND category='electronics'"
"JMSPriority > 5"
```

**Operadores:**
- `AND`, `OR`, `NOT`
- `=`, `<>`, `>`, `<`, `>=`, `<=`
- `IS NULL`, `IS NOT NULL`
- `IN`, `LIKE`

### Serialização de Objetos

**ObjectMessage:**
```java
// Produtor
Pedido pedido = new PedidoFactory().geraPedidoComValores();
Message message = session.createObjectMessage(pedido);

// Consumidor
ObjectMessage objectMessage = (ObjectMessage) message;
Pedido pedido = (Pedido) objectMessage.getObject();
```

**Requisitos:**
- Classe implementa `Serializable`
- mesmas classes no classpath (produtor e consumidor)
- `serialVersionUID` compatível

### JAXB para XML

**Conversão Objeto → XML:**
```java
StringWriter writer = new StringWriter();
JAXB.marshal(pedido, writer);
String xml = writer.toString();
```

**Saída XML:**
```xml
<pedido>
    <codigo>2459</codigo>
    <dataPedido>2016-12-22T00:00:00</dataPedido>
    <valorTotal>145.98</valorTotal>
    <itens>
        <item>
            <id>23</id>
            <nome>Moto G</nome>
        </item>
    </itens>
</pedido>
```

## 📝 Comparação: Fila vs Tópico

| Aspecto | Fila (Queue) | Tópico (Topic) |
|---------|--------------|----------------|
| **Padrão** | Point-to-Point | Publish/Subscribe |
| **Consumidores** | Um por mensagem | Todos assinantes |
| **Uso** | Processamento de tarefas | Notificações/eventos |
| **Garantia** | Uma vez para um consumidor | Uma vez para cada assinante |
| **Exemplo** | Fila de pedidos financeiros | Atualização de estoque |

## 🎓 Casos de Uso do Projeto

### 1. Sistema de Pedidos Distribuído

```
┌──────────────┐
│   Website    │
└──────┬───────┘
       │ cria pedido
       ↓
┌──────────────────┐
│  fila.financeiro │
└──────────────────┘
       │
       ↓
┌──────────────┐
│  Financeiro  │ → Processa pagamento
└──────────────┘
```

### 2. Notificação de Múltiplos Departamentos

```
       ┌──────────────┐
       │ topico.loja  │
       └───────┬──────┘
               │
      ┌────────┼────────┐
      ↓        ↓        ↓
┌─────────┐ ┌────────┐ ┌─────────┐
│ Estoque │ │Comercial │Logística│
└─────────┘ └────────┘ └─────────┘
```

### 3. Filtro de Produtos (Seletor)

```
Mensagem: {produto: "livro", ebook: false}  → Consumidor Estoque ✅
Mensagem: {produto: "ebook", ebook: true}   → Consumidor Estoque ❌
```

### 4. Tratamento de Erros

```
Pedido inválido → Rollback → Retry → ... → DLQ → Análise manual
```

## 🔍 Vantagens da Arquitetura

### 1. **Desacoplamento**
- Produtores não conhecem consumidores
- Fácil adicionar novos consumidores
- Mudanças independentes

### 2. **Escalabilidade**
- Múltiplos consumidores em paralelo (fila)
- Load balancing automático
- Processamento distribuído

### 3. **Confiabilidade**
- Mensagens persistidas
- Redelivery automático
- DLQ para erros

### 4. **Flexibilidade**
- Seletores filtram mensagens
- Assinaturas duráveis
- Transações JMS

### 5. **Auditoria**
- Rastreamento de mensagens
- Monitoramento no console ActiveMQ
- Dead letter queue para análise

## 🐛 Troubleshooting

### Problema: Mensagens vão direto para DLQ
```
Causa: Rollback repetido no consumidor
```
**Solução:** Remover `session.rollback()` ou ajustar lógica de negócio

### Problema: Consumidor não recebe mensagens do tópico
```
Causa: Consumidor iniciado após envio (sem durable subscription)
```
**Solução:** Usar `createDurableSubscriber()` com ClientID

### Problema: Duplicate key ao usar ClientID
```
javax.jms.InvalidClientIDException: Broker already has subscriber
```
**Solução:** Usar ClientID único ou fechar conexão anterior

### Problema: ClassNotFoundException no ObjectMessage
```
Causa: Classe Pedido não está no classpath do consumidor
```
**Solução:** Garantir que modelo está em ambos os lados

### Problema: Mensagens não são filtradas pelo seletor
```
Causa: Propriedade não definida no produtor
```
**Solução:** Verificar `message.setBooleanProperty("ebook", false)`

## 📚 Referências

### Curso Base
**"JMS e ActiveMQ: mensageria com Java"** - Alura

### Tópicos do Curso:
- ✅ Introdução ao Mensageria com ActiveMQ
- ✅ Consumindo mensagens com JMS
- ✅ Recebendo mensagens com MessageListener
- ✅ Enviando e distribuindo mensagens
- ✅ Tópicos e assinaturas duráveis
- ✅ Seletores e propriedades da Mensagem
- ✅ Enviando mensagens específicas e tratamento de erro
- ✅ Prioridade e Tempo de vida da mensagem

### Padrões Implementados:
- ✅ Point-to-Point (Queue)
- ✅ Publish/Subscribe (Topic)
- ✅ Durable Subscriber
- ✅ Message Selector
- ✅ Dead Letter Channel
- ✅ Transactional Client

## 🌟 Possíveis Extensões

### Melhorias Sugeridas
- [ ] Implementar Request/Reply pattern
- [ ] Adicionar correlationID para rastreamento
- [ ] Implementar Message Expiration
- [ ] Adicionar prioridades de mensagens
- [ ] Criar virtual topics
- [ ] Implementar Message Groups
- [ ] Adicionar Spring JMS integration
- [ ] Implementar retry policy customizado
- [ ] Adicionar métricas e monitoramento
- [ ] Implementar circuit breaker

## 📄 Licença

Este é um projeto educacional desenvolvido para fins de aprendizado.

## 👨‍💻 Autor

Projeto desenvolvido durante o curso de JMS e ActiveMQ da Alura.

## 🔗 Links Úteis

- [Apache ActiveMQ Documentation](https://activemq.apache.org/documentation)
- [JMS 1.1 Specification](https://jcp.org/en/jsr/detail?id=914)
- [ActiveMQ Message Selectors](https://activemq.apache.org/selectors)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)

---

**Nota:** Este README técnico documenta um sistema completo de mensageria empresarial, demonstrando padrões avançados de integração e boas práticas com JMS e ActiveMQ.
