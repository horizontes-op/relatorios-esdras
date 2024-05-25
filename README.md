# Relatórios individuais da Disciplina de Plataformas, Apis e Microsserviços

## Roteiro individual sobre bottleneck:

### Redis

Nesse roteiro serão apresentados os passos para conectar a solução de caching do Redis com uma aplicação Spring Boot. 
A aplicação em questão é um CRUD básico de instituições de ensino e será aplicado cache nas rotas GET. Como as informações sobre as instituições variam pouco, o cache não deve trazer problemas de informações desatualizadas. A seguir os passos utilizados para implementar na aplicação:

#### 1. Adicionar Dependências ao pom.xml

No arquivo pom.xml do seu projeto Spring Boot, adicione as dependências necessárias para o Redis e para o cache Spring:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

#### 2. Configurar as Propriedades do Redis

No arquivo application.properties ou application.yml, configure as propriedades de conexão com o Redis. Aqui serão inseridos os dados para conexão com um redis local (rodando na mesma máquina), quando inserirmos o script para adicionar no docker compose devemos sobrescrever essas configurações.

```yaml
spring:
  cache: 
    type: redis
  redis:
    host: 127.0.0.1
    port: 6379

```


#### 3. Habilitar a Anotação de Caching

Na classe principal da sua aplicação, adicione a anotação @EnableCaching para habilitar o suporte a caching:

```java

package insper.store.instituicao;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
@EnableCaching // Adicione essa anotação
public class InstituicaoApplication {

    public static void main(String[] args) {
        SpringApplication.run(InstituicaoApplication.class, args);
    }   

}

```


#### 4. Configurar o Redis CacheManager

Crie uma classe de configuração para configurar o CacheManager usando Redis:
```java

package insper.store.instituicao.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    JedisConnectionFactory jedisConnectionFactory() {
        JedisConnectionFactory jedisConFactory = new JedisConnectionFactory();
        jedisConFactory.setHostName("redis"); // aqui o hostname está configurado para a conexão usando uma docker network, para testes locais mude para 127.0.0.1 (localhost)
        jedisConFactory.setPort(6379);
        return jedisConFactory;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(jedisConnectionFactory());
        
        // Definindo os serializadores
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        
        return template;
    }
}


```


#### 5. Adicionar Anotações de Cache nos Métodos de Serviço

No seu serviço, adicione a anotação @Cacheable para as rotas GET. É possível adicionar os paramêtros "value" que define qual a "tabela" que esses dados serão salvos. Em cada uma dessas, cada informação deve ter um identificador único utilizado para buscar pelo dado, que corresponde ao paramêtro "key". A seguir o trecho de código em que foram inseridas as anotações:

```java

package insper.store.instituicao;

import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import lombok.NonNull;

@Service
public class InstituicaoService {

    @Autowired
    private InstituicaoRepository instituicaoRepository;

    public Instituicao create(Instituicao in) {
        return instituicaoRepository.save(new InstituicaoModel(in)).to();
    }

    @Cacheable(value = "instituicao", key = "#id") // AQUI
    public Instituicao read(@NonNull String id) {
        return instituicaoRepository.findById(id).map(InstituicaoModel::to).orElse(null);
    }

    @Cacheable(value = "instituicao", key = "'ALLinstituicoes'") // AQUI
    public List<Instituicao> readAll() {
        Iterable<InstituicaoModel> allInstituicoes = instituicaoRepository.findAll();
        return StreamSupport.stream(allInstituicoes.spliterator(), false)
                            .map(InstituicaoModel::to)
                            .collect(Collectors.toList());
    }
    
    @Cacheable(value = "instituicao", key = "#nome") // AQUI
    public Instituicao getByNome(@NonNull String nome) {
        Optional<InstituicaoModel> instituicaoModel = instituicaoRepository.findByNome(nome);

        if (instituicaoModel.isEmpty()) {
            // Handle the case where no institution is found, e.g., throw a custom exception or return null
            return null;
        }

        return instituicaoModel.get().to();
    }
    
}


```
Com isso, finalizamos a configuração do serviço e podemos compilar usando o:

```bash
mvn clean install
```
ou

```bash
mvn clean package
```
#### 6. Adicionar no docker compose

Por fim, precisamos adicionar um container com o redis e alterar as environment variables do serviço que configuramos para usar o redis. Abaixo está os trechos do docker compose utilizado:
```yaml
version: '3.8'
name: store
services:
  # configuração básica do redis
  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - $VOLUME/redis/store/data:/var/lib/redis/data
    networks:
      - private-network

# restante dos microsserviços 

# ...

  instituicao:
    build:
      context: ../instituicao-resource/
      dockerfile: Dockerfile
    image: store-instituicao:latest
    environment:
      - spring.datasource.url=jdbc:postgresql://store-db-store:5432/store
      - spring.datasource.username=store
      - spring.datasource.password=store
      - eureka.client.service-url.defaultZone=http://store-discovery:8761/eureka/
      # variaveis de ambiente do redis
      - spring.redis.host=redis
      - spring.redis.port=6379
    deploy:
      mode: replicated
      replicas: 1
    networks:
      - private-network
    depends_on:
      - db-store
      - discovery
      - account

# criar a rede para que os serviços se conectem
networks:
  private-network:
    driver: bridge

```
Com isso, podemos seguir com o comando:

```bash
docker compose up -d --build
```
E subir a arquitetura

Author: Esdras Gomes Carvalho