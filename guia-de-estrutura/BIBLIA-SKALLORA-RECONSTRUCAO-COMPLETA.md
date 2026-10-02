# 📖 Bíblia Técnica do Skallora / Afillimetrics

> **Documento de reconstrução, arquitetura, operação e contexto para desenvolvedores e agentes de IA**  
> Estado documentado: **após o Commit 5 — `fix: improve sqlite concurrency and database lock handling`**  
> Base inspecionada: `ecossistema-commit-5-sqlite-concorrencia.zip`

---

## 0. Como usar este documento

Este arquivo foi escrito para atender quatro objetivos ao mesmo tempo:

1. permitir que um desenvolvedor que nunca viu o projeto entenda a arquitetura atual;
2. permitir reconstruir o projeto do zero, arquivo por arquivo, até chegar ao estado atual;
3. servir como manual de instalação, execução, manutenção, segurança e diagnóstico;
4. fornecer contexto suficiente para um agente de IA alterar o código sem destruir decisões arquiteturais importantes.

Este documento descreve **o que realmente existe no código atual**. Quando houver uma recomendação futura, ela será marcada como **Recomendação**, para não confundir intenção com implementação existente.

### Estado importante do projeto

O nome de produto atualmente usado pelo projeto é **Skallora**, mas o código ainda conserva vários identificadores históricos:

- projeto Django: `aflmain`;
- interface: várias páginas ainda exibem `Afillimetrics`;
- `render.yaml`: serviço chamado `afillimetrics`;
- README interno: `Afillimetrics — Ecossistema de Automação para Afiliados`.

Isso não impede o funcionamento. Uma renomeação global deve ser tratada como tarefa separada porque mexer em pacote Django, URLs, imports e infraestrutura apenas por estética cria risco desnecessário.

---

# 1. O que é o projeto

O Skallora/Afillimetrics é um ecossistema Django para automação de afiliados. O sistema centraliza:

- cadastro e autenticação de usuários;
- configuração individual de canais de divulgação;
- coleta automatizada de produtos;
- obtenção e validação de links de afiliado;
- captura de nome, preço antigo, preço atual e imagem;
- persistência no banco;
- geração automática de ofertas;
- templates de chamadas por categoria e palavra-chave;
- publicação em Telegram e WhatsApp;
- cache/republicação inteligente;
- alertas de configuração e falhas de canal;
- histórico de preço;
- ranking de produtos quentes;
- retomada de coleta por página/alvo;
- execução, pausa, retomada e parada segura do bot;
- exportação auxiliar para Excel/CSV;
- dashboard com métricas e termômetro de oportunidade.

Hoje o marketplace funcional é o **Magalu / Magazine Você**. A interface da Coleta Geral já apresenta Mercado Livre e Amazon como futuras integrações, mas os respectivos coletores não existem nesta base.

---

# 2. Arquitetura em uma frase

**Django cuida de usuários, banco, interface, configuração, ofertas e publicação; Selenium controla uma sessão real do Chrome para operar o marketplace Magalu e devolver produtos e links de afiliado ao Django.**

```text
Usuário
   │
   ▼
Django / interface web
   │
   ├── autenticação
   ├── configuração por usuário
   ├── templates
   ├── ofertas
   ├── dashboard
   └── controlador do bot
             │
             ▼
        thread Selenium
             │
             ▼
   Chrome remoto :9222
             │
             ▼
      Magazine Você
             │
             ├── produtos
             ├── preços
             ├── imagens
             └── link afiliado
             │
             ▼
        persistência
       ┌─────┴─────┐
       ▼           ▼
 Django DB      Excel/CSV
       │
       ▼
 geração de oferta
       │
       ▼
 publicação inteligente
    ┌───────┴────────┐
    ▼                ▼
Telegram          WhatsApp
```

---

# 3. Limites arquiteturais atuais que um novo dev precisa saber

## 3.1 Multiusuário não significa múltiplos bots simultâneos

Os dados principais possuem `owner` e são isolados por usuário. Porém o controlador do coletor é um singleton global:

```python
controlador_bot = ControladorBot()
```

Portanto, **uma única execução Selenium pode ficar ativa por vez no processo Django**. Se o bot estiver rodando para um usuário, outro usuário recebe resposta de “ocupado”.

Isso é uma decisão atual importante. Não confundir isolamento de dados com paralelismo de workers.

## 3.2 Selenium ainda é local/desktop

O bot atual abre/conecta ao Chrome usando:

```text
127.0.0.1:9222
```

com perfil persistente:

```text
C:\ChromeDebug
```

E o caminho do Chrome está hardcoded para Windows:

```text
C:\Program Files\Google\Chrome\Application\chrome.exe
```

Logo, o web app pode ser hospedado, mas **o coletor Selenium atual não está pronto para funcionar dentro do serviço web do Render como se fosse um navegador desktop local**.

### Recomendação futura

Separar o Selenium em worker dedicado, com fila, navegador e ciclo de vida próprios. PostgreSQL deve ser o banco compartilhado.

## 3.3 SQLite é apenas a opção local

O projeto implementou mitigação de locks com:

- timeout de 30 segundos;
- `PRAGMA busy_timeout`;
- `WAL`;
- `synchronous=NORMAL`;
- retry exponencial apenas para `database is locked`;
- descarte de conexões antigas entre tentativas.

Isso melhora muito o desenvolvimento local, mas não torna SQLite um substituto de PostgreSQL para múltiplos workers e alta concorrência.

## 3.4 Diversidade de produtos ainda é uma limitação conhecida

O código possui a lógica `familia_produto()` e `ordenar_produtos_diversos()`, além de `familias_recentes` em `ProgressoColeta`.

Entretanto, na validação real foi observado que ainda podem ocorrer sequências como:

```text
armário
armário
armário
```

Portanto, a diversidade semântica **não deve ser considerada resolvida**. Não remover a infraestrutura existente, mas também não assumir que ela atende ao comportamento final desejado.

## 3.5 Nome do pacote `magalu_bot`

Na base atual existe:

```text
magalu_bot/__init___.py
```

com três `_` no final, em vez do nome convencional:

```text
magalu_bot/__init__.py
```

Imports podem funcionar graças ao mecanismo de namespace packages do Python, mas isso é uma inconsistência estrutural real.

### Recomendação

Criar/renomear para `magalu_bot/__init__.py` em commit isolado e testar todos os imports.

---

# 4. Estrutura atual completa

A estrutura relevante dentro de `aflmain/` é:

```text
aflmain/
├── .env.example
├── .gitignore
├── README.md
├── build.sh
├── manage.py
├── render.yaml
├── requirements.txt
│
├── aflmain/
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
│
├── core/
│   ├── __init__.py
│   ├── admin.py
│   ├── alerts.py
│   ├── apps.py
│   ├── config_service.py
│   ├── context_processors.py
│   ├── db.py
│   ├── fields.py
│   ├── marketing.py
│   ├── models.py
│   ├── opportunity.py
│   ├── publicacao.py
│   ├── services.py
│   ├── template_service.py
│   ├── tests.py
│   ├── urls.py
│   ├── views.py
│   │
│   ├── management/
│   │   └── commands/
│   │       ├── configurar_canais.py
│   │       ├── migrar_credenciais.py
│   │       └── publicar_ofertas.py
│   │
│   ├── migrations/
│   │   ├── 0001_initial.py
│   │   └── 0002_progressocoleta.py
│   │
│   ├── static/core/
│   │   ├── sidebar.css
│   │   ├── ui-loader.css
│   │   └── ui-loader.js
│   │
│   └── templates/core/
│       ├── _alertas.html
│       ├── _sidebar.html
│       ├── cadastro.html
│       ├── coleta_geral.html
│       ├── configuracoes.html
│       ├── dashboard.html
│       ├── home.html
│       ├── index.html
│       ├── login.html
│       ├── magalu_bot.html
│       ├── ofertas.html
│       ├── produtos_quentes.html
│       └── templates_oferta.html
│
├── magalu_bot/
│   ├── __init___.py
│   ├── bot.py
│   ├── bot_controller.py
│   ├── controlador.py
│   ├── estados.py
│   │
│   ├── automacao/
│   │   ├── __init__.py
│   │   ├── afiliados.py
│   │   ├── categorias.py
│   │   └── logger.py
│   │
│   ├── config/
│   │   ├── __init__.py
│   │   └── config.py
│   │
│   ├── persistencia/
│   │   ├── __init__.py
│   │   ├── django_db.py
│   │   └── excel.py
│   │
│   ├── relatorios/
│   │   ├── __init__.py
│   │   └── relatorios.py
│   │
│   └── tests/
│       ├── __init__.py
│       └── test_afiliados.py
│
└── scripts/
    ├── configurar_local.py
    └── limpar_local.py
```

Na raiz do repositório existe ainda outro `README.md` e um `build.sh`. O projeto Django operacional está dentro da pasta `aflmain/`.

---

# 5. Dependências

O `requirements.txt` atual contém:

```text
Django>=5.2,<7
selenium>=4.20
webdriver-manager>=4.0
pandas>=2.0
openpyxl>=3.1
python-decouple>=3.8
dj-database-url>=2.1
whitenoise>=6.6
requests>=2.31
beautifulsoup4>=4.12
gunicorn>=23.0
cryptography>=43.0
```

## 5.1 Para que serve cada dependência

