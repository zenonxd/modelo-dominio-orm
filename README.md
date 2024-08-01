
<p align="center">
  <img src="https://img.shields.io/static/v1?label=SpringProfessional - Dev Superior&message=Modelo de Dominio e ORM&color=8257E5&labelColor=000000" alt="Testes automatizados na prática com Spring Boot" />
</p>

# Objetivo

Interpretar o diagrama UML abaixo e implementar no Spring + Java, fazendo seu seed da base dados. Ou seja, dar uma carga
inicial com alguns dados.

## Requisitos projeto

Todas as premissas e o sumário com o que deve ser feito está no "Documento de Requesitos DSCommerce.pdf". 
Como é algo específico do curso, não colocarei o link, mas você pode adquirir no site [devsuperior]().

## UML

![img.png](img.png)

## Criação de classes, qual criar primeiro vendo a UML?

A ideia é sempre começar por uma entidade independente. Para detectar isso, olhamos no diagrama as entidades que estão
nas pontas, como User, Payment. Mas no outro lado dela (no relacionamento), tem que estar o "muitos", não pode ter "um".

Por exemplo, no "User". Na sua linha de relacionamento, do outro lado é um "muitos" (no caso, o orders ali), ou seja, o 
User é independente, podendo ser iniciado sem colocar o pedido.

Outra entidade válida para ser iniciado primeiro, poderia também ser a "Category".

## Configuração banco H2, entidade User

Em application.properties:
```properties
spring.application.name=aula
spring.profiles.active=test
spring.jpa.open-in-view=false
```

Em application-test.properties, configurando banco H2:
```properties
spring.application.name=aula

# Dados de conexão com o banco H2
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.username=sa
spring.datasource.password=
# Configuração do cliente web do banco H2
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
# JPA, SQL
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.defer-datasource-initialization=true

# Configuração para mostrar o SQL no console
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

No nosso projeto, teremos vários perfis, seja: de teste, homologação, produção (nuvem). Mas enquanto estivermos
desenvolvendo e testando, o perfil será de teste, então, usaremos o banco H2.

## Mapeamento

Para que a nossa classe User apareça no nosso banco relacional, precisamos mapeá-la.

![img_1.png](img_1.png)

Assim, ao rodarmos a nossa Application, o Spring gerará a table e através da url "localhost:8080/h2-console", será
possivel checar nossa tabela tb_user.

Depois, só conectar no banco H2 com a URL do properties + user.

Tabela user criada:

![img_2.png](img_2.png)

## Order, Enum, relacionamento muitos-para-um

### Relacionamento muitos-para-um

Um pedido para um usuário. Um usuário, pode ter vários pedidos.

Classe Order:
```java
    @ManyToOne
    @JoinColumn(name = "client_id")
    private User client;
```

Como são várias Orders, será uma lista na classe User.

Classe User:
```java
    @OneToMany(mappedBy = "client")
    private List<Order> orders = new ArrayList<>();
```

Ao rodar o projeto com as anotações inseridas, dentro do H2, teremos uma nova coluna e tabela, veja:

![img_3.png](img_3.png)

Tabela Order criada, juntamente com a coluna "client_id", conforme passamos em sua classe.

❗Recomendação

No nosso atributo Instant, passar na coluna uma definição, para que ele seja salvo no banco de dados como um instante
padronizado em UTC (fuso horário de Londres, GMT), veja:
```java
    @Column(columnDefinition = "TIMESTAMP WITHOUT TIME ZONE")
    private Instant moment;
```

Assim, já saberemos que essa coluna é do tipo UTC. Será mais fácil depois converter para o horario local do usuário.

## Payment, relacionamento um-para-um

O pedido tem um pagamento. O pagamento está associado com um pedido. **(Payment depende do Order)**, pois para ele 
existir, precisa ter **AO MENOS, 1 PEDIDO**.

Além disso, o pedido pode existir sem pagamento (o minimo é zero pedidos).

Na entidade pagamento:
```java
    @OneToOne
    @MapsId
    private Order order;
```

Na entidade Order:
```java
    @OneToOne(mappedBy = "order", cascade = CascadeType.ALL)
    private Payment payment;
```

Ao rodarmos a nossa aplicação, no banco H2 veremos:

A table payment criada:

![img_4.png](img_4.png)

❗A coluna order_id existe, pois quando fizemos o mapeamento de um para um, fizemos a anotação @MapsId. Isso significa
que, a chave primária do payment, também será uma chave estrangeira com o MESMO número do pedido correspondente.

**Exemplo: Se tiver um pedido de número 5, o payment do pedido número 5, terá como id: 5!**

## Muitos-para-muitos, column unique e text

Aqui, relacionaremos Product e Category. 

Um produto pode ter muitas categorias. Uma categoria pode ter vários produtos.

Como a Category está na extremidade, ela não depende de Product.

Para indicarmos pro JPA que não terá qualquer tipo de repetição de category_id e product_id, utilizamos Set ao invés de
List.

Na entidade Product:
```java
    @ManyToMany
    @JoinTable(name = "tb_product_category",
            joinColumns = @JoinColumn(name = "product_id"),
            inverseJoinColumns = @JoinColumn(name = "category_id")
    )
    private Set<Category> categories = new HashSet<>();
```

O JoinTable criará uma tabela do meio (auxiliar/associação). Nela, passamos a referência para as duas chaves 
estrangeiras. 

Passamos o JoinClumns para o product_id e o InverseJoinColumns (do outro lado), a category_id.

❗O JoinColumn leva o nome da classe que estamos utilizando. Como é a Classe Product, é product_id.
<hr>

Na entidade Category:
```java
    @ManyToMany(mappedBy = "categories")
    private Set<Product> products = new HashSet<>();
