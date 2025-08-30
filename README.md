````markdown
# Rotina Interna de Logs no Sankhya
> Projeto didático implementado com base nas aulas do SatyaPass (Java no Sankhya).

Este repositório entrega uma rotina de gerenciamento interno de logs no Sankhya, persistindo informações em tabelas próprias e oferecendo botões/ações para registrar execuções, detalhar atividades e efetuar limpeza automática conforme política de retenção.

## Sumário
- [Objetivo e escopo](#objetivo-e-escopo)
- [Modelo de dados](#modelo-de-dados)
- [Arquitetura e componentes](#arquitetura-e-componentes)
- [Fluxo de execução](#fluxo-de-execução)
- [Padrões transacionais (JAPE/JapeSession)](#padrões-transacionais-japejapesession)
- [Exemplos de uso (trechos reais)](#exemplos-de-uso-trechos-reais)
- [Implantação no Sankhya (Módulo Java)](#implantação-no-sankhya-módulo-java)
- [Pontos de extensão](#pontos-de-extensão)
- [Créditos](#créditos)

---

## Objetivo e escopo
- Objetivo: registrar e consultar logs de rotinas/serviços dentro do ERP, sem depender do log do servidor (WildFly).
- Escopo:
  - Log Pai (`AD_STPLOG`): uma execução/unidade de processamento.
  - Log Filho (`AD_STPLOGITE`): eventos/etapas vinculadas ao log pai.
  - Serviço (`AD_STPSER`): catálogo de serviços que podem ser logados e suas configurações (ex.: retenção de dias).
  - Limpeza automática: ação agendada para remoção de logs conforme política.

---

## Modelo de dados
As telas e instâncias foram construídas no Sankhya Place:

### Tabelas/Instâncias
- `AD_STPSER` — Serviço do Log
  - `CODSER` (número inteiro) – identificador do serviço
  - `DESCRICAO` (texto)
  - `ATIVO` (lista de opções: S/N)
  - `QTD DIAS` (número inteiro) – política de retenção

- `AD_STPLOG` — Rotina de Log (Log Pai)
  - `CODLOG` (número inteiro) – PK
  - `CODSER` (número inteiro) – FK para serviço
  - `DTEXEC` (data/hora) – início/registro da execução
  - `MODALIDADE` (lista de opções)
  - `STATUSLOG` (lista de opções)
  - `STATUSEXEC` (lista de opções)
  - `CODUSU` (número inteiro) – usuário

- `AD_STPLOGITE` — Itens do Log (Log Filho)
  - `CODLOG` (número inteiro) – FK para `AD_STPLOG`
  - `SEQUENCIA` (número inteiro)
  - `TEMPORIZADOR` (data/hora)
  - `DESCRICAO` (texto)
  - `ATIVIDADE` (texto longo/CLOB)
  - `CATEGORIA` (texto)
  - `STATUS` (lista de opções)

### Diagrama ER (simplificado)
```mermaid
erDiagram
  AD_STPSER ||--o{ AD_STPLOG : "CODSER"
  AD_STPLOG ||--o{ AD_STPLOGITE : "CODLOG"

  AD_STPSER {
    INT CODSER
    TEXT DESCRICAO
    TEXT ATIVO
    INT QTD_DIAS
  }
  AD_STPLOG {
    INT CODLOG
    INT CODSER
    DATETIME DTEXEC
    TEXT MODALIDADE
    TEXT STATUSLOG
    TEXT STATUSEXEC
    INT CODUSU
  }
  AD_STPLOGITE {
    INT CODLOG
    INT SEQUENCIA
    DATETIME TEMPORIZADOR
    TEXT DESCRICAO
    CLOB ATIVIDADE
    TEXT CATEGORIA
    TEXT STATUS
  }
````

---

## Arquitetura e componentes

### Pacotes principais

```
br.com.satyacode.satyapass.log
├─ exception/                 
├─ listener/                  
├─ model/                     
│  ├─ ModalidadeEnum
│  ├─ StatusExecucaoEnum
│  ├─ StatusLogEnum
│  └─ StatusItemEnum
├─ repository/                
│  ├─ LogRepository           
│  ├─ LogItemRepository       
│  └─ LogServicoRepository    
├─ service/
│  └─ LogService              
└─ view/                      
   ├─ BotaoAcao               
   ├─ DeletarLogBT            
   └─ DeletarLogAcaoAgendada  
```

### Repositórios

* LogRepository: CRUD Log Pai (`AD_STPLOG`).
* LogItemRepository: CRUD Log Filho (`AD_STPLOGITE`) + verificação de erro.
* LogServicoRepository: consulta serviços (`AD_STPSER`).

---

## Fluxo de execução

```mermaid
flowchart
    A["Factory.isApresentouErro(CODLOG)"] -->|Sim| B[H]
    A -->|Não| C[Fim]

  B --> C[LogRepository.incluirLogPai -> AD_STPLOG]
  C --> D[LogService.incluirItem]
  D --> E[LogItemRepository.incluirLogFilho -> AD_STPLOGITE]
  E --> F{Encerrar execução?}
  F -- Não --> D
  F -- Sim --> H[LogItemRepository.isApresentouErro(CODLOG)]
  H -- Sim --> I[LogRepository.atualizarStatus/Execução (ERRO)]
  H -- Não --> J[LogRepository.atualizarStatus/Execução (OK)]
  I --> K[Fim]
  J --> K[Fim]

  G --> L[Determina retenção (AD_STPSER)]
  L --> M[Excluir filhos: LogItemRepository.deletarLogFilho]
  M --> N[Excluir pais: LogRepository.deletarLog]
  N --> O[Fim]
```

---

## Padrões transacionais (JAPE/JapeSession)

* Métodos `...TM` encapsulam transação com `JapeSession.open()` e `hnd.execWithTX`.
* `hnd.setCanTimeout(false)` em operações longas.
* Exceções propagadas com `MGEModelException.throwMe(e)`.
* Atualizações via `prepareToUpdateByPK`.
* Exclusões via `deleteByCriteria` e `dao.delete`.
* Consultas de erro com `NativeSql COUNT(*)`.

---

## Exemplos de uso (trechos reais)

### Abertura do log pai e inclusão de itens

```java
LogService logService = new LogService();
boolean txAutomatica = true;

BigDecimal codLog = logService.incluirLog(
    BigDecimal.ONE,
    ModalidadeEnum.BOTAO_ACAO,
    StatusExecucaoEnum.EM_ANDAMENTO,
    BigDecimal.ZERO,
    txAutomatica
);

logService.incluirItem(codLog, "Mensagem 1", "Teste 123", "TESTE", StatusItemEnum.OK, txAutomatica);
logService.incluirItem(codLog, "Mensagem 2", "Teste 123", "TESTE", StatusItemEnum.ERRO, txAutomatica);

logService.atualizarStatusLogPai(codLog, true);
```

### Limpeza de logs

```java
new LogService().gerenciarExclusaodaRotinaDeLogs(true);
```

---

## Implantação no Sankhya (Módulo Java)

1. Build do `.jar` (Maven/Gradle).
2. Upload no ERP: Configurações → Avançado → Módulo Java.
3. Configurar UI/Agendador:

   * Botões de Ação:

     * `br.com.satyacode.satyapass.log.view.BotaoAcao`
     * `br.com.satyacode.satyapass.log.view.DeletarLogBT`
   * Ação Agendada:

     * `br.com.satyacode.satyapass.log.view.DeletarLogAcaoAgendada`

---

## Pontos de extensão

* Relatórios e filtros.
* Auditoria de execução.
* Melhorias de segurança (binds).
* Observabilidade com notificações.
* Retenção granular.

---

## Créditos

Projeto implementado com base nas aulas do SatyaPass (Java no Sankhya).

```
