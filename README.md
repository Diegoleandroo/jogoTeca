# jogoTeca

Aplicação web CRUD para gerenciamento de uma coleção pessoal de jogos, com autenticação de usuários. Construída com **Flask**, **SQLAlchemy** e **MySQL**, renderizando as páginas no servidor com **Jinja2**.

> Projeto de estudo (baseado no curso da Alura), documentado aqui também como exercício de **análise arquitetural de software** — diagnóstico do estado atual e roteiro de evolução para uma arquitetura em camadas / DDD.

## Funcionalidades

- Listagem de jogos cadastrados, com capa, nome, categoria e console
- Cadastro de novo jogo (com upload de imagem de capa)
- Edição e exclusão de jogo
- Login e logout de usuário (sessão simples)
- Proteção CSRF em formulários e senhas com hash (bcrypt)

## Stack técnica

| Camada | Tecnologia |
|---|---|
| Backend | Python 3, Flask |
| ORM | Flask-SQLAlchemy |
| Banco de dados | MySQL |
| Formulários / CSRF | Flask-WTF, WTForms |
| Hash de senha | Flask-Bcrypt |
| Frontend | Jinja2, Bootstrap, jQuery |

## Como executar localmente

### Pré-requisitos
- Python 3.10+
- MySQL em execução localmente

### Passo a passo

```bash
# 1. Clonar o repositório
git clone https://github.com/<seu-usuario>/jogoTeca.git
cd jogoTeca

# 2. Criar e ativar um ambiente virtual
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate

# 3. Instalar as dependências
pip install flask flask_sqlalchemy flask_wtf flask_bcrypt mysql-connector-python

# 4. Configurar o banco de dados
# Edite config.py com suas credenciais de MySQL
# e então execute o script de preparação do banco:
python prepara_banco.py

# 5. Rodar a aplicação
python app.py
```

A aplicação estará disponível em `http://localhost:5000`.

> ⚠️ **Atenção:** `prepara_banco.py` executa `DROP DATABASE IF EXISTS` antes de recriar o schema — não execute em um banco que já contenha dados importantes.

## Estrutura do projeto

```
jogoTeca/
├── app.py              # Inicialização do Flask, db, csrf e bcrypt
├── config.py            # Configurações (SECRET_KEY, URI do banco, upload path)
├── models.py             # Entidades ORM: Jogos, Usuarios
├── helpers.py            # Formulários (WTForms) e funções auxiliares de imagem
├── views_game.py          # Rotas de CRUD de jogos
├── views_user.py          # Rotas de autenticação
├── prepara_banco.py        # Script de criação/seed do banco MySQL
├── templates/            # Views Jinja2 (lista, novo, editar, login)
├── static/              # CSS, JS, bibliotecas front-end
└── uploads/              # Imagens de capa dos jogos
```

## Diagnóstico arquitetural

Esta seção documenta uma análise de engenharia de software feita sobre o projeto, útil tanto para quem for dar manutenção quanto como material de estudo.

### Qual arquitetura é esta?

O projeto segue um **Transaction Script** sobre um MVC simplificado do Flask — **não** é Clean Architecture, Hexagonal ou DDD. É o padrão típico de tutoriais introdutórios: *"Fat Controller"* com o ORM acoplado direto nas rotas.

- **Model** (`models.py`) — classes SQLAlchemy sem nenhum comportamento, apenas colunas. Caracteriza um **Anemic Domain Model**.
- **View** (`templates/`) — papel correto, sem problemas relevantes.
- **Controller** (`views_game.py`, `views_user.py`) — a função de rota faz tudo: valida formulário, consulta o banco, aplica regra de negócio, manipula upload de arquivo e decide o redirect. Não há camada de aplicação nem de domínio separada.

| Característica exigida por DDD/Clean Architecture | Presente? |
|---|---|
| Domínio independente de framework | ❌ `models.py` depende diretamente do SQLAlchemy/Flask |
| Casos de uso isolados da camada web | ❌ lógica de negócio embutida nas rotas |
| Repository como abstração | ❌ `Jogos.query.filter_by(...)` chamado direto no controller |
| Entidades com comportamento | ❌ entidades são apenas *data holders* |
| Inversão de dependência | ❌ import circular entre `app.py` e `views_*.py` |

### Sem API

O sistema é inteiramente **server-side rendering**. Não há nenhuma rota que devolva JSON, nem `Blueprint` de API. Isso limita a evolução para um front-end desacoplado (SPA, app mobile) sem uma reestruturação.

### Bugs e riscos conhecidos

1. `autenticar()` pode lançar `AttributeError` se o nickname informado não existir (acessa `usuario.senha` antes de checar `usuario is None`).
2. `Jogos.__repr__` e `Usuarios.__repr__` referenciam `self.name`, mas o atributo correto é `self.nome`.
3. Credenciais hardcoded em `config.py` e `prepara_banco.py` (senha de banco, `SECRET_KEY`).
4. Upload de arquivo sem validação de extensão/`content-type`.
5. `prepara_banco.py` não é uma migration versionada — usa `DROP DATABASE IF EXISTS`.
6. Sem testes automatizados, sem paginação, sem tratamento de erros nas rotas.

### Roteiro de refatoração proposto

- [ ] Extrair `GameService` / `UserService` (camada de aplicação) para remover a regra de negócio das rotas
- [ ] Criar `GameRepository` com interface abstrata, separando a implementação concreta do SQLAlchemy
- [ ] Dar comportamento às entidades (ex.: `Jogo.pode_ser_excluido()`), saindo do modelo anêmico
- [ ] Adicionar uma camada de API (`/api/v1/jogos` em JSON) ao lado do SSR existente
- [ ] Substituir `prepara_banco.py` por migrations versionadas com Alembic
- [ ] Mover segredos para variáveis de ambiente (`python-dotenv`)
- [ ] Adicionar testes automatizados (unitários e de integração)

## Licença

Projeto de estudo, sem licença definida.
