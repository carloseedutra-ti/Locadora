# Locadora de Video Games - API Segura vs Vulnerável

Aplicação acadêmica em Java + Spring Boot que expõe uma API REST para gerenciar usuários, clientes, jogos e locações de uma locadora de video games. O projeto roda em dois modos distintos:

- **Modo Seguro (default)**: aplica boas práticas de Secure Coding.
- **Modo Vulnerável (laboratório)**: ativa vulnerabilidades propositalmente para aulas/demonstrações.

## Arquitetura

- **Stack**: Spring Boot 3, Spring Web, Spring Data JPA, Spring Security, H2 (memória).
- **Camadas**: `controller` (exposição REST) → `service` (regras + modo seguro/inseguro) → `repository` (JPA) → banco H2.
- **DTOs**: separados para requests/responses, evitando expor entidades diretamente.
- **Segurança**: filtro próprio (`TokenAuthenticationFilter`) + `TokenService` ciente do modo; `AppSecurityProperties` centraliza `app.security.mode`.
- **Utilidades**: `InputSanitizer` (JSoup) e `DataProtectionService` (AES) mostram contraste entre dados sanitizados/cifrados x sem proteção.

## Modelagem

| Entidade | Campos principais | Observações |
| --- | --- | --- |
| `Usuario` | `username`, `senha`, `role` (`ADMIN`/`ATENDENTE`) | Senha com BCrypt no modo seguro, texto puro no modo inseguro. |
| `Cliente` | `nome`, `email`, `documento`, `telefone` | Documento cifrado com AES no modo seguro. |
| `Jogo` | `titulo`, `genero`, `precoDiaria`, `disponivel` | Controle de disponibilidade e validações no modo seguro. |
| `Locacao` | `cliente`, `jogo`, `dataLocacao`, `valorTotal`, `status` | Calcula valor, controla devolução e multas. |

## Endpoints

| Método | Caminho | Função |
| --- | --- | --- |
| POST | `/usuarios` | Cria usuários com perfis. |
| POST | `/login` | Autentica e emite token (ou token fraco no modo inseguro). |
| GET | `/jogos` | Lista jogos. |
| POST | `/jogos` | Cadastra jogo (com/sem validação conforme modo). |
| PUT | `/jogos/{id}` | Atualiza jogo. |
| DELETE | `/jogos/{id}` | Remove jogo (controle de acesso apenas no modo seguro). |
| GET | `/clientes` | Lista clientes. |
| POST | `/clientes` | Cadastra cliente (documento cifrado apenas no modo seguro). |
| POST | `/locacoes` | Registra locação, bloqueando disponibilidade. |
| GET | `/locacoes` | Lista locações. |
| PUT | `/locacoes/{id}/devolucao` | Registra devolução e calcula multa. |

## Implementação (arquivos-chave)

- `pom.xml`: dependências e inclusão proposital de `commons-collections 3.2.1` (vulnerável).
- `src/main/java/com/example/locadora/config/*`: propriedades e `SecurityConfig` com `PasswordEncoder` sensível ao modo.
- `security/TokenService` + `TokenAuthenticationFilter`: comparam token seguro (UUID + expiração) vs token previsível e cabeçalho `X-Bypass-Auth`.
- `service/*`: cada serviço aplica regras, validações, cifragem e falhas propositais baseadas em `app.security.mode`.
- `exception/GlobalExceptionHandler`: oculta stack trace no modo seguro e expõe no vulnerável.
- `resources/application.properties`: configuração do H2 e modo padrão.

### Fluxo geral

1. Usuário cadastrado (`POST /usuarios`).
2. Login (`POST /login`) retorna token (Seguro: UUID+expiração / Inseguro: `username::ROLE`).
3. Token enviado via `Authorization: Bearer <token>`. Em modo inseguro é possível burlar com `X-Bypass-Auth`.
4. CRUD de jogos, clientes e locações seguem validações do serviço.

## Modo Vulnerável (o que muda)

| # | Vulnerabilidade | Onde/Código | Impacto | Exploração | Correção (modo seguro) |
|---|-----------------|-------------|---------|------------|------------------------|
|1|SQL Injection em login|`AuthService.login` usa concatenação de SQL|Bypass de autenticação|`username=admin' OR '1'='1`|Usar parâmetros (`Query#setParameter`) / `UsuarioRepository`|
|2|Cadastro sem sanitização (XSS)|`UsuarioService.criarUsuario` não usa `InputSanitizer`|Armazena `<script>`|Enviar `<script>alert(1)</script>` no nome|Sanitizar entrada com JSoup|
|3|Senha em texto puro|`SecurityConfig.passwordEncoder` retorna `NoOpPasswordEncoder`|Vazamento total do banco|Dump da tabela `usuarios`|BCrypt (`BCryptPasswordEncoder`)|
|4|Token sem expiração|`TokenService.emitToken` retorna `username::ROLE`|Session hijacking|Reuso infinito do token|UUID + expiração, guardar server-side|
|5|Controle de acesso fraco|`JogoService.remover` não verifica role|Usuário comum deleta jogos|`DELETE /jogos/1` como atendente|Checar `SecurityUtil.hasRole(ADMIN)`|
|6|Endpoint sem validação|`JogoService.validarJogo` não roda em modo inseguro|Persistência de dados maliciosos|JSON com `<img onerror>`|Aplicar validações e sanitização|
|7|Stack trace exposto|`GlobalExceptionHandler` adiciona `stackTrace`|Divulga detalhes internos|Qualquer erro revela classe/linha|Retornar mensagens genéricas|
|8|Dados sensíveis sem criptografia|`DataProtectionService.protect` retorna claro|Documento do cliente vazado|Dump tabela `clientes`|Cifrar AES + mascarar ao responder|
|9|Falha na validação de ID|`JogoService.atualizar` aceita `new Jogo()`|Manipulação de registros|`PUT /jogos/9999` cria objeto órfão|Forçar `findById` obrigatório|
|10|Dependência vulnerável|`pom.xml` inclui `commons-collections:3.2.1`|Passível de RCE conhecido|Ataques via gadget chains|Atualizar para versão corrigida|