| Dependência | Função no projeto |
|---|---|
| `Django` | Framework web, autenticação, ORM, views, templates, migrations e admin. |
| `selenium` | Controle do Chrome e automação do Magazine Você/WhatsApp Web. |
| `webdriver-manager` | Dependência histórica para gerenciamento de driver. O fluxo atual conecta ao Chrome por remote debugging e não depende diretamente dele em `bot.py`. |
| `pandas` | Agregações no dashboard, principalmente dados para gráficos por categoria e marketplace. |
| `openpyxl` | Criação e atualização do relatório Excel legado. |
| `python-decouple` | Leitura de variáveis do `.env`, especialmente em `settings.py` e comandos. |
| `dj-database-url` | Converte `DATABASE_URL` em configuração de banco Django. |
| `whitenoise` | Entrega arquivos estáticos no deploy web sem servidor dedicado de assets. |
| `requests` | Chamadas HTTP para Telegram, WhatsApp API e download da imagem de produto. |
| `beautifulsoup4` | Está instalada, mas não é peça central do fluxo atual inspecionado. |
| `gunicorn` | Servidor WSGI utilizado no Render. |
| `cryptography` | `Fernet` para criptografar tokens sensíveis de canais. |

## 5.2 Versão de Python

O `render.yaml` fixa:

```text
Python 3.12.8
```

O `requirements.txt` aceita Django de `5.2` até menor que `7`. O cabeçalho de `settings.py` informa que o projeto foi gerado originalmente com Django `6.1`.

### Recomendação operacional

Use **Python 3.12** para padronizar desenvolvimento e produção, evitando diferenças desnecessárias de comportamento entre Python 3.12 e ambientes mais novos.

---

# 6. Variáveis de ambiente

O `.env.example` atual é:

```env
SECRET_KEY=
CREDENTIAL_ENCRYPTION_KEY=change-me
DEBUG=True
ALLOWED_HOSTS=127.0.0.1,localhost
CSRF_TRUSTED_ORIGINS=
TIME_ZONE=America/Sao_Paulo
DATABASE_URL=sqlite:///db.sqlite3
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
WHATSAPP_API_TOKEN=
WHATSAPP_API_ENDPOINT=
WHATSAPP_DESTINATION=

AUTO_PUBLICAR_OFERTAS=false
CANAIS_AUTOMATICOS=TELEGRAM
CACHE_RETENCAO_HOURS=120
CAMPANHA_SAZONAL=NENHUM
TURNO_PADRAO=
```

## 6.1 `SECRET_KEY`

É a chave interna do Django. Ela deve ser estável enquanto o ambiente existir.

Responsabilidades:

- segurança interna do Django;
- assinatura de dados;
- sessões/tokens e mecanismos que dependem da chave do projeto;
- descriptografia **somente de credenciais legadas** com prefixo `enc::` durante a transição histórica.

Ela **não deve ser usada como a chave atual dos tokens Telegram/WhatsApp**.

## 6.2 `CREDENTIAL_ENCRYPTION_KEY`

É a chave separada para credenciais da aplicação.

Responsabilidades:

- token do Telegram;
- token de WhatsApp/API;
- demais segredos armazenados em campos `EncryptedTextField`.

Novos valores são armazenados com prefixo:

```text
enc2::
```

A chave não deve ser regenerada depois que houver dados criptografados no banco.

## 6.3 Regra crítica para backup

Para conseguir restaurar uma instalação completa, é necessário preservar **ao mesmo tempo**:

```text
.env
banco de dados
```

Backup apenas do banco sem `CREDENTIAL_ENCRYPTION_KEY` pode deixar os tokens impossíveis de descriptografar.

## 6.4 `DATABASE_URL`

Local:

```env
DATABASE_URL=sqlite:///db.sqlite3
```

Produção:

```env
DATABASE_URL=postgresql://usuario:senha@host:5432/banco
```

## 6.5 Credenciais de canais no `.env`

As variáveis `TELEGRAM_*` e `WHATSAPP_*` permanecem como referência/fallback de desenvolvimento e podem ser importadas pelo comando `configurar_canais` em registros sem `owner`.

O fluxo principal multiusuário configura os canais em `/configuracoes/` e salva as credenciais no banco ligadas ao usuário.

---

# 7. Segurança das credenciais

O arquivo responsável é:

```text
core/fields.py
```

Ele define `EncryptedTextField`.

## 7.1 Formato antigo

```text
enc::...
```

É descriptografado usando uma chave derivada de `SECRET_KEY`.

## 7.2 Formato atual

```text
enc2::...
```

É criptografado usando uma chave derivada de `CREDENTIAL_ENCRYPTION_KEY`.

## 7.3 Derivação

A string da configuração é processada com SHA-256 e transformada em uma chave compatível com Fernet.

O objetivo é permitir que o usuário coloque no `.env` um segredo de alta entropia sem precisar fornecer diretamente uma chave Fernet codificada.

## 7.4 Migração de credenciais antigas

Existe:

```text
core/management/commands/migrar_credenciais.py
```

Uso:

```bash
python manage.py migrar_credenciais
```

O comando lê cada token usando o suporte legado e o salva novamente. O `get_prep_value()` converte então para `enc2::`.

### Ordem correta para rotação

1. manter a `SECRET_KEY` antiga;
2. configurar `CREDENTIAL_ENCRYPTION_KEY` nova e estável;
3. fazer backup do banco e `.env`;
4. executar `python manage.py migrar_credenciais`;
5. conferir que os tokens continuam válidos;
6. só então considerar rotação da `SECRET_KEY`.

---

# 8. Reconstrução do absoluto zero

Esta seção mostra como reconstruir a arquitetura atual começando em uma pasta vazia.

---

## Etapa 1 — criar o repositório

```bash
mkdir ecossistema
cd ecossistema
mkdir aflmain
cd aflmain
```

Opcionalmente:

```bash
git init
```

Crie `.gitignore` desde o início para impedir commit de segredos e artefatos locais.

Arquivo:

```text
.gitignore
```

Responsabilidade:

- excluir `.env`;
- excluir SQLite local;
- excluir bytecode/cache;
- excluir `staticfiles/` gerado;
- excluir logs;
- excluir virtualenv;
- excluir sidecars SQLite.

O atual contém:

```gitignore
.env
db.sqlite3
__pycache__/
*.pyc
staticfiles/
logs/
.venv/

# Inicializador local (nao versionar)
iniciar_local.bat

*.sqlite3-journal
*.sqlite3-wal
*.sqlite3-shm
.pytest_cache/
.coverage
```

> O BAT discutido no fluxo atual chama-se `iniciar.bat`. Se ele não deve ser versionado, inclua também `iniciar.bat` no `.gitignore`.

---

## Etapa 2 — instalar Python e criar o virtualenv

Recomendado:

```text
Python 3.12.x
```

Windows:

```bat
py -3.12 -m venv venv
venv\Scripts\activate
```

Linux/macOS:

```bash
python3.12 -m venv venv
source venv/bin/activate
```

Atualize pip:

```bash
python -m pip install --upgrade pip
```

---

## Etapa 3 — criar `requirements.txt`

Crie o arquivo com as dependências listadas na seção 5 e instale:

```bash
pip install -r requirements.txt
```

---

## Etapa 4 — criar o projeto Django

Dentro de `aflmain/`:

```bash
django-admin startproject aflmain .
```

Isso cria:

```text
manage.py
aflmain/__init__.py
aflmain/settings.py
aflmain/urls.py
aflmain/asgi.py
aflmain/wsgi.py
```

### `manage.py`

É o ponto de entrada dos comandos Django:

```bash
python manage.py migrate
python manage.py runserver
python manage.py test
python manage.py createsuperuser
```

### `aflmain/settings.py`

Configuração central da aplicação.

Na arquitetura atual ele deve:

- ler `.env` com `python-decouple`;
- configurar `SECRET_KEY`;
- configurar `CREDENTIAL_ENCRYPTION_KEY`;
- aceitar `DATABASE_URL`;
- ativar o app `core`;
- ativar WhiteNoise;
- registrar `core.context_processors.alertas_usuario`;
- configurar timezone;
- configurar static files;
- habilitar cookies seguros fora de DEBUG;
- configurar timeout especial quando o banco for SQLite.

### `aflmain/urls.py`

Roteador raiz:

```text
/admin/ → Django admin
/       → core.urls
```

### `aflmain/wsgi.py`

Entrada utilizada por Gunicorn no deploy tradicional WSGI.

### `aflmain/asgi.py`

Entrada ASGI padrão. Não existe uso específico de WebSocket no código atual.

### `aflmain/__init__.py`

Marca o diretório como package Python. Atualmente vazio.

---

## Etapa 5 — criar o app Django `core`

```bash
python manage.py startapp core
```

Adicione em `INSTALLED_APPS`:

```python
'core.apps.CoreConfig',
```

O `core` é o centro da aplicação. Na versão atual **não há apps Django separados chamados `accounts` ou `products`**. Usuários, produtos, ofertas, alertas e configuração ficam integrados ao `core`.

---

# 9. Arquivo por arquivo — `core`

## 9.1 `core/apps.py`

Define `CoreConfig`.

O método `ready()` é usado para garantir o carregamento das integrações que precisam ser registradas quando Django inicializa, especialmente o sinal de configuração SQLite em `core.db`.

A ideia é que `connection_created` esteja conectado antes das conexões começarem a ser usadas.

---

## 9.2 `core/models.py`

É o mapa principal do domínio.

### `Produto`

Representa o produto coletado.

Campos principais:

- `owner`: usuário dono;
- `marketplace`;
- `categoria`;
- `nome`;
- `url_produto`;
- `preco_anterior`;
- `preco_atual`;
- `imagem_url`;
- timestamps.

Restrição importante:

```text
owner + url_produto deve ser único
```

Assim, dois usuários podem possuir o mesmo produto, mas o mesmo usuário não duplica a mesma URL.

### `Afiliado`

Relação `OneToOne` com `Produto`.

Guarda:

- `link_afiliado`;
- `status` (`OK`, `REVISAR` etc. conforme fluxo);
- timestamps.

Separar Produto e Afiliado permite preservar o produto mesmo quando ainda não há link válido.

### `Execucao`

Representa uma execução do bot.

Guarda:

