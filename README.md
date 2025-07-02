# Documentação da Pratica 01

Arquivos incluídos na raiz:
- .dockerignore
- Dockerfile (backend)
- docker-compose.yaml

Arquivos inclídos na pasta `frontend`:
- .dockerignore
- Dockerfile (frontend)
- nginx.conf

URLS com NGINX:
- frontend: http://localhost:8080
- backend: http://localhost:8080/api
- Banco de dados: 
    - Host: localhost
    - Porta: 5432
    - Banco: postgres
    - Usuario: postgres
    - Senha: secretpass

A estrutura do backend e frontend foi configurada por meio do nginx.conf, que atua como proxy reverso e balanceador de carga do backend, roteando as chamadas da interface /api para o backend Flask.

---

## Como rodar a aplicação com docker-compose

Na raiz do projeto se encontra o arquivo `docker-compose.yaml`:

1. Pelo terminal, dirija-se à raiz do projeto e execute o comando a seguir para subir os containers:

    ```bash
    docker-compose up --build
    ```

    O banco irá iniciar primeiro, logo após o backend e o frontend (aguardar alguns segundos para esperar o `Healthcheck` do banco).

2. Verificar os Serviços em Execução:

    ```bash
    docker-compose ps
    ```

    Após subir os serviços, serão expostos pelas urls:
    - frontend: http://localhost:8080
    - backend: http://localhost:8080/api
    
    Para verificar, basta abrir o navegador e digitar: http://localhost:8080, irá abrir a aplicação do GUESS_GAME

3. Desligar os Serviços:

    ```bash
    docker-compose down # mantem os servicos
    docker-compose down --volumes # exclui os volumes criados
    ```

4. Escalar o backend e reiniciar um serviço específico:

    ```bash
    docker-compose up -d --scale backend=3
    docker compose restart frontend
    ```
  
---

## Opções de Design Adotadas

### Rede `bridge`

Utilizei uma rede Docker do tipo `bridge` chamada `game-guess-network` para conectar os containers do projeto.  

A rede bridge é a padrão do Docker e oferece isolamento entre containers e do host, permitindo comunicação segura e eficiente entre os serviços através do DNS interno do Docker.  
Essa escolha facilita o gerenciamento e evita expor portas desnecessariamente, mantendo o ambiente controlado e organizado.

Portanto, para se conectar aos outros serviços do compose, é preciso usar seu dns, por isso em `FLASK_DB_HOST` se usa `postgresdb` e no `nginc.conf` se usa `http://backend:5000/`.

### Healthcheck no Serviço `postgresdb`

No docker-compose.yaml implementei um healthcheck no banco de dados utilizando o comando `pg_isready -U postgres` para verificar se o banco de dados PostgreSQL está pronto para receber conexões.  

Esse healthcheck realiza verificações periódicas para monitorar a saúde do banco de dados. Durante a inicialização dos containers, ele garante que o serviço do backend só seja iniciado quando o banco estiver realmente disponível e "saudável". Isso evita erros de conexão e reduz a necessidade de reinícios constantes do backend, aumentando a estabilidade da aplicação e inicialização dos containers.

Escolhi essa abordagem no compose ao invés do Dockerfile pois:

- Permite um controle mais detalhado sobre o momento em que o backend começa a rodar;  
- Evita falhas iniciais causadas por tentativas prematuras de conexão ao banco; 
- Mantém o Dockerfile mais limpo, sem a necessidade de adicionar scripts extras dentro da imagem;
- Facilita a manutenção e ajuste dessa espera diretamente no ambiente de orquestração;


### Dependência do Backend pelo Banco com `depends_on` com condição `service_healthy`

Configurei o serviço backend para depender da saúde do banco PostgreSQL usando `depends_on` com a condição `service_healthy`.  

Isso significa que o backend só será iniciado após o banco passar no healthcheck, garantindo que as conexões de banco estejam disponíveis antes do backend tentar acessá-las evitando falhas de acessos prematuros.


### Volume nomeado `postgres_data`

Optei por utilizar um volume nomeado chamado `postgres_data` em vez de mapear um caminho absoluto no host.  

Essa abordagem torna o projeto mais portátil e independente do ambiente onde será executado, já que o gerenciamento a localização física do volume é feito pelo Docker .  
Além disso, como o projeto é um "monorepo" com os arquivos do backend e o projeto do frontend na raiz, o volume nomeado evita possíveis dependências de caminhos locais e facilita a instalação e o uso em diferentes máquinas. 


### Uso de ARG no Dockerfile do Frontend

No frontend, utilizei `ARG` para definir variáveis de ambiente durante o build, `REACT_APP_BACKEND_URL`, ao invés de variáveis de runtime.

Essa abordagem torna o processo de build mais flexível, permitindo configurar endpoints externos sem precisar alterar o código-fonte. As variáveis são definidas no momento do build, por meio do arquivo docker-compose.yml, e já ficam incorporadas na imagem gerada. Isso garante consistência entre ambientes.
Se fossem usadas variáveis de ambiente (environment:), elas só seriam aplicadas em tempo de execução, o que não atenderia bem neste caso, já que o frontend precisa dessas informações no build para gerar os arquivos finais corretamente.

### Comando de inicialização do Backend

Optei por não usar um script externo (`start-backend.sh`) como entrypoint.  

Em vez disso, defini o comando flask run --host=0.0.0.0 --port=5000 diretamente no Dockerfile usando a instrução CMD, e configurei as variáveis de ambiente no próprio docker-compose.yml.

Essa abordagem torna o processo mais flexivel na inicialização e evita a necessidade de alterar arquivos dentro da imagem para ajustar variáveis ou comandos, facilitando atualizações, manutenções e futuros testes locais independente da máquina ou como está configurado o `start-backend.sh`.

### Arquivo nginx.conf

O arquivo `nginx.conf` que está na pasta frontend, representa a configuração do NGINX para servir o frontend e redirecionar as chamadas da API para o backend. Ele faz duas coisas principais:

- **Serve o frontend estático** a partir da pasta `/usr/share/nginx/html` especificada na imagem Docker, onde copio o build da aplicação frontend.
- **Proxy reverso para a API**: requisições que começam com `/api` são redirecionadas para `http://backend:5000`, container do serviço backend onde exponho pela porta 5000 no compose.

Permitindo que compartilhe a mesma porta `8080`.