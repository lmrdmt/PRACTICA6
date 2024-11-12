

---

## Proxecto de Configuración DNS con Bind9 en Docker

Este proxecto utiliza Docker para configurar un servidor DNS con `bind9` nun contedor, e un cliente configurado para utilizar ese servidor DNS.

### Estrutura de `docker-compose.yml`

O ficheiro `docker-compose.yml` define dous servizos: un servidor DNS (`asir_bind9`) e un cliente (`cliente`). A continuación explícanse as opcións e parámetros clave para cada servizo e a rede configurada.

#### Servizos

1. **asir_bind9**: Servidor DNS baseado en `bind9`
   - **container_name**: Define o nome do contedor como `asir_bind9`.
   - **image**: Usa a imaxe `ubuntu/bind9`, que inclúe a versión de Bind9 para Ubuntu.
   - **platform**: Especifica `linux/amd64` como a plataforma para garantir compatibilidade en sistemas que usen esta arquitectura.
   - **ports**: Expon o porto 54 tanto en UDP como en TCP. Isto permite conexións DNS dende outros contedores e dende o host.
     - `54:54/udp`: Mapea o porto 54 UDP do contedor ao host.
     - `54:54/tcp`: Mapea o porto 54 TCP do contedor ao host.
   - **networks**: Conecta o contedor á rede `bind9_subnet`, asignándolle a dirección IP `192.168.1.254`.
   - **volumes**: Monta dous volumes locais para configuración e ficheiros de zona:
     - `./conf:/etc/bind`: Directorio local `conf` para ficheiros de configuración de Bind9.
     - `./zonas:/var/lib/bind`: Directorio local `zonas` para ficheiros de zona do DNS.

2. **cliente**: Contedor cliente configurado para usar o servidor DNS `asir_bind9`.
   - **container_name**: Nome do contedor `cliente`.
   - **image**: Imaxe de `ubuntu`, que permite unha configuración lixeira baseada en Ubuntu.
   - **platform**: Configurado para `linux/amd64`.
   - **tty** e **stdin_open**: Activa un terminal interactivo no contedor para facilitar a interacción directa.
   - **dns**: Configura o DNS do contedor apuntando a `192.168.1.254`, que é a IP do contedor `asir_bind9`.
   - **networks**: Conecta o contedor á rede `bind9_subnet` coa dirección IP `192.168.1.2`.

#### Rede

**bind9_subnet**: Rede `bridge` configurada para ambos contedores.
   - **driver**: Usa `bridge`, un controlador estándar de Docker para redes locais.
   - **ipam** (IP Address Management): Define o rango de direccións IP dispoñible:
     - `subnet: 192.168.1.0/24`: Rango de direccións IP da subrede, da que se asignan IPs estáticas a cada contedor (`192.168.1.254` para o servidor e `192.168.1.2` para o cliente).

### Comandos para Comprobar a Configuración de DNS dende o Cliente

Para verificar a configuración, utilizaremos o comando `docker exec` para acceder ao contedor cliente e probar a resolución DNS con `dig`. 

1. **Iniciar os Contedores**
   Primeiro, inicia os contedores definidos no ficheiro `docker-compose.yml`:
   ```bash
   docker-compose up -d
   ```

2. **Acceder ao Contedor Cliente**
   Utiliza `docker exec` para abrir un terminal no contedor `cliente`:
   ```bash
   docker exec -it cliente bash
   ```

   - **Explicación do comando `exec`:**
     - `docker exec`: Executa un comando dentro dun contedor en execución.
     - `-it`: Activa un modo interactivo con terminal (permite escribir comandos directamente).
     - `cliente`: Nome do contedor ao que queremos acceder.
     - `bash`: É o comando que se executa, neste caso, para abrir un terminal `bash`.

3. **Instalar `dig` no Cliente**
   Se `dig` non está dispoñible no contedor `ubuntu`, instálao cos seguintes comandos:
   ```bash
   apt update
   apt install -y dnsutils
   ```

4. **Realizar unha Proba DNS con `dig`**
   Unha vez instalado `dig`, podes usalo para facer unha consulta DNS ao servidor:
   ```bash
   dig @192.168.1.254 google.com
   ```

   - **Explicación do comando `dig`**:
     - `dig`: Ferramenta de consulta DNS.
     - `@192.168.1.254`: Especifica o servidor DNS ao que se envía a consulta (a IP de `asir_bind9`).
     - `google.com`: Dominio que se quere resolver.
   
   Se a configuración é correcta, `dig` debería devolver a dirección IP de `google.com` ou do dominio configurado no servidor `asir_bind9`.

### Detener e Eliminar os Contedores

Para detener e eliminar os contedores e recursos asociados, executa:

```bash
docker-compose down
```

---


## Arquivos de Configuración de Bind9

Os arquivos de configuración de Bind9 definen o comportamento xeral do servidor, as zonas xestionadas e os reenviadores DNS para consultas externas. Esta modularidade permite unha configuración máis clara e organizada.

### Arquivos de Configuración