- dono;
- início/fim;
- estado;
- contadores;
- categorias;
- keywords;
- quais categorias/keywords estão em loop;
- erro final.

É a base para histórico, dashboard, polling e logs.

### `Mensagem`

Representa uma tentativa/publicação por canal.

Guarda:

- produto;
- canal;
- status;
- conteúdo;
- data de envio;
- erro;
- snapshot de preço anterior;
- preço atual;
- desconto no momento da publicação.

O snapshot é fundamental para decidir se um produto mudou o suficiente para ser republicado antes do cache expirar.

### `Oferta`

É a entidade editorial gerada a partir de um Produto.

Estados:

```text
RASCUNHO
PRONTA
ENVIADA
ARQUIVADA
```

Guarda título, mensagem e desconto calculado.

### `ConfiguracaoCanal`

Configuração de Telegram/WhatsApp por usuário.

Campos:

- `owner`;
- `canal`;
- `ativo`;
- `destino`;
- `token` criptografado;
- `endpoint`.

Restrição:

```text
owner + canal único
```

### `LogExecucao`

Log persistido no banco por execução.

Níveis:

```text
INFO
OK
WARNING
ERROR
```

É consumido pelo endpoint de logs e pela interface do bot.

### `HistoricoPreco`

Registra preço coletado ao longo do tempo para o mesmo produto.

Serve para:

- detectar queda de preço;
- alimentar score de produtos quentes;
- futuras análises temporais.

### `ConfiguracaoAutomacao`

`OneToOne` por usuário.

Controla:

- publicação automática;
- canais automáticos;
- horas de retenção/cache;
- campanha sazonal;
- turno padrão.

### `TemplateOferta`

Templates por usuário.

Tipos:

```text
CATEGORIA
KEYWORD
```

Campos relevantes:

- `chave`;
- lista JSON de chamadas;
- `divulgar_sem_template`;
- `nativo`;
- `ativo`.

### `AlertaSistema`

Alerta persistente e contextual.

Exemplos:

- Telegram não configurado;
- token recusado;
- API temporariamente indisponível.

A constraint `owner + codigo` evita criar dezenas de alertas iguais.

### `ProdutoQuente`

Ranking calculado para Produtos Quentes.

Guarda:

- produto;
- usuário;
- score;
- motivo;
- data de cálculo.

### `PerfilUsuario`

Localização e timezone.

Campos:

- país;
- estado;
- cidade;
- timezone.

Atualmente o principal uso funcional é timezone para mensagens por turno e termômetro de oportunidade.

### `ProgressoColeta`

Cursor persistente por usuário/alvo.

Chave lógica:

```text
owner + marketplace + tipo_alvo + alvo
```

Guarda:

- próxima página;
- últimas famílias semânticas;
- timestamp.

Objetivo: recomeçar uma execução recente de uma página avançada em vez de voltar sempre para a página 1.

---

## 9.3 `core/migrations/0001_initial.py`

Cria o conjunto inicial de tabelas do domínio.

Nunca deve ser “reescrita casualmente” em um projeto já utilizado. Migrations são histórico do esquema.

---

## 9.4 `core/migrations/0002_progressocoleta.py`

Adiciona `ProgressoColeta`.

É a migration do mecanismo de continuidade/retomada da coleta.

Aplicação:

```bash
python manage.py migrate
```

Não se deve usar `makemigrations` automaticamente em toda inicialização. `makemigrations` é etapa de desenvolvimento; `migrate` é etapa de aplicação do histórico versionado.

---

## 9.5 `core/fields.py`

Implementa criptografia transparente no ORM.

`EncryptedTextField` funciona assim:

```text
salvar texto comum
      ↓
get_prep_value
      ↓
Fernet + CREDENTIAL_ENCRYPTION_KEY
      ↓
enc2::<ciphertext>
      ↓
banco
```

Na leitura:

```text
enc2:: → chave nova
enc::  → SECRET_KEY legada
texto sem prefixo → devolvido como está
```

Não registrar tokens em logs.

---

## 9.6 `core/db.py`

Camada de proteção contra locks transitórios do SQLite.

### `configurar_sqlite()`

Executado quando uma conexão SQLite é criada.

Configura:

```sql
PRAGMA busy_timeout=30000;
PRAGMA synchronous=NORMAL;
PRAGMA journal_mode=WAL;
```

### `banco_esta_bloqueado()`

Só considera retry se a mensagem for realmente:

```text
database is locked
database table is locked
```

### `executar_com_retry_db()`

Tenta a operação até 6 vezes por padrão, com backoff crescente.

Não engole outros `OperationalError`.

Essa distinção é importante: retry indiscriminado esconderia defeitos reais do banco.

---

## 9.7 `core/config_service.py`

Cria/carrega `ConfiguracaoAutomacao` de cada usuário.

Quando um usuário ainda não possui o registro, os defaults podem vir de variáveis de ambiente:

- `AUTO_PUBLICAR_OFERTAS`;
- `CANAIS_AUTOMATICOS`;
- `CACHE_RETENCAO_HOURS`;
- `CAMPANHA_SAZONAL`;
- `TURNO_PADRAO`.

Depois disso, o registro persistido no banco vira a fonte principal.

---

## 9.8 `core/alerts.py`

Centraliza classificação e persistência de alertas de canais.

Classifica erros aproximadamente em:

### Canal desativado

Nível `WARNING`.

A coleta continua, publicação não.

### Configuração incompleta

Nível `ERROR`.

Exemplo:

```text
token/chat_id ausente
```

### Credenciais recusadas

Detecta sinais como:

```text
401
403
unauthorized
forbidden
invalid token
bot blocked
```

### Indisponibilidade temporária

Detecta:

```text
timeout
connection
500
502
503
504
unavailable
```

### Segurança

O detalhe técnico bruto não é colocado diretamente no alerta da UI. A UI recebe uma mensagem sanitizada.

### Resolução

Um envio bem-sucedido chama `resolver_alerta_canal()`.

A página de configurações também resolve alertas de configuração imediatamente quando o usuário salva uma configuração localmente válida.

---

## 9.9 `core/context_processors.py`

Disponibiliza alertas não resolvidos do usuário para os templates.

Isso permite incluir:

```django
{% include "core/_alertas.html" %}
```

em diferentes páginas sem duplicar query/lógica.

---

## 9.10 `core/template_service.py`

Responsável por templates nativos, templates de keyword e ranking de produtos quentes.

### `garantir_templates_nativos(owner)`

Cria templates padrão para:

```text
cozinha
quarto
sala
banheiro
acessórios
```

Cada usuário possui seus próprios registros.

### `keywords_sem_template()`

Antes de iniciar coleta por keyword, verifica se cada keyword:

- possui chamadas configuradas; ou
- foi explicitamente marcada como “divulgar sem template”.

### `garantir_registros_keywords()`

Cria registro pendente/inativo quando uma keyword ainda não foi configurada.

### Produtos Quentes

`atualizar_produtos_quentes()` recalcula no máximo a cada 5 dias, salvo quando `forcar=True`.

Score atual considera:

- desconto;
- recência;
- queda desde preço anterior;
- existência de preço atual.

A lista mantém os 100 melhores.

---

## 9.11 `core/marketing.py`

É o motor de copy e composição da oferta.

Ele contém:

- chamadas genéricas;
- chamadas por categoria;
- gatilhos por palavras-chave específicas;
- textos de urgência por turno;
- detecção de turno pelo timezone do usuário;
- cálculo de desconto;
- formatação monetária;
- compatibilidade entre produto e contexto;
- montagem final HTML.

### Compatibilidade de template

O Commit 3 adicionou `CATEGORY_SIGNALS` e `avaliar_compatibilidade_contexto()`.

Exemplo de problema evitado:

```text
Produto: Jogo de Panelas
Contexto: banheiro
```

O sistema detecta sinais de cozinha. Se o contexto for claramente banheiro, ele não usa a chamada contextual de banheiro.

Ele pode usar:

- uma correspondência segura no próprio nome;
- ou uma chamada genérica.

A validação é conservadora: produto sem evidência suficiente não é rejeitado.

### `gerar_mensagem()`

Estrutura típica:

```text
[chamada]

[ganchos/turno]

📦 NOME DO PRODUTO

❌ De: R$ ...
✅ Por: R$ ... (...% OFF)

🛒 COMPRE AQUI:
👉 link afiliado
```

O nome e o link são escapados para HTML porque o Telegram usa `parse_mode=HTML`.

---

## 9.12 `core/services.py`

Camada de serviço mais simples para ofertas.

Responsabilidades:

- formatar oferta;
- criar `Oferta`;
- calcular desconto;
- delegar envio para Telegram/WhatsApp.

O fluxo de publicação inteligente mais completo fica em `core/publicacao.py`.

---

## 9.13 `core/publicacao.py`

É o núcleo da publicação multicanal.

### `avaliar_elegibilidade_publicacao()`

Evita spam e decide quando um produto pode voltar a ser publicado.

Regras atuais:

1. nunca enviado → elegível;
2. cache expirou → elegível;
3. menos de uma hora → não republica mesmo que o preço tenha mudado;
4. queda de preço de pelo menos 3% → pode antecipar;
5. aumento do desconto em pelo menos 3 pontos percentuais → pode antecipar;
6. Produto Quente com score >= 70 → janela reduzida, respeitando cooldown;
7. caso contrário → `CACHE`.

Default de retenção:

```text
120 horas
```

configurável por usuário.

### Telegram

A configuração vem do usuário autenticado (`ConfiguracaoCanal`).

O envio tenta incluir imagem.

Se a mensagem tiver até 1024 caracteres, usa caption da foto.

Se for maior:

1. envia foto sem caption;
2. envia texto em `sendMessage`.

Se o download/envio da foto falhar, ainda tenta publicar a mensagem de texto.

