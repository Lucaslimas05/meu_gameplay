# 🎮 Meu Gameplay

![Oracle APEX](https://img.shields.io/badge/Oracle%20APEX-26.2.1-C74634?logo=oracle&logoColor=white)
![PL/SQL](https://img.shields.io/badge/PL%2FSQL-Oracle%20Database-F80000?logo=oracle&logoColor=white)
![Status](https://img.shields.io/badge/status-projeto%20de%20estudo-blue)

Aplicação **Oracle APEX** para registrar os jogos que você joga: quantas horas jogou em cada um e se já zerou.

![Ícone do app](shared-components/static-files/icons/app-icon-192.png)

> 📚 **Projeto de estudo**
>
> Fiz este projeto para **aprender e praticar Oracle APEX**. Ele não foi pensado para uso em produção.
> O objetivo é explorar na prática os recursos principais da plataforma: páginas de relatório e formulário, diálogos modais, gráficos, listas de valores, autenticação customizada em PL/SQL, itens de aplicação e modelagem de dados no Oracle Database.
>
> Algumas decisões são simples de propósito, e a seção **Próximos passos**, no fim deste documento, lista o que ainda dá para evoluir.

---

## ✨ Funcionalidades

- **Login por e-mail e senha**, com autenticação customizada em PL/SQL e senha armazenada com hash SHA-512 + *salt*
- **Dashboard (Início)** com:
  - mensagem de boas-vindas com o nome do usuário logado
  - gráfico de pizza com as **horas jogadas por jogo**
  - gráfico de barras com o **total de jogos zerados e não zerados**
  - os dois gráficos mostram só os dados do usuário logado
- **Categorias**: cadastro, edição e exclusão de categorias de jogos (ex.: RPG, Ação, Corrida)
- **Jogos**: catálogo de jogos, cada um ligado a uma categoria
- **Meus Jogos**: lista dos jogos do usuário logado, com horas jogadas ("Não informado" quando vazio) e status "zerado" (Sim/Não), e cadastro e edição em janela modal
  - o usuário do registro é preenchido automaticamente a partir da sessão (`G_USER_ID`)
  - o botão **"Novo jogo"** abre o cadastro de jogo do catálogo por cima do registro; quando ele fecha, a lista de jogos é atualizada e o jogo recém-criado já vem selecionado
  - mensagens de sucesso ao adicionar, atualizar e remover um jogo da lista

---

## 🧱 Tecnologias

| Tecnologia | Uso |
|---|---|
| Oracle APEX 26.2.1 | Construção da aplicação (low-code) |
| Oracle Database (23ai ou superior) | Banco de dados; usa o tipo nativo `BOOLEAN` |
| PL/SQL | Autenticação e regras de negócio (`pkg_auth`) |
| Universal Theme (estilo Iris) | Tema visual padrão do APEX |
| APEXlang (`.apx`) | Exportação legível da aplicação, boa para versionar |

---

## 📁 Estrutura do projeto

```
meu-gameplay/
├── application.apx            # Definição geral do app (nome, navegação, autenticação, idioma)
├── page-groups.apx            # Grupos de páginas
├── pages/                     # Uma página do APEX por arquivo
│   ├── p00000-global-page.apx
│   ├── p00001-home.apx               # Início / dashboard com gráficos
│   ├── p00002-categorias.apx         # Relatório de categorias
│   ├── p00003-nova-categoria.apx     # Formulário modal de categoria
│   ├── p00004-jogos.apx              # Relatório de jogos
│   ├── p00005-novo-jogo.apx          # Formulário modal de jogo
│   ├── p00008-meus-jogos.apx         # Jogos do usuário logado
│   ├── p00009-registrar-novo-jogo.apx# Formulário modal: horas jogadas / zerado
│   └── p09999-login.apx              # Página de login
├── shared-components/         # Componentes compartilhados
│   ├── authentications.apx    # Esquemas de autenticação
│   ├── authorizations.apx     # Esquemas de autorização
│   ├── app-items.apx          # Itens globais (G_USER_ID, G_USER_NOME)
│   ├── lists/                 # Menu lateral e barra de navegação
│   ├── lovs/                  # Listas de valores (jogos, categorias, usuários, boolean)
│   ├── breadcrumbs/           # Navegação estrutural
│   ├── static-files/icons/    # Ícones do app
│   └── themes/                # Universal Theme
├── supporting-objects/        # Objetos de suporte e substituições
├── database/                  # Scripts do banco de dados
│   ├── ddl_banco_full.sql     # Tabelas, constraints, chaves estrangeiras e package pkg_auth
│   └── user.sql               # Cria um usuário de teste
├── sql/
│   └── f148903.sql            # Exportação SQL tradicional do app (para importar)
└── deployments/default.json   # Configuração de deploy (ID do app)
```

---

## 🗄️ Modelo de dados

```
USUARIOS                 CATEGORIA
─────────                ─────────
ID (PK)                  ID (PK)
NOME                     NOME (único)
EMAIL (único)                │
SENHA (salt:hash)            │ 1
    │ 1                      │
    │                        │ N
    │ N                  JOGO
JOGO_USUARIO ─────── N:1 ─────────
─────────────            ID (PK)
ID (PK)                  NOME (único)
JOGO_ID (FK)             CATEGORIA_ID (FK)
USUARIO_ID (FK)
HORAS_JOGADAS
ZERADO (BOOLEAN)
```

**Chaves estrangeiras:**

| Constraint | Coluna | Referência |
|---|---|---|
| `JOGO_CON` | `JOGO.CATEGORIA_ID` | `CATEGORIA.ID` |
| `JOGO_USUARIO_CON` | `JOGO_USUARIO.JOGO_ID` | `JOGO.ID` |
| `JOGO_USUARIO_USUARIO_ID` | `JOGO_USUARIO.USUARIO_ID` | `USUARIOS.ID` |

> As FKs não têm `ON DELETE`. Por isso, uma categoria que ainda tem jogos, ou um jogo que algum usuário registrou, não pode ser excluído: o banco retorna `ORA-02292`.

- **USUARIOS**: quem usa o app. A senha fica salva como `salt:hash`.
- **CATEGORIA**: gênero ou categoria do jogo.
- **JOGO**: catálogo geral de jogos.
- **JOGO_USUARIO**: tabela associativa com o progresso de cada usuário em cada jogo.

> A tabela `HTMLDB_PLAN_TABLE` que aparece no `ddl_banco_full.sql` é criada pelo próprio APEX (*explain plan*) e não faz parte do modelo do app.

---

## 🔐 Autenticação

O app usa o esquema customizado **"Login por Email"**, implementado no package `pkg_auth`, que fica no fim do [`ddl_banco_full.sql`](database/ddl_banco_full.sql):

| Rotina | O que faz |
|---|---|
| `criar_usuario(nome, email, senha)` | Gera um *salt* aleatório de 32 caracteres, calcula `SHA512(salt || senha)` e grava `salt:hash` |
| `autenticar(username, password)` | Função de autenticação do APEX: busca o usuário pelo e-mail, separa o *salt* do hash e compara |
| `pos_login` | Procedimento pós-autenticação: preenche os itens globais `G_USER_ID` e `G_USER_NOME` |

O e-mail é sempre normalizado com `lower(trim(...))`, então o login não diferencia maiúsculas de minúsculas.

---

## 🚀 Como rodar

### Pré-requisitos

- Um workspace Oracle APEX 26.2.1 ou superior (versão em que o app foi exportado), por exemplo em [apex.oracle.com](https://apex.oracle.com) ou no Oracle Autonomous Database
- Oracle Database 23ai ou superior, por causa da coluna `BOOLEAN`

### Passo a passo

1. **Criar os objetos do banco**: no *SQL Workshop → SQL Scripts*, execute nesta ordem:
   1. [`database/ddl_banco_full.sql`](database/ddl_banco_full.sql): tabelas, constraints, FKs e o package `pkg_auth`
   2. [`database/user.sql`](database/user.sql) (opcional; cria um usuário de teste)

   > Num schema limpo, os `CREATE UNIQUE INDEX` do script retornam `ORA-00955` (o nome já está em uso), porque as constraints `PRIMARY KEY`/`UNIQUE` já criaram esses índices. Pode ignorar esses erros: o resultado final fica correto.

2. **Importar a aplicação**: em *App Builder → Import*, envie o arquivo [`sql/f148903.sql`](sql/f148903.sql) e siga o assistente.

3. **Acessar**: rode o app e entre com o usuário de teste:

   | E-mail | Senha |
   |---|---|
   | `lucas@teste.com` | `123456` |

### Criar novos usuários

```sql
begin
  pkg_auth.criar_usuario('Seu Nome', 'seu@email.com', 'sua_senha');
  commit;
end;
```

---

## 🧭 Navegação

| Menu | Página | Descrição |
|---|---|---|
| 🏠 Início | 1 | Dashboard com gráficos |
| 🏆 Meus Jogos | 8 → 9 (→ 5) | Jogos do usuário, o formulário de registro e, a partir dele, o cadastro rápido de um jogo novo |
| 🎮 Jogos | 4 → 5 | Catálogo de jogos e o formulário |
| 🔷 Categorias | 2 → 3 | Categorias e o formulário |

---

## 📚 Conceitos de APEX praticados

- Páginas de **relatório clássico** e **formulário** com *Automatic Row Processing (DML)*
- **Diálogos modais** com *Dynamic Action* que atualiza o relatório quando o diálogo fecha
- **Modal aberto a partir de outro modal** (diálogo não encadeado, `chained: false`) e processo *Close Dialog* com **Items to Return** (`P5_ID`)
- *Dynamic Action* com **JavaScript** (`apex.item().refresh()` e `setValue()`) para selecionar o jogo recém-criado
- Item **Switch** ligado a uma coluna `BOOLEAN`
- **Mensagens de sucesso** em processos
- **Página de login** com a opção "Lembrar nome do usuário" (cookie)
- **Gráficos** (pizza e barras) a partir de consultas SQL
- **Listas de valores (LOV)** compartilhadas
- **Autenticação customizada** com PL/SQL e procedimento pós-login
- **Application Items** para guardar dados da sessão (`G_USER_ID`, `G_USER_NOME`)
- **Substitution strings** (`&G_USER_NOME.`, `&APP_USER.`)
- Exportação em **APEXlang** (`.apx`) para versionar no Git

---

## 🛠️ Próximos passos

Ideias de funcionalidades novas para continuar o estudo, cada uma com o recurso do APEX que ela ajuda a praticar:

- [ ] **Cadastro de usuários**: página pública de auto-registro usando `pkg_auth.criar_usuario` *(página pública, validações)*
- [ ] **Perfil de administrador**: só administradores gerenciam Categorias e Jogos; os outros usuários só registram seus jogos *(esquemas de autorização, grupo de páginas "Administração")*
- [ ] **Registro de jogo mais completo**: novos campos no formulário de Meus Jogos (`JOGO_USUARIO`) *(novas colunas, nova tabela, LOV, Select Many, Star Rating, Text Area, Date Picker)*
  - status além de "zerado": *Quero jogar*, *Jogando*, *Zerado*, *Abandonado*
  - avaliação de 1 a 5 estrelas e uma resenha curta
  - datas de início e de conclusão
  - plataformas (PC, PlayStation, Xbox, Switch...), cadastradas numa nova tabela
- [ ] **Busca e filtros** em Meus Jogos por categoria, status e plataforma *(Faceted Search ou Interactive Report)*
- [ ] **Dashboard mais completo**: total de horas, horas por categoria, jogos mais jogados e jogos zerados por mês *(Metric Cards, novos gráficos)*
- [ ] **Importar dados de uma API pública de jogos** (nome, capa, gênero) *(REST Data Source)*

---

## 👤 Autor

**Lucas Lima**: projeto pessoal de estudo de Oracle APEX.