1. **named.conf**
   - **Descrición**: Este é o arquivo de configuración principal de Bind9, que inclúe outras configuracións específicas en arquivos adicionais.
   - **Contido**:
     ```conf
     include "/etc/bind/named.conf.options";
     include "/etc/bind/named.conf.local";
     ```
   - **Propósito**: Organiza a configuración ao incluír os arquivos `named.conf.options` e `named.conf.local`, o que permite xestionar as configuracións en arquivos separados para unha administración máis clara e manexable.
   - **Uso no Servidor DNS**: Este arquivo permite ao servidor cargar todas as configuracións necesarias ao incluír tanto as opcións xerais (definidas en `named.conf.options`) como as zonas locais (definidas en `named.conf.local`).

2. **named.conf.default-zones**
   - **Descrición**: Este arquivo define varias zonas predeterminadas que son esenciais para calquera configuración DNS. Estas zonas permiten a resolución de enderezos especiais (como `localhost` e `127.0.0.1`) e as consultas inversas.
   - **Contido**:
     ```conf
     zone "." {
         type hint;
         file "/usr/share/dns/root.hints";
     };

     zone "localhost" {
         type master;
         file "/etc/bind/db.local";
     };

     zone "127.in-addr.arpa" {
         type master;
         file "/etc/bind/db.127";
     };

     zone "0.in-addr.arpa" {
         type master;
         file "/etc/bind/db.0";
     };

     zone "255.in-addr.arpa" {
         type master;
         file "/etc/bind/db.255";
     };
     ```
   - **Explicación das zonas**:
     - **Zona raíz (".")**: Configura o servidor para comunicarse cos servidores raíz de DNS, permitindo resolver nomes de dominio externos.
     - **Zona `localhost`**: Permite a resolución DNS para `localhost`, habilitando ao servidor a resolver este nome á IP local.
     - **Zonas inversas (`127.in-addr.arpa`, `0.in-addr.arpa`, `255.in-addr.arpa`)**: Permiten que o servidor realice consultas inversas (IP a nome de dominio), necesarias para a resolución inversa e conformidade coa RFC 1912.
   - **Uso no Servidor DNS**: Define as zonas locais e de resolución inversa que o servidor xestionará directamente, esenciais para a operatividade básica de Bind9.

3. **named.conf.local**
   - **Descrición**: Este arquivo define as zonas locais personalizadas, neste caso, a zona `asircastelao.int`, que o servidor xestionará como `master` ou servidor principal.
   - **Contido**:
     ```conf
     zone "asircastelao.int" {
         type master;
         file "/var/lib/bind/db.asircastelao.int";
         allow-query {
             any;
         };
     };
     ```
   - **Propósito**: Define a zona `asircastelao.int`, indicando que o servidor Bind9 actúa como `master` para esta zona específica.
   - **Opcións específicas**:
     - **file**: Especifica o arquivo de zona `/var/lib/bind/db.asircastelao.int`, onde se almacenan os rexistros DNS de `asircastelao.int`.
     - **allow-query**: Permite que calquera cliente realice consultas DNS para esta zona grazas á opción `any`.
   - **Uso no Servidor DNS**: Configura o servidor Bind9 para responder a consultas relacionadas coa zona `asircastelao.int`, o que fai que este contedor actúe como o servidor DNS autoritativo para esta zona.

4. **named.conf.options**
   - **Descrición**: Este arquivo establece as opcións xerais de Bind9, como os servidores reenviadores externos, o directorio de traballo e as políticas de consulta.
   - **Contido**:
     ```conf
     options {
         directory "/var/cache/bind";

         forwarders {
             8.8.8.8;
             1.1.1.1;
         };
         forward only;

         listen-on { any; };
         listen-on-v6 { any; };

         allow-query {
             any;
         };
     };
     ```
   - **Opcións importantes**:
     - **directory**: Define o directorio de traballo `/var/cache/bind`, onde Bind9 garda arquivos de caché e datos temporais.
     - **forwarders**: Configura servidores DNS externos (como Google `8.8.8.8` e Cloudflare `1.1.1.1`) aos que se reenviarán as consultas de dominios que o servidor non xestione.
     - **forward only**: Indica ao servidor que só use os reenviadores e non intente resolver nomes de dominio de maneira independente.
     - **listen-on** e **listen-on-v6**: Configuran o servidor para escoitar consultas en todas as interfaces, tanto para IPv4 como IPv6.
     - **allow-query**: Permite que calquera cliente realice consultas DNS ao servidor (`any` permite consultas de todas as IPs).
   - **Uso no Servidor DNS**: Configura o servidor para reenviar consultas non locais a servidores DNS externos, aceptar consultas en todas as interfaces e admitir solicitudes de calquera orixe.

---

Estes arquivos configuran e xestionan o entorno DNS no servidor Bind9, permitindo tanto a resolución local de dominios específicos (como `asircastelao.int`) como o reenvío de consultas non locais a servidores DNS externos. Esta configuración é ideal para un entorno controlado de probas de rede, como o que se implementa en Docker.