O texto é limitado a 4096 caracteres sem cortar HTML de maneira perigosa.

### WhatsApp API

Envia JSON com:

```text
messaging_product = whatsapp
to
type = text
text.body
```

usando bearer token e endpoint configurado.

### WhatsApp Web

Quando não existe API válida mas existe um `driver` Selenium, o sistema consegue procurar uma aba de `web.whatsapp.com`, selecionar o destino e enviar o texto.

### Falha não invalida coleta

Esse é um princípio estrutural:

```text
produto coletado/link afiliado válido
            │
            ├── salva produto
            ├── salva afiliado
            ├── cria oferta
            └── tenta publicação
                       │
                       └── se falhar, coleta continua
```

A falha gera `Mensagem(status=FALHOU)` e `AlertaSistema`, mas não apaga o produto.

---

## 9.14 `core/opportunity.py`

Calcula o **Termômetro de Oportunidade** da Home.

Produz uma leitura de oportunidade a partir do horário/fuso do usuário e retorna score/label/contexto visual para o dashboard.

Não confundir com score de `ProdutoQuente`: são recursos distintos.

---

## 9.15 `core/views.py`

É o maior concentrador HTTP do projeto.

### Rotas públicas

#### `index`

Landing page. Usuário autenticado é redirecionado para Home.

#### `login`

Autentica usando e-mail armazenado como `username`.

#### `cadastro`

Valida:

- nome;
- e-mail;
- senha;
- confirmação;
- unicidade;
- validadores nativos do Django.

Cria `User` + `PerfilUsuario` dentro de `transaction.atomic()`.

#### `logout_view`

Logout via POST e retorno à landing.

### Rotas autenticadas

#### `home`

Monta:

- total de produtos;
- links afiliados;
- ofertas;
- ofertas enviadas;
- última execução;
- dados para gráficos;
- termômetro de oportunidade;
- informação se há coleta anterior configurada.

Usa pandas para agrupar categoria e marketplace.

#### `coleta_geral`

Interface simplificada multi-marketplace.

Hoje:

- Magalu funcional;
- Mercado Livre desabilitado;
- Amazon desabilitado.

Permite categorias nativas e personalizadas.

#### `criar_categoria_personalizada_view`

Endpoint JSON para criar keyword/template sem sair da coleta.

#### `magalu_bot`

Central específica do Magalu.

#### `iniciar_bot_view`

Valida payload, isolamento de usuário, templates pendentes e chama:

```python
controlador_bot.iniciar(...)
```

Retorna avisos não bloqueantes se Telegram/WhatsApp estiverem incompletos.

#### `controlar_coleta_dashboard_view`

A Home possui controle simplificado:

- se bot está rodando → solicita parada;
- se está parado → tenta iniciar com a última configuração válida.

#### `pausar_bot_view`

Solicita pausa. A pausa real ocorre em ponto seguro do bot.

#### `retomar_bot_view`

Continua a mesma thread pausada.

#### `parar_bot_view`

Solicita parada segura e idempotente.

#### `status_bot_view`

Entrega estado para polling da interface.

#### `logs_bot_view`

Retorna logs incrementais após determinado ID (`since`).

#### `dashboard`

Mantido por compatibilidade e redireciona para Home.

#### `ofertas_view`

Após o Commit 4, não é mais uma página redundante. Redireciona para:

```text
/ofertas/templates/#ofertas-geradas
```

#### `templates_oferta_view`

CRUD simplificado de templates e edição de ofertas recentes.

#### `produtos_quentes_view`

Atualiza ranking quando necessário e permite filtro por categoria.

#### `configuracoes_view`

Configura:

- localização/timezone;
- Telegram;
- WhatsApp;
- auto-publicação;
- canais automáticos;
- cache;
- campanha;
- turno.

Tokens já salvos não são reenviados para a tela.

#### `resolver_alerta_view`

Permite resolver/dispensar alerta pertencente ao próprio usuário.

#### `enviar_oferta_view`

Envio manual de uma oferta em canal escolhido.

---

## 9.16 `core/urls.py`

Mapa HTTP atual:

| URL | Função |
|---|---|
| `/` | landing |
| `/login/` | login |
| `/cadastro/` | cadastro |
| `/home/` | home operacional |
| `/coleta/` | coleta geral |
| `/coleta/categoria-personalizada/` | cria keyword/template inline |
| `/logout/` | logout |
| `/magalu/` | central Magalu |
| `/magalu/iniciar/` | inicia bot |
| `/magalu/pausar/` | pausa |
| `/magalu/retomar/` | retoma |
| `/magalu/parar/` | para |
| `/magalu/status/` | polling de status |
| `/magalu/logs/` | logs incrementais |
| `/dashboard/` | compatibilidade → Home |
| `/dashboard/coleta/controlar/` | iniciar/parar pela Home |
| `/ofertas/` | atalho para seção de ofertas geradas |
| `/ofertas/templates/` | central de templates/ofertas |
| `/produtos-quentes/` | ranking |
| `/configuracoes/` | canais e automação |
| `/alertas/<id>/resolver/` | resolver alerta |
| `/ofertas/<id>/enviar/` | envio manual |

---

# 10. Templates e frontend

O frontend atual usa principalmente:

- Django Templates;
- Bootstrap 5.3.3 por CDN;
- Bootstrap Icons;
- CSS inline em várias telas;
- JavaScript nativo;
- polling por `fetch()` para status/logs.

Não há framework SPA.

## `_sidebar.html`

Sidebar compartilhada entre páginas autenticadas. Centraliza navegação e estado ativo.

## `_alertas.html`

Renderiza alertas persistentes do usuário.

## `index.html`

Landing pública.

## `login.html`

Login por e-mail e senha.

## `cadastro.html`

Cadastro e coleta de país/estado/cidade/timezone.

## `home.html`

Dashboard operacional principal.

Inclui métricas, gráficos, termômetro e controle simplificado da coleta.

## `dashboard.html`

Arquivo legado/compatibilidade. A view atual redireciona `/dashboard/` para `home`.

## `coleta_geral.html`

Fluxo simplificado:

1. marketplace;
2. categorias nativas/personalizadas;
3. loop;
4. iniciar/parar.

Permite criar categoria personalizada via modal sem abandonar a página.

## `magalu_bot.html`

Interface operacional detalhada do bot.

É um arquivo grande porque concentra:

- controles;
- seleção de categorias;
- categorias personalizadas;
- keywords;
- loop;
- status;
- progresso;
- logs;
- polling;
- modais;
- JavaScript de interação.

Para uma refatoração futura, esse JS é candidato forte a ser extraído para arquivos estáticos separados.

## `configuracoes.html`

Configuração de canais e automação.

Possui feedback visual de sucesso e fechamento por `X`. O alerta de sucesso é configurado para desaparecer automaticamente.

## `templates_oferta.html`

Duas responsabilidades:

1. gerenciar templates;
2. editar ofertas geradas recentemente.

A seção de ofertas recentes possui:

```html
id="ofertas-geradas"
```

O menu “Ofertas” aponta diretamente para essa âncora.

## `ofertas.html`

Existe no repositório, mas o endpoint `/ofertas/` atual redireciona para a seção útil de `templates_oferta.html`. Deve ser tratado como legado até eventual remoção deliberada.

## `produtos_quentes.html`

Lista ranking, score, motivo, desconto real e link afiliado.

## `static/core/sidebar.css`

CSS compartilhado da sidebar.

## `static/core/ui-loader.css` e `ui-loader.js`

Na base inspecionada ambos estão vazios, apesar de serem referenciados por páginas. Isso é uma inconsistência atual. Não presumir que contêm o loader de versões anteriores.

---

# 11. O módulo `magalu_bot`

`magalu_bot` não é um Django app registrado. É um módulo Python especializado, chamado pelo `core`.

A divisão atual é:

```text
controlador.py     → lifecycle/thread/controle externo
bot.py             → orquestração completa Selenium
automacao/         → scraping e coleta
persistencia/      → DB + Excel/CSV
config/            → constantes
relatorios/        → saída textual
estados.py         → máquina de estados
```

---

# 12. `magalu_bot/config/config.py`

Contém constantes do bot.

Entre elas:

- URL base;
- URL de login;
- debugger do Chrome;
- timeout de login;
- limite por categoria;
- categorias principais.

Esse arquivo deve guardar configuração operacional estática, não tokens sensíveis.

---

# 13. `magalu_bot/estados.py`

Define `EstadoBot` e `GerenciadorEstados`.

Estados usados pelo fluxo incluem:

```text
INICIANDO
VERIFICANDO_LOGIN
AGUARDANDO_LOGIN
LOGADO
EXECUTANDO
PAUSANDO
PAUSADO
PARANDO
FINALIZADO
PARADO
ERRO
```

A UI converte esses estados em status visuais.

O `GerenciadorEstados` centraliza mudança de estado em vez de deixar strings soltas espalhadas pelo código.

---

# 14. `magalu_bot/controlador.py`

É a ponte entre as views Django e `bot.py`.

## `ControladorBot`

Mantém:

- thread atual;
- events de pause/stop;
- erro;
- configuração atual;
- usuário dono;
- execução persistida.

### `iniciar()`

Cria `Execucao`, prepara eventos e abre uma thread em segundo plano.

Não bloqueia a requisição HTTP esperando o Selenium terminar.

### `_executar()`

Wrapper da thread que chama o motor e trata lifecycle/cleanup.

### `pausar()`

Liga `pausar_evento`.

A thread só pausa em um ponto seguro.

### `retomar()`

Limpa `pausar_evento` e continua a mesma thread.

### `parar()`

Liga `parar_evento`.

Também libera uma eventual pausa para permitir que a thread chegue ao trecho de encerramento.

É idempotente: chamadas repetidas não repetem o efeito.

### `status()`

Transforma estado interno + thread viva + eventos em payload JSON para interface.

