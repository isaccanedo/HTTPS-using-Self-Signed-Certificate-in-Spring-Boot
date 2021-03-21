# HTTPS usando certificado autoassinado no Spring Boot - Spring Boot Security MVC

### HTTPS usando certificado autoassinado no Spring Boot

# 1. Visão Geral
Neste tutorial, vamos mostrar como habilitar HTTPS no Spring Boot. Para isso, também geraremos um certificado autoassinado e configuraremos um aplicativo simples.

# 2. Gerando um certificado autoassinado
Antes de começar, criaremos um certificado autoassinado. Usaremos um dos seguintes formatos de certificado:

- PKCS12: Padrões criptográficos de chave pública é um formato protegido por senha que pode conter vários certificados e chaves; é um formato usado em todo o setor;
- JKS: Java KeyStore é semelhante ao PKCS12; é um formato proprietário e é limitado ao ambiente Java.
Podemos usar as ferramentas keytool ou OpenSSL para gerar os certificados a partir da linha de comando. O Keytool é fornecido com o Java Runtime Environment e o OpenSSL pode ser baixado aqui.

Vamos usar o keytool para nossa demonstração.

2.1. Gerando um Keystore
Agora vamos criar um conjunto de chaves criptográficas e armazená-lo em um armazenamento de chaves.

Podemos usar o seguinte comando para gerar nosso formato de armazenamento de chaves PKCS12:

```
keytool -genkeypair -alias isac -keyalg RSA -keysize 2048 -storetype PKCS12 
-keystore isac.p12 -validity 3650
```

Podemos armazenar tantos números de par de chaves no mesmo armazenamento de chaves, cada um identificado por um alias exclusivo.

Para gerar nosso armazenamento de chaves em formato JKS, podemos usar o seguinte comando:

```
keytool -genkeypair -alias isac -keyalg RSA -keysize 2048 -keystore isac.jks -validity 3650
```

Recomenda-se usar o formato PKCS12, que é um formato padrão da indústria. Portanto, caso já tenhamos um keystore JKS, podemos convertê-lo para o formato PKCS12 usando o seguinte comando:

```
keytool -importkeystore -srckeystore isac.jks -destkeystore isac.p12 -deststoretype pkcs12
```

Teremos que fornecer a senha do keystore de origem e também definir uma nova senha do keystore. O alias e a senha do keystore serão necessários posteriormente.

# 3. Habilitando HTTPS no Spring Boot
Spring Boot fornece um conjunto de propriedades declarativas server.ssl. *. Usaremos essas propriedades em nosso aplicativo de amostra para configurar HTTPS.

Começaremos com um aplicativo Spring Boot simples com Spring Security que contém uma página de boas-vindas tratada pelo endpoint “/ welcome”.

Em seguida, copiaremos o arquivo denominado isac.p12 ″ gerado na etapa anterior para o diretório“ src/main/resources/keystore ”.

3.1. Configurando Propriedades SSL
Agora, vamos configurar as propriedades relacionadas ao SSL:

```
# O formato usado para o armazenamento de chaves. Pode ser definido como JKS no caso de ser um arquivo JKS
server.ssl.key-store-type=PKCS12
# O caminho para o keystore que contém o certificado
server.ssl.key-store=classpath:keystore/isac.p12
# A senha usada para gerar o certificado
server.ssl.key-store-password=password
# O alias mapeado para o certificado
server.ssl.key-alias=isac
```

Como estamos usando um aplicativo habilitado para Spring Security, vamos configurá-lo para aceitar apenas solicitações HTTPS:

```
server.ssl.enabled=true
```

# 4. Invocando um URL HTTPS
Agora que habilitamos o HTTPS em nosso aplicativo, vamos prosseguir para o cliente e explorar como invocar um endpoint HTTPS com o certificado autoassinado.

Primeiro, precisamos criar um armazenamento confiável. Como geramos um arquivo PKCS12, podemos usar o mesmo que o armazenamento confiável. Vamos definir novas propriedades para os detalhes do armazenamento confiável:

```
#localização do armazenamento confiável
trust.store=classpath:keystore/isac.p12
#senha de armazenamento confiável
trust.store.password=password
```

Agora precisamos preparar um SSLContext com o armazenamento confiável e criar um RestTemplate personalizado:

```
RestTemplate restTemplate() throws Exception {
    SSLContext sslContext = new SSLContextBuilder()
      .loadTrustMaterial(trustStore.getURL(), trustStorePassword.toCharArray())
      .build();
    SSLConnectionSocketFactory socketFactory = new SSLConnectionSocketFactory(sslContext);
    HttpClient httpClient = HttpClients.custom()
      .setSSLSocketFactory(socketFactory)
      .build();
    HttpComponentsClientHttpRequestFactory factory = 
      new HttpComponentsClientHttpRequestFactory(httpClient);
    return new RestTemplate(factory);
}
```

Por causa da demonstração, vamos nos certificar de que o Spring Security permite todas as solicitações de entrada:

```
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
      .antMatchers("/**")
      .permitAll();
}
```

Finalmente, podemos fazer uma chamada para o endpoint HTTPS:

```
@Test
public void whenGETanHTTPSResource_thenCorrectResponse() throws Exception {
    ResponseEntity<String> response = 
      restTemplate().getForEntity(WELCOME_URL, String.class, Collections.emptyMap());

    assertEquals("<h1>Welcome to Secured Site</h1>", response.getBody());
    assertEquals(HttpStatus.OK, response.getStatusCode());
}
```

# 5. Conclusão
Neste tutorial, aprendemos primeiro como gerar um certificado autoassinado para habilitar HTTPS em um aplicativo Spring Boot. Além disso, mostramos como invocar um endpoint habilitado para HTTPS.

Finalmente, para executar o exemplo de código, precisamos descomentar a seguinte propriedade start-class no pom.xml:

```
<start-class>com.isac.ssl.HttpsEnabledApplication</start-class>
```