## Alternar modo

1. Edite `src/main/resources/application.properties`.
2. Defina `app.security.mode=SECURE` ou `INSECURE`.
3. Reinicie a aplicação. Em produção mantenha **sempre** em `SECURE`.

## Como executar

```bash
# Requisitos: JDK 17+ e Maven
mvn spring-boot:run
```

A aplicação sobe em `http://localhost:8080`. Console H2: `http://localhost:8080/h2` (user `sa`, senha vazia).

## Interface Web

Um painel estático em `src/main/resources/static/index.html` gera formulários para todos os endpoints.

1. Acesse `http://localhost:8080/` após subir o Spring Boot.
2. Na seção **Autenticação & Segurança**:
   - Use *Cadastro de Usuário* para criar um administrador ou atendente.
   - Faça login e o token será exibido em "Token atual"; ele é enviado automaticamente nas próximas requisições.
   - Para demonstrar o modo inseguro marque "Enviar cabeçalho X-Bypass-Auth" e informe o valor desejado (só funciona com `app.security.mode=INSECURE`).
3. Nas seções **Jogos**, **Clientes** e **Locações** há formulários para POST/PUT/DELETE e botões de listagem que renderizam o JSON retornado em `<pre>` ao lado.
4. O painel "Console da API" mostra o histórico das respostas, útil para aulas de SAST/DAST.

Os arquivos de suporte estão em `src/main/resources/static/assets/styles.css` (estilos) e `.../assets/app.js` (fetch + integração com a API).

### Exemplos de requisições (cURL)

```bash
# Criar usuário ADMIN seguro
curl -X POST http://localhost:8080/usuarios   -H 'Content-Type: application/json'   -d '{"username":"admin","senha":"SenhaForte123","nome":"Administrador","role":"ADMIN","email":"admin@locadora.dev"}'

# Login
TOKEN=$(curl -s -X POST http://localhost:8080/login   -H 'Content-Type: application/json'   -d '{"username":"admin","senha":"SenhaForte123"}')

# Criar jogo
curl -X POST http://localhost:8080/jogos   -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json'   -d '{"titulo":"Zelda","genero":"Aventura","precoDiaria":12.5,"descricao":"Clássico"}'
```

No modo inseguro é possível usar `-H 'X-Bypass-Auth: hacker'` e enviar payloads com scripts.

## Execução de testes de segurança

### SAST (SonarQube)
1. Suba o SonarQube local ou em container.
2. Configure `sonar-project.properties` (exemplo abaixo) ou use parâmetros CLI.
3. Rode `mvn clean verify` e depois `sonar-scanner` ou `mvn sonar:sonar` com os parâmetros de host/token.

Exemplo mínimo `sonar-project.properties`:
```
sonar.projectKey=locadora
sonar.sources=src/main/java
sonar.java.binaries=target/classes
```

**Achados esperados** (modo inseguro):
- SQL Injection detectada em `AuthService`.
- Uso de `NoOpPasswordEncoder` e armazenamento de senha em texto.
- Exposição de stack trace e cabeçalhos permissivos.
- Dependência vulnerável (SCA) `commons-collections:3.2.1`.

### DAST (OWASP ZAP)
1. Inicie a API (`mvn spring-boot:run`).
2. No OWASP ZAP, configure contexto `http://localhost:8080`.
3. Configure usuário com token inseguro ou utilize `X-Bypass-Auth`.
4. Execute **Baseline Scan**:
   ```bash
   zap-baseline.py -t http://localhost:8080 -r zap-report.html
   ```
5. Para ataques autenticados, adicione cabeçalho `Authorization` nas opções avançadas ou use o comando:
   ```bash
   zap-baseline.py -t http://localhost:8080 -r zap-auth.html      -z "-config replacer.full_list(0).description=auth          -config replacer.full_list(0).matchtype=REQ_HEADER          -config replacer.full_list(0).matchstr=Authorization          -config replacer.full_list(0).replacement=Bearer hacker::ADMIN"
   ```

**Endpoints vulneráveis demonstráveis**:
- `/login`: SQL Injection.
- `/usuarios`: XSS persistente.
- `/jogos` (POST/PUT): falta de validação.
- `/jogos/{id}` (DELETE): acesso sem autorização.
- `/clientes`: dados sensíveis sem criptografia (respostas mostram documento claro).

## README Checklist

- [x] Descrição e arquitetura
- [x] Passos de execução e troca de modo
- [x] Lista das 10 vulnerabilidades com detalhes
- [x] Orientações SAST (SonarQube) e DAST (OWASP ZAP)
- [x] Exemplos de requisição JSON/cURL

## Próximos passos sugeridos

1. Acrescentar testes automatizados cobrindo os dois modos.
2. Integrar pipelines CI com Sonar e ZAP para reforçar o material didático.
3. Demonstrar correções progressivas removendo vulnerabilidades uma a uma.