---

# 15. `magalu_bot/bot.py`

É o coração operacional do coletor.

O arquivo é grande porque orquestra todo o fluxo.

## 15.1 Chrome remoto

### `chrome_9222_esta_aberto()`

Consulta:

```text
http://127.0.0.1:9222/json/version
```

### `iniciar_chrome_se_necessario()`

Se a porta já estiver respondendo, reutiliza o Chrome existente.

Caso contrário, abre:

```text
chrome.exe
--remote-debugging-port=9222
--user-data-dir=C:\ChromeDebug
```

Isso é essencial porque o perfil persistente conserva sessão/login.

### `fechar_chrome_se_necessario()`

Só fecha o Chrome se o próprio bot tiver aberto o processo.

Se o Chrome já existia antes, o bot não o mata.

## 15.2 Login

### `pagina_eh_login()`

Usa URL e sinais no conteúdo da página.

### `verificar_login()`

Abre a URL de autenticação e detecta se existe sessão.

### `aguardar_login()`

Se não houver sessão:

- muda estado para `AGUARDANDO_LOGIN`;
- permite login manual;
- verifica periodicamente;
- respeita timeout.

## 15.3 Descoberta da vitrine

O bot procura a loja Magazine Você vinculada ao usuário e normaliza URL.

Regras evitam interpretar caminhos reservados como `/admin` ou `/login` como slug de vitrine.

## 15.4 Controle de pausa/parada

`verificar_controle_bot()` é chamado em pontos do loop para que o processo reaja a comandos sem corromper dados.

Parada não é um `kill` abrupto. O objetivo é chegar a um ponto seguro, salvar e encerrar.

## 15.5 `executar_bot()`

Fluxo conceitual:

```text
iniciar
  ↓
abrir/conectar Chrome
  ↓
verificar login
  ↓
aguardar login se necessário
  ↓
descobrir vitrine
  ↓
carregar resultados antigos Excel/CSV
  ↓
montar alvos categoria/keyword
  ↓
consultar ProgressoColeta
  ↓
abrir página do alvo
  ↓
coletar URLs
  ↓
ordenar diversidade heurística
  ↓
para cada produto
    ↓
verificar pausa/parada
    ↓
ignorar processados conforme regras
    ↓
gerar link afiliado
    ↓
capturar nome/preços/imagem
    ↓
salvar DB
    ↓
atualizar progresso
    ↓
criar Oferta
    ↓
publicar se auto-publicação estiver ativa
    ↓
salvar Excel/CSV
  ↓
próxima página / próximo alvo / novo ciclo
  ↓
salvamento final
```

## 15.6 Loop

Sem nenhum alvo marcado para loop, a execução termina após a passagem configurada.

Com alvo em loop, o bot inicia novas rodadas até parada explícita ou erro fatal.

O limite por categoria deve ser interpretado como lote/ciclo, não como obrigação de encerrar definitivamente uma execução contínua.

## 15.7 Publicação acoplada ao resultado válido

A publicação só é tentada depois que existe `link_afiliado`.

Ela ocorre depois da persistência do resultado, protegendo o dado coletado contra falhas do canal.

---

# 16. `magalu_bot/automacao/categorias.py`

Responsável pela descoberta de URLs de produtos.

## `rolar_pagina()`

Executa scroll progressivo com delays aleatórios para carregar conteúdo.

## `coletar_produtos_categoria()`

Abre:

```text
<base_url>/busca/<categoria>/
```

Com paginação:

```text
?page=N
```

Procura anchors cujo `href` contém `/p/`, remove query string e deduplica.

## `coletar_produtos_keyword()`

Mesma ideia para palavra-chave.

## `familia_produto()`

Tenta extrair uma assinatura semântica simples a partir do nome/slug.

## `ordenar_produtos_diversos()`

Reordena candidatos penalizando famílias recentes.

### Limitação conhecida

Esse método atual usa heurística lexical simples. Ele ainda não impediu repetição real de famílias como armários em todos os cenários. Trate essa função como base para evolução, não como solução final.

---

# 17. `magalu_bot/automacao/afiliados.py`

Responsável pela parte mais delicada da automação: transformar uma página de produto em dados + link de afiliado.

Funções principais:

### `normalizar_preco()`

Converte preço textual para número consistente.

### `validar_link_afiliado()`

Aceita somente padrões/domínios previstos como links afiliados e rejeita URLs normais de produto/vitrine.

### `extrair_links_validos_de_texto()`

Procura links permitidos dentro de textos/copys/elementos.

### `_coletar_candidatos_dom()`

Busca possíveis campos/elementos no DOM comum.

### `_coletar_candidatos_shadow_dom()`

Fallback para componentes encapsulados em Shadow DOM.

### `_encontrar_botao_gerar()`

Localiza o botão “Gerar link”.

### `_capturar_link_apos_clique()`

Compara estado anterior e posterior ao clique para identificar o link novo.

### `_capturar_precos()`

Busca preço anterior e atual por seletores conhecidos.

### `_capturar_nome()`

Obtém nome do produto.

### `_capturar_imagem()`

Obtém URL de imagem.

### `_tentativa_gerar_link()`

Uma tentativa isolada do fluxo de geração.

### `gerar_link_afiliado()`

Orquestra retries e devolve dados finais.

Princípio: falha em um produto deve ser registrada como `REVISAR`/erro e a execução deve seguir para o próximo, em vez de derrubar o loop inteiro quando possível.

---

# 18. `magalu_bot/persistencia/django_db.py`

É a fronteira entre o bot e o ORM.

Isso é importante porque evita espalhar `Produto.objects...` por todo `bot.py`.

## `registrar_log()`

Cria `LogExecucao`, usando retry curto. Falha de log não pode derrubar o bot.

## `atualizar_execucao()`

Atualiza somente campos permitidos.

## `salvar_produto_resultado()`

Dentro de `transaction.atomic()`:

1. atualiza/cria Produto;
2. atualiza/cria Afiliado;
3. registra Histórico de Preço;
4. atualiza contagem da Execução.

A unidade inteira é repetida se ocorrer lock transitório do SQLite.

## `finalizar_execucao()`

Marca estado, erro e fim.

## `obter_progresso_coleta()`

Retoma o cursor apenas se o progresso tiver sido atualizado nas últimas 12 horas por padrão.

Progresso antigo reinicia na página 1.

## `salvar_progresso_coleta()`

Atualiza próxima página e mantém no máximo as últimas 8 famílias recentes.

---

# 19. `magalu_bot/persistencia/excel.py`

Mantém compatibilidade com o histórico original do bot local.

Responsabilidades:

- carregar resultados antigos;
- escrever Excel;
- escrever CSV;
- salvar atomicamente;
- detectar produto previamente processado.

O banco Django é a persistência principal da aplicação web. Excel/CSV deve ser tratado como relatório/compatibilidade, não como fonte de verdade multiusuário principal.

---

# 20. `magalu_bot/automacao/logger.py`

Logger de arquivo/console do coletor legado.

Fornece helpers:

```text
log_info
log_ok
log_revisar
log_erro
```

Não confundir com `LogExecucao` do banco. Existem duas camadas históricas de logging.

---

# 21. `magalu_bot/relatorios/relatorios.py`

Formata relatório textual final ou parcial.

Usado quando:

- execução termina normalmente;
- usuário interrompe;
- Ctrl+C ocorre no modo compatível/local.

---

# 22. Management commands

## `configurar_canais`

```bash
python manage.py configurar_canais
```

Lê credenciais do ambiente e cria/atualiza configurações com `owner=None`.

É utilitário legado/fallback. O fluxo principal multiusuário usa `/configuracoes/`.

## `migrar_credenciais`

```bash
python manage.py migrar_credenciais
```

Converte tokens legados para a criptografia atual.

## `publicar_ofertas`

Exemplo:

```bash
python manage.py publicar_ofertas --owner 1 --limite 20 --canal TELEGRAM
```

Opções:

- `--owner`;
- `--limite`;
- `--turno`;
- `--campanha`;
- `--canal` repetível;
- `--forcar` para ignorar cache.

---

# 23. Scripts locais

## `scripts/configurar_local.py`

Prepara `.env`.

Comportamento importante:

- se `.env` existe, lê ele;
- senão, usa `.env.example`;
- gera `SECRET_KEY` se vazia;
- gera `CREDENTIAL_ENCRYPTION_KEY` se vazia ou `change-me`;
- preserva segredo válido existente;
- garante DEBUG, hosts, timezone e banco local mínimos.

Esse script pode ser executado em toda inicialização justamente porque não deve rotacionar chaves já existentes.

## `scripts/limpar_local.py`

Limpeza conservadora.

Remove:

- `__pycache__`;
- `.pyc`.

Nunca apaga `db.sqlite3`.

Se o banco existe, tenta:

```sql
PRAGMA wal_checkpoint(TRUNCATE)
```

Sidecars só são removidos quando o banco principal não existe.

---

# 24. Inicializador Windows recomendado

O `iniciar.bat` discutido no fluxo atual é um inicializador operacional, não uma migration nem parte essencial do framework.

A intenção é usá-lo **sempre que quiser ligar o projeto**, não somente uma vez.

Fluxo desejado:

```text
verificar projeto
   ↓
localizar Python
   ↓
criar/reusar venv
   ↓
instalar requirements apenas na primeira vez ou modo atualizar
   ↓
configurar .env
   ↓
limpeza segura
   ↓
migrate
   ↓
criar superuser se ainda não existir
   ↓
manage.py check
   ↓
abrir navegador
   ↓
runserver
```

Modo normal:

```bat
iniciar.bat
```

Após alteração de `requirements.txt`:

```bat
iniciar.bat atualizar
```

### Por que não rodar `makemigrations` automaticamente?

Porque migrations são artefatos de código que devem ser revisados e commitados.

