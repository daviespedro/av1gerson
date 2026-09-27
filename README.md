# Greencode

Uma interface de linha de comando (CLI) que atua como o núcleo operacional de uma **plataforma de logística reversa de resíduos eletrônicos**. O sistema gerencia o cadastro de organizações geradoras, o controle de lotes recebidos, a rastreabilidade individual de equipamentos (da chegada à destinação final) e fornece uma trilha de auditoria completa de todas as operações.

Desenvolvido em **Node.js + TypeScript**, o projeto utiliza persistência em arquivos de texto cifrados (sem dependência de banco de dados relacional nesta etapa) e implementa *journaling write-ahead* para garantir integridade, rastreabilidade e recuperação segura em caso de falhas.

## Sumário

- [Requisitos](#requisitos)
- [Instalação](#instalação)
- [Executando o Sistema](#executando-o-sistema)
- [Primeiro Uso (Provisionamento)](#primeiro-uso-provisionamento)
- [Papéis e Permissões](#papéis-e-permissões)
- [Comandos Disponíveis](#comandos-disponíveis)
- [Exemplo de Sessão](#exemplo-de-sessão-completa)
- [Testes Automatizados](#testes-automatizados)
- [Arquitetura e Estrutura](#estrutura-do-projeto)
- [Documentação Adicional](#documentação-adicional)

---

## Requisitos

- **Node.js 18 LTS ou superior** (testado nativamente nas versões 18 e 20). O `npm` já vem incluso e é utilizado para gerenciar dependências e compilar o projeto.
- Terminal com suporte a UTF-8 (Padrão no Linux/macOS. No Windows, recomenda-se o Windows Terminal ou PowerShell moderno).

> **Nota de Portabilidade:** O sistema não depende de bancos de dados externos, serviços de rede ou bibliotecas nativas compiladas. Ele roda puramente no runtime do Node.js, garantindo execução fluida entre Windows e Linux sem ajustes de código.

---

## Instalação

### Ubuntu 24.04 LTS (e derivadas)

```bash
# 1. Instale o Node.js 18+ (via NodeSource, se necessário):
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2. Confirme a instalação:
node -v   # Deve exibir v18.x ou superior
npm -v

# 3. Clone/extraia o projeto e instale as dependências:
cd greencode
npm install

# 4. Compile o código TypeScript:
npm run build
```

### Windows 10 ou superior

1. Baixe e instale o Node.js LTS diretamente do [site oficial](https://nodejs.org).
2. Abra o **PowerShell** na pasta raiz do projeto extraído.
3. Execute os comandos de configuração:
   ```powershell
   node -v
   npm -v
   npm install
   npm run build
   ```

> 🛡️ **Segurança no Windows (Ambientes Corporativos):** O sistema tenta aplicar permissões restritas aos arquivos de configuração (`chmod 600`), mas isso tem efeito limitado no NTFS. Recomenda-se restringir a pasta `data/` via ACL do Windows apenas ao usuário de serviço que executará o Greencode. (Mais detalhes em `docs/SECURITY.md`).

---

## Executando o Sistema

Após a compilação, inicie a CLI com:

```bash
npm start
```
Ou, alternativamente:
```bash
node dist/cli/index.js
```

**Diretório de Dados Customizado:**
Por padrão, os dados são salvos em uma pasta `data/` criada onde o comando é executado. Para definir um diretório customizado (útil para testes ou múltiplos ambientes), utilize a variável de ambiente `GREENCODE_DATA_DIR`:

```bash
# Linux/macOS
GREENCODE_DATA_DIR=/caminho/para/dados npm start

# Windows (PowerShell)
$env:GREENCODE_DATA_DIR="C:\caminho\para\dados"; npm start
```

---

## Primeiro Uso (Provisionamento)

Se o arquivo de configuração mestre (`data/master.config.json`) não existir, o sistema entra automaticamente em **modo de provisionamento inicial** ao ser executado:

```text
========================================
 Greencode - Logística Reversa
========================================
[AVISO] Arquivo de configuracao mestre nao encontrado.
[INFO] Entrando em modo de provisionamento inicial do sistema.

Defina o nome de usuario do administrador: admin
Defina a senha do administrador (min. 8 caracteres, 1 maiuscula, 1 numero): ********
Confirme a senha: ********

[OK] Provisionamento concluido. Chave de criptografia mestra gerada automaticamente.
[OK] Administrador 'admin' criado. Faca login para continuar.
```
*A chave criptográfica mestra (AES-256) é gerada internamente e de forma automática.*

---

## Papéis e Permissões

O acesso é rigidamente controlado via RBAC (Role-Based Access Control). O menu de ajuda da CLI (`ajuda`) adapta-se dinamicamente ao nível de acesso do usuário.

| Papel | Escopo de Acesso |
|---|---|
| `admin` | Acesso total. Pode gerenciar usuários, configurar parâmetros globais, aplicar políticas de retenção e executar todas as ações dos demais papéis. |
| `cadastro` | Focado no cliente: cadastra organizações geradoras, gerencia contratos de coleta e cria/consulta lotes. |
| `almoxarifado` | Focado na operação: cadastra equipamentos (com geração de código interno), gerencia triagem, e altera status/estado físico dos itens. |
| `auditor` | Acesso "Read-Only" estrito: visualiza organizações, lotes, equipamentos, rastreabilidade e journal. Nenhuma operação de escrita é permitida. |

---

## Comandos Disponíveis

Agrupados por contexto de uso. Valores com espaços devem ser envolvidos em aspas (`"..."`).

### 🔐 Sessão e Sistema
```bash
login                     # Inicia sessão (solicita credenciais)
logout                    # Encerra sessão ativa
ajuda                     # Lista comandos permitidos para seu papel
sair                      # Encerra a CLI
config set <chave> <val>  # Altera parâmetros globais (ex: config set aliquota_icms 0.18)
config get [chave]        # Lê parâmetros
config retencao           # Aplica política de retenção do journal
```

### 👤 Gestão de Usuários
```bash
usuario criar <user> --senha <senha> --papel <admin|cadastro|almoxarifado|auditor>
usuario desativar <user>
usuario listar
```

### 🏢 Organizações e Contratos
```bash
org criar --cnpj <cnpj> --razao "<razao social>" [--contato <contato>]
org ver <cnpj>
org listar

contrato criar --org <cnpj> --inicio <AAAA-MM-DD> [--frequencia <dias>] [--condicoes "texto"]
contrato ver <id>
contrato listar [--org <cnpj>]
contrato encerrar <id>    # Marca como inativo sem apagar o histórico
```

### 📦 Lotes de Recebimento
```bash
lote criar --org <cnpj> --nf <numero> --transp <transportadora> [--data AAAA-MM-DD]
lote ver <id>
lote listar
lote status <id> --novo <recebido|em_triagem|triagem_completa|encerrado>
```

### 💻 Equipamentos e Rastreabilidade
```bash
# O --codigo é opcional. Se omitido, gera um código interno (INT-timestamp-sufixo).
equipamento cadastrar --lote <id> [--codigo <cod>] --tipo <tipo>
equipamento triagem <id>
equipamento mover <id> --status <status>
equipamento estado <id> --novo <estado> [--justificativa "texto"]
equipamento ver <id>
equipamento listar [--lote <id>]
equipamento depreciacao <id> --meses <numero_de_meses_de_uso>

rastreio equipamento <id>
rastreio journal [--ultimas N]
```

> **Tipos de Equipamento:** Subclasses nativas incluem `terminal`, `cpu`, `monitor` e `servidor` (com regras próprias de depreciação). Tipos não mapeados caem automaticamente no padrão estendido (`EquipamentoGenerico`), evitando que o cadastro falhe.

---

## Exemplo de Sessão Completa

```text
greencode> usuario criar cad1 --senha CadSenha1 --papel cadastro
[OK] Usuario 'cad1' criado com papel 'cadastro'.

greencode> logout
[OK] Logout efetuado.

Usuario: cad1
Senha: ********
[OK] Login efetuado. Bem-vindo(a), cad1 (cadastro).

greencode> org criar --cnpj 11222333000181 --razao "Hospital Central"
[OK] Organizacao criada: Hospital Central (11.222.333/0001-81)

greencode> lote criar --org 11222333000181 --nf 123456 --transp TransRapida
[OK] Lote criado: bc8287c8-... | org=11222333000181 | nf=123456 | status=recebido
```

---

## Testes Automatizados

A suíte de testes (composta por 75 verificações) interage com a CLI real emulando um usuário final, utilizando o diretório nativo temporário do S.O. (`os.tmpdir()`) para garantir isolamento total.

```bash
npm run build      # Obrigatório antes dos testes
npm test           # Executa todas as 5 suítes abaixo

# Execuções individuais:
npm run test:journey      # Jornada: provisionamento -> 4 papéis -> rastreabilidade
npm run test:security     # Segurança: permissões, senhas, autenticação e erros
npm run test:domain       # Domínio: polimorfismo, cálculo de depreciação e fábricas
npm run test:compliance   # Regras: unicidade de códigos, retenção de contratos
npm run test:write-ahead  # Persistência: checagem de checksums, lacunas e ordem do journal
```

---

## Estrutura do Projeto

O projeto segue princípios sólidos de Orientação a Objetos, Design Patterns (Factories) e Clean Architecture adaptada.

```text
greencode/
├── src/
│   ├── types.ts                    # Enums e Tipos base
│   ├── models/                     # ENTIDADES DE DOMÍNIO (Classes Reais)
│   │   ├── Entidade.ts
│   │   ├── equipamentos/           # SUBCLASSES (Herança + Polimorfismo)
│   │   │   ├── Terminal.ts, CPU.ts, Monitor.ts, Servidor.ts
│   │   │   └── EquipamentoGenerico.ts
│   ├── validation/                 # Classes base e contratos de Validação
│   ├── factories/                  # FABRICAS (Factory Method para instâncias e reidratação)
│   ├── core/
│   │   ├── crypto.ts               # AES-256-GCM, SHA-256, scrypt
│   │   ├── storage.ts              # Escrita cifrada atômica em disco
│   │   ├── journal.ts              # Journaling Write-Ahead
│   │   └── auth.ts                 # Autenticação e RBAC
│   ├── repositories/               # Persistência: orquestra Fabrics e Storage
│   ├── commands/                   # Lógica de comandos e roteamento
│   └── cli/                        # Entrypoint, readline interativo, histórico
├── tests/                          # Suítes de testes isoladas
├── docs/                           # Documentação detalhada
└── package.json
```

### Destaques do Modelo de Domínio
1. **Polimorfismo Real:** O comando `equipamento depreciacao` atua sobre a classe abstrata `Equipamento`. O cálculo e a curva de depreciação variam automaticamente em tempo de execução dependendo da subclasse instanciada (`Servidor` possui teto máximo; `Monitor` tem curva diferente, etc).
2. **Reidratação Inteligente:** Os Repositórios não constroem objetos manualmente, eles delegam isso para as **Factories**, que analisam o JSON armazenado e reidratam as propriedades na subclasse polimórfica correta.
3. **Open/Closed Principle:** O sistema está protegido contra tipos desconhecidos graças ao fallback transparente da `EquipamentoFactory` para o `EquipamentoGenerico`.

---

## Documentação Adicional

- [`docs/SECURITY.md`](docs/SECURITY.md) — Justificativa arquitetural de segurança, criptografia (AES-256-GCM, PBKDF2/scrypt), gerenciamento de sessão, e caminhos de evolução para integração com bancos de dados relacionais.
- [`docs/FAILURE_SCENARIOS.md`](docs/FAILURE_SCENARIOS.md) — Matriz completa de cenários de falha testados pela aplicação (validações, crash-recovery, manipulação de state) e como o sistema se recupera.# av1gerson
