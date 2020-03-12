# Circuit Breaker

O Hystrix é mais uma solução do grupo Netflix OSS, ele implementa o padrão Circuit Breaker e visa lidar com as possíveis falhas nas chamadas entre microsserviços.


## **Preparação do ambiente**

executar a aplicação Eureka Server para que as outras aplicações possam registrar suas instâncias. A aplicação utiliza a porta **8761** por padrão, sendo acessível após a inicialização em [http://localhost:8761](http://localhost:8761/).

![circuit-breaker-1](http://www.matera.com/br/wp-content/uploads/2018/07/circuit-breaker-1.jpg)

Iniciaremos também a execução da aplicação “**greeting-service**”, que fornecerá o serviço a ser consumido, ela representará um serviço sujeito a falhas. Após sua inicialização, confira a página do Eureka para confirmar que ela se registrou com sucesso, como na imagem a seguir:

![circuit-breaker-2](http://www.matera.com/br/wp-content/uploads/2018/07/circuit-breaker-2.jpg)

Essa aplicação, possui um _endpoint_ mapeado em “**/greeting**” que ao receber uma requisição **Get** retorna aleatoriamente as saudações: “**Hi**”, “**Hey**” ou “**Hello**”.

## **Circuit Breaker**

Na aplicação “**user-greeting-service**” existe uma classe chamada **GreetingConsumer** que utiliza um _RestTemplate_ para fazer chamadas para a aplicação “**greeting-service**” e obter uma saudação aleatória. Este é um potencial ponto de falhas da aplicação, portanto, vamos implementar o Circuit Breaker para essa chamada.

Primeiramente, para utilizar o Hystrix será necessário adicionar a seguinte dependência ao projeto:

Maven:

```maven
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

Gradle:

```gradle
compile('org.springframework.cloud:spring-cloud-starter-netflix-hystrix')
```

Para que o Spring saiba que esta aplicação possui _circuit breakers_ e permitir o monitoramento das chamadas adicione a anotação **@EnableCircuitBreaker** na classe principal:

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public class UserGreetingServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserGreetingServiceApplication.class, args);
    }
}
```

No método **getRandomGreeting()** da classe **GreetingConsumer,** vamos adicionar a anotação **@HystrixCommand** e informar qual será o nome do método de _fallback_ através do atributo “fallbackMethod”.

Também criaremos o método de _fallback_ com os mesmos parâmetros e tipo de retorno do método original, assim, quando a chamada original falhar o Hystrix irá disparar o _fallback_ que retornará uma saudação padrão, desta forma o usuário não percebe que o comportamento da aplicação foi afetado.
```java
@HystrixCommand(fallbackMethod = "getDefaultGreeting")
public String getRandomGreeting() {
    String uri = "[http://greeting-service](http://greeting-service/)" + greetingEndpointUri;
    String greeting = restTemplate.getForObject(uri, String.class);
    return greeting;
}

public String getDefaultGreeting() {
    return "Good bye";
}
```

Neste ponto, a aplicação já está pronta para utilizar os serviços básicos de _Circuit Breaker_ oferecidos pelo _Hystrix_.

Para facilitar a visualização, criaremos um RestController com um _endpoint_ que recebe um nome como parâmetro de _url_ e exibe para o usuário uma saudação com o nome informado.

```java
@RestController
@RequestMapping("/hello")
public class UserGreetingController {

    @Autowired
    private GreetingConsumer consumer;

    @GetMapping("/{username}")
    public String sayHello(@PathVariable String username) {
        String greeting = consumer.getRandomGreeting();
        return greeting + " " + username + "!";
    }
}
```

Para verificar que a comunicação entre os serviços está funcionando adequadamente, é preciso executar a aplicação e acessar [http://localhost:9090/hello/world](http://localhost:9090/hello/world). O comportamento esperado é obter as respostas: “**Hi world!**”, “**Hello world!**” ou “**Hey world!**”.

Para finalmente vermos o padrão Circuit Breaker agindo será preciso simular uma falha na comunicação, para isso pare a execução do serviço “**greeting-service**”. Então tente acessar o endpoint [http://localhost:9090/hello/world](http://localhost:9090/hello/world) novamente, desta vez, a resposta será gerada pelo método de _fallback_, pois não foi possível acessar o serviço original, sendo assim a resposta obtida deve ser “**Good bye world!**”.