O inicializador deve executar:

```bat
python manage.py migrate
```

mas não deve inventar migrations em máquinas diferentes.

---

# 25. Configuração manual do Chrome para o bot

O Chrome precisa estar acessível por debugging remoto.

O próprio código tenta iniciar:

```bat
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir=C:\ChromeDebug
```

Para verificar manualmente:

```text
http://127.0.0.1:9222/json/version
```

Se houver JSON com informações do Chrome, a porta está operacional.

O diretório `C:\ChromeDebug` mantém a sessão do marketplace, reduzindo necessidade de login em toda execução.

### Cuidados

Não abra duas instâncias incompatíveis usando o mesmo perfil ao mesmo tempo.

O usuário ainda pode precisar realizar login manual no marketplace quando a sessão expirar.

---

# 26. Fluxo completo de uma coleta

```text
1. Usuário autentica no Django
2. Abre /coleta/ ou /magalu/
3. Escolhe categorias/keywords
4. Escolhe quais alvos entram em loop
5. Django valida templates de keywords
6. View chama ControladorBot.iniciar
7. É criado um registro Execucao
8. Thread é iniciada
9. Bot conecta ao Chrome :9222
10. Verifica sessão no Magalu
11. Se necessário, aguarda login manual
12. Descobre vitrine Magazine Você
13. Consulta ProgressoColeta por alvo
14. Busca página do alvo
15. Extrai URLs /p/
16. Aplica deduplicação/reordenação atual
17. Abre produto
18. Clica em Gerar link
19. Valida link afiliado
20. Captura nome/preços/imagem
21. Persiste Produto/Afiliado/Histórico
22. Atualiza Execucao
23. Atualiza ProgressoColeta
24. Cria Oferta
25. Se auto-publicação ativa, tenta os canais
26. Registra Mensagem ENVIADO/FALHOU/CACHE
27. Atualiza alertas
28. Exporta Excel/CSV
29. Continua lote/página/alvo/ciclo
30. Para somente ao fim natural, stop ou erro fatal
```

---

# 27. Coleta Geral versus Central Magalu

## Coleta Geral

Objetivo: UX simples e preparada para vários marketplaces.

Hoje somente Magalu está habilitado.

Categorias personalizadas são tratadas como conceitos reaproveitáveis para futuros marketplaces.

## Central Magalu

Objetivo: controles detalhados específicos do coletor Magalu.

Inclui:

- múltiplos alvos;
- loops;
- progresso;
- logs;
- pausa;
- retomada;
- parada;
- criação contextual de categoria/template.

As duas páginas chamam o mesmo backend de controle do bot.

---

# 28. Templates e categorias personalizadas

Uma keyword personalizada é armazenada como:

```text
TemplateOferta(tipo="KEYWORD")
```

Antes de processá-la, o backend exige uma das duas condições:

```text
possui ao menos uma chamada
OU
divulgar_sem_template = True
```

Isso impede que uma keyword nova entre em publicação com um estado editorial ambíguo.

Categorias nativas são criadas automaticamente por usuário.

---

# 29. Sistema de alertas de publicação

Princípio central:

> **Canal quebrado não deve parar scraping.**

Ao iniciar sem canal totalmente configurado, a API retorna um aviso não bloqueante.

Se o Telegram estiver inválido:

```text
coleta continua
produto continua salvo
oferta continua criada
mensagem de publicação falha
alerta visual aparece
```

Quando a configuração local é corrigida, o alerta de configuração é resolvido imediatamente.

Quando se trata de erro remoto, um envio posterior bem-sucedido resolve o alerta.

---

# 30. Cache e republicação inteligente

O sistema não deve publicar o mesmo produto sem parar durante loops AFK.

Cada `Mensagem` salva snapshot do produto.

A política consulta:

- última publicação no mesmo canal;
- idade dessa publicação;
- preço publicado;
- preço atual;
- desconto publicado;
- desconto atual;
- score de Produto Quente.

Estados retornados pelo fluxo podem incluir:

```text
ENVIADO
CACHE
FALHOU: ...
```

Isso é diferente de “produto processado recentemente” na coleta. Existem duas preocupações:

1. evitar trabalho repetitivo do coletor;
2. evitar spam no canal.

Não misturar as duas em uma única flag.

---

# 31. Produtos Quentes

O ranking é recalculado automaticamente aproximadamente a cada 5 dias.

Pode ser forçado pela interface usando query parameter apropriado.

Score combina principalmente:

```text
desconto * peso
+ frescor
+ bônus por queda de preço
+ disponibilidade de preço
```

O ranking influencia a possibilidade de republicação antecipada.

---

# 32. Timezone

Cada usuário possui `PerfilUsuario.timezone`.

O cadastro tenta armazenar timezone e a configuração permite alteração.

`marketing.detectar_turno()` usa `ZoneInfo` para converter a hora atual para o fuso do usuário.

Turnos atuais:

```text
MANHA
ALMOCO
TARDE
NOITE
RELAMPAGO
```

Assim, “Achadinho da manhã” não deve depender cegamente do timezone do servidor.

---

# 33. Banco local e concorrência

O erro histórico era:

```text
OperationalError: database is locked
```

Motivo estrutural:

```text
request Django
     +
thread Selenium
     +
SQLite com um escritor por vez
```

O Commit 5 atua em quatro níveis:

### Django

`OPTIONS.timeout = 30`

### SQLite

```sql
busy_timeout=30000
journal_mode=WAL
synchronous=NORMAL
```

### ORM

Operações importantes usam `executar_com_retry_db()`.

### Threads

Conexões antigas são descartadas entre retries quando necessário.

## Ainda pode haver lock?

Sim. Exemplos:

- duas instâncias inteiras do projeto abertas;
- ferramenta externa mantendo transação no SQLite;
- processo morto segurando recurso por algum tempo;
- operação longa inadequada dentro de transaction.

Para produção real use PostgreSQL.

---

# 34. Testes existentes

## `core/tests.py`

Atualmente cobre exemplos como:

- criação de oferta/desconto;
- defaults de Execução;
- cadastro/login/proteção de rotas;
- escape da mensagem Telegram;
- compatibilidade de template;
- navegação `/ofertas/` para a âncora correta.

## `magalu_bot/tests/test_afiliados.py`

Testa:

- domínios de link afiliado permitidos;
- rejeição de produto/vitrine comum;
- extração de URL permitida de texto;
- rejeição de URL não permitida.

Executar:

```bash
python manage.py test
```

ou módulos específicos:

```bash
python manage.py test core
python manage.py test magalu_bot.tests
```

### Lacuna importante

A lógica Selenium real depende de DOM externo e não está plenamente coberta por testes de integração estáveis.

Ao mexer em seletores Magalu, testar manualmente em sessão real continua necessário.

---

# 35. Deploy atual no Render

`render.yaml` define um serviço web Python.

Build:

```bash
./build.sh
```

`build.sh` executa:

```bash
pip install -r requirements.txt
python manage.py collectstatic --noinput
python manage.py migrate --noinput
```

Start:

```bash
gunicorn aflmain.wsgi:application --bind 0.0.0.0:$PORT --workers 2 --timeout 120
```

Variáveis definidas/configuradas no Render incluem:

- Python 3.12.8;
- SECRET_KEY;
- DEBUG=False;
- ALLOWED_HOSTS;
- CSRF_TRUSTED_ORIGINS;
- DATABASE_URL;
- TIME_ZONE.

## Limite importante do deploy atual

Esse `render.yaml` sobe o **web app**, não implementa um worker Selenium completo com Chrome persistente equivalente ao Windows local.

Um deploy web bem-sucedido não significa automaticamente que o bot Magalu desktop está operacional em produção.

---

# 36. Reconstrução do banco

Em um projeto novo, depois de criar todos os models/migrations:

```bash
python manage.py migrate
```

Criar admin:

```bash
python manage.py createsuperuser
```

Checar projeto:

```bash
python manage.py check
```

Subir:

```bash
python manage.py runserver 127.0.0.1:8000
```

URLs úteis:

```text
http://127.0.0.1:8000/
http://127.0.0.1:8000/admin/
http://127.0.0.1:8000/home/
http://127.0.0.1:8000/coleta/
http://127.0.0.1:8000/magalu/
http://127.0.0.1:8000/configuracoes/
```

---

# 37. Ordem recomendada para reconstruir os arquivos

Se um desenvolvedor tiver apenas este documento e quiser reimplementar o sistema, a ordem mais segura é:

1. criar repositório e virtualenv;
2. criar `requirements.txt`;
3. criar Django `aflmain`;
4. configurar `.env` e `settings.py`;
5. criar app `core`;
6. implementar `fields.py` antes do model de canais;
7. implementar todos os models;
8. criar migrations;
9. implementar `db.py` e carregá-lo via `CoreConfig.ready()`;
10. implementar autenticação/views/URLs básicos;
11. criar templates públicos e sidebar;
12. implementar TemplateOferta/template service;
13. implementar marketing;
14. implementar publicação e alertas;
15. implementar configuração de canais;
16. implementar Produtos Quentes;
17. criar pacote `magalu_bot`;
18. criar estados;
19. criar config;
20. criar scraping de categorias;
21. criar extração de afiliados;
22. criar persistência ORM;
23. criar persistência Excel/CSV;
24. montar `bot.py`;
25. montar `controlador.py`;
26. ligar as views ao controlador;
27. adicionar polling/status/logs no frontend;
28. adicionar retomada `ProgressoColeta`;
29. adicionar cache de publicação;
30. adicionar scripts locais;
31. adicionar testes;
32. adicionar build/deploy.

Essa ordem reduz dependências circulares durante a reconstrução.

---

# 38. Fluxo de criação de um produto no banco