```

Aqui é somente o mapeamento de como essa entidade (Category), foi mapeada na outra classe (Product).

Além disso, iremos ao JPA que a coluna "description", será uma coluna com vários caracteres. Usaremos o @Column na
entidade User, passando:
```java
    @Column(columnDefinition = "TEXT")
    private String description;
```

Assim, o JPA entende que ali será um texto longo, e não um varchar.

Por fim, configuraremos outros campos unicos, como email, por exemplo.
```java
    @Column(unique = true)
    private String email;
```

**Ao rodar o código:**

Tabela auxiliar criada:

![img_5.png](img_5.png)

## Mapeamento Muitos-para-muitos com classe de associação, parte 1

Ao analisar o diagrama, notamos que o relacionamento entre product e order é muitos para muitos e, além disso, possui
uma classe de associação (OrderItem).

**Isso acontece numa situação parecida com a de cima. A diferença é que a tabela auxiliar terá mais dados do que somente
as da chave estrangeira.**

### Como mapear uma classe de associação?

Bom, ela terá uma chave múltipla primária. Ou seja, dentro da nossa tabela de OrderItem no banco de dados, vai ter uma
chave estrangeira para o **ID do produto** e para o **ID da order**. Como as duas chaves estrangeiras são primárias,
temos um caso de **CHAVE COMPOSTA (EMBEDDED KEY)**.

### Chave Composta (Embedded Key)

Quando temos uma chave primária composta no banco relacional para mapearmos ela para o java (JPA), teremos que criar
uma classe a parte para representar essa chave primária, veja:

```java
//anotação para dizer que essa classe será incorporada por outras
@Embeddable
public class OrderItemPK {

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;

    @ManyToOne
    @JoinColumn(name = "product_id")
    private Product product;
}
```

Agora, na nossa classe OrderItem (que terá preço e quantidade), instanciaremos a classe auxiliar criada aima uma ID, 
veja:

```java
public class OrderItem {
    
    //usando a classe como referência, instanciando-a
    @EmbeddedId
    private OrderItemPK id = new OrderItemPK();
}
```

❗Importante. Não esquecer de ao instanciar o construtor na classe OrderItem, trocar os parâmetros. Não faz sentido
expormos no construtor a OrderItemPK. 

Se outra classe precisa instanciar a OrderItem, ela não precisa (e nem deve), conhecer artícios internos da JPA.
Portanto, o construtor terá: order e product, veja:

```java
    public OrderItem(Order order, Product product, Integer quantity, Double price) {
        id.setOrder(order);
        id.setProduct(product);
        this.quantity = quantity;
        this.price = price;
    }
```

Dentro do construtor, nós chamados o id (que está na classe OrderItem, mas inicialmente é da OrderItemPK), settando
seu Order e Product.

❗Importante. Dentro da classe **Order** e da classe **Product** podemos ter referência para os itens. Por exemplo:
Dentro do Product, usar um "getItems" e acessar os OrderItems associados a ele? Sim! Veja:

Classe Product: Importamos uma coleção de OrdemItem. E devemos lembrar o seguinte: Na classe do OrdemItem, temos o
id que é uma OrderItemPK. Dentro da OrderItemPK, temos a anotação ManyToOne. Dito isso, passaremos a anotação OneToMany
na coleção de OrderItem dentro de Product.
```java
    //O mapeamento é feito diferente. O Product que estamos nos referindo,
    // está dentro do OrderItemPK, que este, por sua vez, está dentro de OrderItem
    // como "id". Usamod o id e o "." para entrar na classe, selecionando o product
    @OneToMany(mappedBy = "id.product")
    private Set<OrderItem> items = new HashSet<>();
```

Classe Order: Faremos a mesma coisa acima.

```java
    @OneToMany(mappedBy = "id.order")
    private Set<OrderItem> items = new HashSet<>();
```

**Ao rodar o código:**

Tabela auxiliar OrderItem agora possui referência para o category_id e product_id.

![img_6.png](img_6.png)

## Mapeamento Muitos-para-muitos com classe de associação, parte 2

Na classe Order nós podemos ter um getProducts e na classe Product um getOrders. Isso é util em um cenário em que
gostariamos de ver, quais produtos estão em um pedido, por exemplo, ou vice-versa. Veja:

Na classe Order:
```java
    public Set<OrderItem> getItems() {
        return items;
    }
    public List<Product> getProducts() {
        return items.stream().map(OrderItem::getProduct).toList();
    }   
```

Aqui, nós pegamos o Set de Items do tipo OrderItem, e através do Map, transformamos eles em um Product.

Na classe Product:
```java
    public Set<OrderItem> getItems() {
        return items;
    }

    public List<Order> orders() {
        return items.stream().map(OrderItem::getOrder).toList();
    }
```

## Seeding da base de dados

Daremos uma carga inicial de dados quando a aplicação rodar.

Na pasta resources: criar um arquivo import.sql, contendo [isso.](https://gist.github.com/acenelio/664c3508edd4d418d566ed86179fdf8b)

## Gerando equals e hashCode das entidades

Colocaremos equals e hashCode em cada entidade. Caso a gente precise comparar uma com a outra, será possível.

Compararemos tudo por ID.

<hr>
Spring JPA - Subframework que auxilia a implementar acesso a dado a banco de dado relacional.