```text
Selenium encontra URL
      ↓
gera_link_afiliado()
      ↓
dados = {
  nome,
  link_afiliado,
  preco_anterior,
  preco_atual,
  imagem_url,
  status,
  marketplace
}
      ↓
salvar_produto_resultado()
      ↓
transaction.atomic
      ├── Produto update_or_create
      ├── Afiliado update_or_create
      ├── HistoricoPreco create
      └── Execucao update
```

Isso explica por que a URL original do produto é uma identidade operacional importante.

---

# 39. Fluxo de criação/publicação de oferta

```text
Produto com Afiliado.link_afiliado
        ↓
criar_oferta()
        ↓
marketing.gerar_mensagem()
        ├── template do usuário
        ├── compatibilidade contextual
        ├── keyword segura
        ├── categoria
        ├── turno/timezone
        ├── preço
        ├── desconto
        └── CTA
        ↓
Oferta PRONTA
        ↓
publicar_oferta()
        ↓
avaliar cache/republicação
        ↓
canal
   ┌────┴─────┐
Telegram   WhatsApp
   │           │
   ▼           ▼
Mensagem   Mensagem
ENVIADO/   ENVIADO/
FALHOU     FALHOU
   │           │
   └─────┬─────┘
         ▼
  alerta resolvido
  ou alerta criado
```

---

# 40. Multiusuário: regras que não devem ser quebradas

Ao criar qualquer nova query operacional, pergunte:

> “Essa query deveria filtrar por `owner=request.user` ou `owner_id`?”

Entidades com escopo de usuário incluem:

- Produto;
- Execucao;
- Mensagem;
- Oferta;
- ConfiguracaoCanal;
- ConfiguracaoAutomacao;
- TemplateOferta;
- AlertaSistema;
- ProdutoQuente;
- PerfilUsuario;
- ProgressoColeta.

A relação `Afiliado` herda o escopo via Produto.

A relação `HistoricoPreco` herda o escopo via Produto.

`LogExecucao` herda via Execucao.

Nunca buscar uma oferta apenas por ID em endpoint autenticado sem também validar `owner`.

---

# 41. Invariantes de negócio

Um agente de IA ou desenvolvedor deve preservar estes princípios:

## Invariante 1 — coleta e divulgação são independentes

Falhar Telegram não pode invalidar scraping.

## Invariante 2 — credenciais são por usuário

Não reutilizar token global de outro usuário.

## Invariante 3 — segredos não aparecem na UI/log

Erro de API deve ser traduzido/sanitizado para alerta visual.

## Invariante 4 — token armazenado deve permanecer descriptografável

Não rotacionar `CREDENTIAL_ENCRYPTION_KEY` automaticamente.

## Invariante 5 — stop é seguro

Não matar thread arbitrariamente se existe caminho para finalizar e persistir.

## Invariante 6 — item ruim não deve parar lote

Produto problemático deve virar REVISAR/erro e permitir continuidade quando possível.

## Invariante 7 — migrations são versionadas

Inicializador roda `migrate`, não `makemigrations` automático.

## Invariante 8 — dados de um usuário não vazam para outro

Toda nova feature deve revisar escopo `owner`.

## Invariante 9 — Produto só é publicável com link afiliado válido

Nunca enviar CTA apontando para link bruto do produto como substituto silencioso.

## Invariante 10 — cache não significa banimento permanente

Produto pode voltar a ser elegível quando cache expira ou há mudança relevante.

---

# 42. Contexto específico para agente de IA

Ao entregar este repositório para um agente, forneça estas instruções junto com o código:

```text
Você está trabalhando no Skallora/Afillimetrics, um Django multiusuário para
coleta e publicação de ofertas de afiliados.

Antes de alterar qualquer fluxo:
1. leia core/models.py;
2. leia core/views.py;
3. leia core/publicacao.py;
4. leia core/marketing.py;
5. leia magalu_bot/controlador.py;
6. leia magalu_bot/bot.py;
7. preserve owner/user isolation;
8. preserve CREDENTIAL_ENCRYPTION_KEY separada de SECRET_KEY;
9. não bloqueie coleta quando publicação falhar;
10. não delete db.sqlite3 ou .env;
11. não gere migrations automaticamente no startup;
12. não substitua parada segura por kill de thread;
13. não suponha que diversidade de produtos está resolvida;
14. lembre que só há um controlador Selenium global atualmente;
15. valide alterações com compile/test/manage.py check quando possível.
```

Antes de editar, o agente deve procurar o caminho real no ZIP atual, porque a estrutura histórica do projeto já mudou várias vezes.

---

# 43. Checklist para qualquer novo commit

## Código

- mudança está isolada ao problema do commit?
- preserva recursos anteriores?
- `owner` está sendo respeitado?
- logs não expõem token?
- erros externos são tratados?
- migrations necessárias foram criadas?

## Validação mínima

```bash
python -m compileall .
python manage.py check
python manage.py test
```

Quando o projeto estiver compactado:

```bash
unzip -t arquivo.zip
```

Para Selenium:

- teste manual de login;
- descoberta da loja;
- geração de link;
- preços;
- stop;
- loop;
- publicação configurada e não configurada.

---

# 44. Troubleshooting

## `database is locked`

1. confirme que não há dois `runserver`;
2. confirme que outro BAT não está aberto;
3. encerre processos Python antigos;
4. não delete `db.sqlite3`;
5. rode novamente;
6. se persistir, investigar a operação/transação específica.

O projeto já possui retry/WAL, então lock persistente pode indicar concorrência externa ou operação longa.

## Chrome não responde em 9222

Teste:

```text
http://127.0.0.1:9222/json/version
```

Se não abrir:

- confira caminho do Chrome;
- confira porta;
- confira processo;
- confira perfil `C:\ChromeDebug`.

## Login não detectado

- veja a janela Chrome;
- faça login manual;
- confirme se URL/DOM da plataforma mudou;
- revise `pagina_eh_login()`.

## Botão “Gerar link” não encontrado

Marketplace provavelmente alterou DOM.

Revisar principalmente:

```text
magalu_bot/automacao/afiliados.py
```

Não altere seletores às cegas: inspecione a página real.

## Telegram 400

O código atual tenta extrair a descrição real retornada pela API.

Causas possíveis:

- token errado;
- chat_id errado;
- bot sem acesso;
- HTML inválido;
- mídia inválida;
- endpoint/permite operação incompatível.

O gerador atual escapa nome/link e tem fallback de imagem para texto.

## Telegram 401/403

Tratar como problema de credencial/permissão, não como falha do scraping.

## WhatsApp API falha

Conferir:

- token;
- endpoint;
- destino;
- formato aceito pelo provedor atual.

A implementação presente usa um payload no estilo Cloud API, mas o endpoint é configurável.

## Produto não publica novamente

Verificar:

- `Mensagem` anterior;
- `CACHE_RETENCAO_HOURS` / configuração do usuário;
- cooldown mínimo;
- preço/desconto atuais;
- ranking Produto Quente.

## Keyword não inicia

Se backend retorna 422 e `keywords_sem_template`, configure:

- ao menos uma chamada; ou
- “divulgar sem template”.

---

# 45. Arquivos que não devem entrar no Git

No mínimo:

```text
.env
db.sqlite3
*.sqlite3-wal
*.sqlite3-shm
*.sqlite3-journal
venv/
.venv/
__pycache__/
*.pyc
staticfiles/
logs/
```

Se o BAT for local por máquina:

```text
iniciar.bat
iniciar_local.bat
```

Excel/CSV gerados pelo bot também devem ser avaliados como artefatos locais, principalmente se contiverem dados operacionais.

---

# 46. Backup correto

Para backup local completo, preservar:

```text
código versionado
.env
db.sqlite3
arquivos Excel/CSV se forem importantes como relatório
```

Para PostgreSQL:

- dump do banco;
- `.env`/secrets em cofre seguro;
- versão/tag exata do código.

Nunca enviar `.env` para GitHub público.

---

# 47. Processo de atualização local

Com `iniciar.bat`:

```text
baixar/extrair nova versão
        ↓
preservar .env e banco quando for upgrade da mesma instalação
        ↓
iniciar.bat atualizar   ← se requirements mudou
        ↓
migrate automático
        ↓
check
        ↓
runserver
```

Se a atualização inclui migrations mas não requirements:

```text
iniciar.bat
```

já é suficiente, pois `migrate` deve ser executado em toda inicialização.

---

# 48. Histórico funcional relevante até este estado

Este documento não substitui Git, mas ajuda a entender por que certas partes existem.

## Credenciais separadas

A criptografia de canais foi separada de `SECRET_KEY` usando `CREDENTIAL_ENCRYPTION_KEY`.

## Alertas de divulgação

Foram adicionados alertas contextuais que não interrompem coleta.

## Stop seguro

A interface passou a ter estado de parada solicitada, evitando comandos repetidos.

## Templates contextuais

Categorias e keywords passaram a ter templates próprios.

## Loop/ciclos

Categorias e keywords ganharam suporte a execução contínua.

## Republicação

Foi criado cache de envio com antecipação por mudança relevante de preço/desconto.

## Produtos Quentes

Ranking por score e recência foi integrado.

## Perfil/timezone

Mensagens por turno passaram a considerar o fuso do usuário.

## Retomada de coleta

`ProgressoColeta` foi adicionado para retomar páginas recentes.

## Compatibilidade de templates

Contexto incompatível, como panela/banheiro, passa a usar fallback seguro.

## Navegação de ofertas

`/ofertas/` direciona para `#ofertas-geradas` dentro da central útil.

## Concorrência SQLite

WAL, timeout e retry foram implementados.

## Diversidade

Existe uma heurística no código, mas o comportamento ainda foi considerado insuficiente em uso real e deve permanecer no backlog.

---

# 49. O que ainda merece refatoração

Esta seção descreve oportunidades, não funcionalidades já implementadas.

## Separar `views.py`

`core/views.py` já concentra muitas responsabilidades. Uma divisão possível:

```text
views/auth.py
views/dashboard.py
views/coleta.py
views/ofertas.py
views/configuracoes.py
```

## Separar JS de `magalu_bot.html`

O template é muito grande e concentra apresentação + lógica de frontend.

## Worker Selenium

Mover execução para processo/worker próprio.

## Fila por usuário

O singleton global deve evoluir para execução gerenciada, se paralelismo real for requisito.

## PostgreSQL como padrão de produção

Evitar depender de tuning SQLite fora do desenvolvimento.

## Diversidade semântica real

Substituir a heurística “primeiros tokens do slug” por classificação de família mais robusta e seleção por buffer global antes de publicar.

## Corrigir `__init___.py`

Normalizar package.

## Consolidar identidade Skallora

Renomear interface/docs/serviço somente em mudança planejada.

## Remover dependências não utilizadas

Revisar `webdriver-manager` e `beautifulsoup4` antes de remover. Não eliminar apenas porque não aparecem em um arquivo isolado.

---

# 50. Modelo mental para entender o sistema rapidamente

Um desenvolvedor novo pode pensar em cinco camadas:

```text
CAMADA 1 — WEB
core/views.py + templates + urls

CAMADA 2 — DOMÍNIO
core/models.py + template_service + marketing + alerts

CAMADA 3 — PUBLICAÇÃO
core/publicacao.py + services.py

CAMADA 4 — AUTOMAÇÃO
magalu_bot/bot.py + controlador + automacao/*

CAMADA 5 — PERSISTÊNCIA/INFRA
Django ORM + django_db.py + excel.py + db.py + .env + Render
```

Quando um bug aparece, identifique primeiro em qual camada ele nasce.

Exemplo:

```text
“armário atrás de armário”
```

não é primariamente um problema do Telegram. É seleção/ordenação no coletor.

```text
“panela com chamada de banheiro”
```

é contexto de marketing/template.

```text
“produto coletado mas não enviado”
```

é publicação/configuração/cache, não necessariamente scraping.

```text
“database is locked”
```

é infraestrutura/concorrência.

---

# 51. Glossário

**Categoria nativa** — categoria padrão como cozinha/quarto/sala/banheiro/acessórios.

**Categoria personalizada** — keyword salva pelo usuário e tratada na UI como alvo reutilizável.

**Alvo** — categoria ou keyword que o coletor deve pesquisar.

**Loop** — alvo que volta a ser processado em novos ciclos até stop.

**Produto** — URL/item coletado do marketplace.

**Afiliado** — extensão do Produto que guarda link de afiliado e status.

**Oferta** — mensagem editorial gerada a partir de um produto.

**Mensagem** — registro de envio/tentativa por canal.

**Template** — uma ou mais chamadas associadas a categoria/keyword.

**Produto Quente** — produto ranqueado com score de oportunidade de republicação.

**Termômetro de Oportunidade** — indicador da Home baseado no contexto temporal, diferente de Produto Quente.

**ProgressoColeta** — checkpoint por alvo/página.

**Cache de publicação** — política que evita spam de uma mesma oferta.

**Remote debugging** — forma utilizada para Selenium se conectar à sessão persistente do Chrome.

---

# 52. Checklist de primeira instalação para um dev externo

```text
[ ] Instalar Python 3.12
[ ] Instalar Google Chrome
[ ] Clonar/extrair o repositório
[ ] Entrar na pasta aflmain
[ ] Criar venv
[ ] Ativar venv
[ ] pip install -r requirements.txt
[ ] Criar .env a partir do .env.example
[ ] Gerar SECRET_KEY
[ ] Gerar CREDENTIAL_ENCRYPTION_KEY
[ ] Manter DEBUG=True localmente
[ ] Configurar DATABASE_URL=sqlite:///db.sqlite3
[ ] python manage.py migrate
[ ] python manage.py createsuperuser
[ ] python manage.py check
[ ] python manage.py test
[ ] python manage.py runserver
[ ] Criar/entrar em usuário normal
[ ] Configurar timezone
[ ] Configurar Telegram/WhatsApp se desejar publicar
[ ] Abrir Central Magalu
[ ] Garantir Chrome :9222
[ ] Fazer login manual no marketplace quando solicitado
[ ] Testar uma categoria sem loop
[ ] Confirmar Produto/Afiliado/Oferta no banco/UI
[ ] Só depois testar loop/AFK
```

---

# 53. Checklist de produção web

```text
[ ] DEBUG=False
[ ] SECRET_KEY forte e persistente
[ ] CREDENTIAL_ENCRYPTION_KEY forte e persistente
[ ] PostgreSQL
[ ] ALLOWED_HOSTS correto
[ ] CSRF_TRUSTED_ORIGINS correto
[ ] HTTPS
[ ] migrate
[ ] collectstatic
[ ] Gunicorn
[ ] backup do banco
[ ] backup seguro das chaves
[ ] monitoramento de logs
[ ] worker Selenium separado antes de prometer coleta Magalu em ambiente cloud
```

---

# 54. Regra de ouro ao modificar o projeto

Antes de alterar qualquer coisa, responda três perguntas:

1. **de quem é esse dado?** → `owner`;
2. **o erro deveria parar a coleta?** → quase sempre publicação externa não;
3. **a mudança pertence ao web app, ao motor de marketing ou ao coletor?** → altere a camada correta.

Isso evita grande parte dos bugs arquiteturais que aparecem quando todas as responsabilidades são tratadas como se fossem “o bot”.

---

# 55. Resumo final da reconstrução

Para reproduzir o estado atual do projeto do zero, você precisa reconstruir quatro pilares em ordem:

```text
1. Django + domínio multiusuário
   ↓
2. automação Selenium do Magalu
   ↓
3. geração/publicação inteligente
   ↓
4. resiliência operacional
```

O primeiro pilar cria usuários, Produto, Afiliado, Execucao, Oferta, Mensagem, Templates, canais e alertas.

O segundo cria Chrome remoto, login, descoberta da vitrine, busca de produtos, botão Gerar link e persistência.

O terceiro cria templates, marketing, compatibilidade contextual, Telegram/WhatsApp, cache e Produtos Quentes.

O quarto cria stop seguro, retomada, logs, timezone, criptografia separada, WAL/retry SQLite e scripts de inicialização.

Com esses blocos e os invariantes documentados acima, um desenvolvedor externo ou agente de IA consegue entender não apenas **quais arquivos existem**, mas **por que eles existem, quem chama quem, quais dados atravessam cada camada e quais decisões não devem ser quebradas**.

---

## Anexo A — comandos essenciais

```bash
# preparar ambiente
python -m venv venv

# Windows
venv\Scripts\activate

# dependências
pip install -r requirements.txt

# preparar ambiente local
python scripts/configurar_local.py
python scripts/limpar_local.py

# banco
python manage.py migrate

# admin
python manage.py createsuperuser

# validação
python manage.py check
python manage.py test

# executar
python manage.py runserver 127.0.0.1:8000

# migrar tokens legados
python manage.py migrar_credenciais

# publicar ofertas por comando
python manage.py publicar_ofertas --owner 1 --limite 20 --canal TELEGRAM

# coletar estáticos para deploy
python manage.py collectstatic --noinput
```

---

## Anexo B — fontes de verdade por assunto

| Assunto | Arquivo principal |
|---|---|
| Configuração Django | `aflmain/settings.py` |
| Rotas raiz | `aflmain/urls.py` |
| Domínio/banco | `core/models.py` |
| Criptografia | `core/fields.py` |
| Concorrência SQLite | `core/db.py` |
| HTTP/views | `core/views.py` |
| Rotas app | `core/urls.py` |
| Templates de oferta | `core/template_service.py` |
| Copy inteligente | `core/marketing.py` |
| Envio Telegram/WhatsApp | `core/publicacao.py` |
| Alertas | `core/alerts.py` |
| Termômetro | `core/opportunity.py` |
| Controle da thread | `magalu_bot/controlador.py` |
| Fluxo Selenium | `magalu_bot/bot.py` |
| Busca de produtos | `magalu_bot/automacao/categorias.py` |
| Link afiliado/preços | `magalu_bot/automacao/afiliados.py` |
| Persistência Django do bot | `magalu_bot/persistencia/django_db.py` |
| Excel/CSV | `magalu_bot/persistencia/excel.py` |
| Estados | `magalu_bot/estados.py` |
| Constantes Magalu | `magalu_bot/config/config.py` |
| Preparação do `.env` | `scripts/configurar_local.py` |
| Limpeza local | `scripts/limpar_local.py` |
| Deploy | `render.yaml`, `build.sh` |
| Dependências | `requirements.txt` |

---

## Anexo C — observações de auditoria da base inspecionada

Estas observações são particularmente úteis para agentes automatizados porque evitam falsas suposições:

1. o app Django central atual é `core`;
2. `magalu_bot` é módulo Python, não app em `INSTALLED_APPS`;
3. o projeto atual ainda contém nomenclatura Afillimetrics internamente;
4. `ui-loader.css` e `ui-loader.js` estão vazios na base inspecionada;
5. `magalu_bot/__init___.py` tem nome não convencional;
6. o bot Selenium é global/singleton por processo;
7. dados são majoritariamente isolados por `owner`;
8. Telegram/WhatsApp falhando não devem derrubar coleta;
9. a chave atual de tokens é `CREDENTIAL_ENCRYPTION_KEY`;
10. a lógica de diversidade existe, porém ainda não entrega o resultado desejado em uso real;
11. o Commit 5 é voltado a concorrência SQLite e não resolve a arquitetura de múltiplos workers;
12. Render atual sobe o web app, não um ambiente desktop equivalente ao bot Selenium local.

---

**Fim da Bíblia Técnica — estado atual do Skallora/Afillimetrics.**
