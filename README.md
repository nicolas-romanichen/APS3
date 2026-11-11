# Dossiê de Arquitetura Viva — Assistente de IA para Exportação (Fiep)

**Funcionalidade:** validação e rastreabilidade das fontes normativas.  
**Requisitos:** RF10 (citações), RF11 (incerteza) e RF12 (atualização regulatória).

## Objetivo
Modelar como o assistente consulta regras aduaneiras, verifica suas fontes oficiais, comunica incertezas e registra pendências de atualização. O material é uma **proposta arquitetural acadêmica**, baseada nos requisitos, na visão de domínio e nos diagramas de classes anexados.

## 1. Diagrama de sequência
Mostra as chamadas síncronas, o envio assíncrono de eventos, os DTOs e o tratamento alternativo de fontes não verificadas.

```mermaid
sequenceDiagram
    autonumber
    actor U as Exportador
    participant API as ConsultaController
    participant S as OrientacaoService
    participant R as RegraRepository
    participant V as ValidacaoNormativaService
    participant F as FonteRepository
    participant IA as ModeloIAAdapter
    participant Q as FilaAtualizacao

    U->>API: consultar(ConsultaDTO)
    API->>S: gerarOrientacao(ConsultaDTO)
    S->>R: buscarAplicaveis(OperacaoDTO)
    R-->>S: ListaRegraDTO

    loop Para cada RegraDTO
        S->>V: validarRegra(RegraDTO)
        V->>F: buscarFontes(regraId)
        F-->>V: ListaFonteDTO
        V-->>S: ValidacaoDTO
        opt Fonte exige revisao
            V-)Q: publicar(AtualizacaoSolicitadaDTO)
        end
    end

    alt Dados insuficientes ou fontes nao verificadas
        S-->>API: OrientacaoDTO com incerteza e pendencias
    else Contexto e fontes verificados
        S->>IA: gerarTexto(ContextoFundamentadoDTO)
        IA-->>S: TextoGeradoDTO
        S-->>API: OrientacaoDTO com citacoes
    end
    API-->>U: OrientacaoDTO
```

[Arquivo Mermaid](docs/diagrams/sequencia.mmd)

**Operações públicas propostas (receptor → método):** `ConsultaController.consultar(ConsultaDTO)`; `OrientacaoService.gerarOrientacao(ConsultaDTO)`; `RegraRepository.buscarAplicaveis(OperacaoDTO)`; `ValidacaoNormativaService.validarRegra(RegraDTO)`; `FonteRepository.buscarFontes(regraId)`; `ModeloIAAdapter.gerarTexto(ContextoFundamentadoDTO)`; `FilaAtualizacao.publicar(AtualizacaoSolicitadaDTO)`. Os retornos são `OrientacaoDTO`, `ListaRegraDTO`, `ValidacaoDTO`, `ListaFonteDTO` e `TextoGeradoDTO`, conforme as setas. A chamada inicial `consultar` representa a API exposta ao exportador. Essas operações complementam o modelo de classes existente e precisam ser declaradas publicamente na implementação.

## 2. Diagrama de atividades
Três raias mostram a consulta, a validação paralela de origem e vigência e os caminhos mutuamente exclusivos. Mermaid usa `subgraph` como aproximação visual de raias UML; as barras Fork/Join são representadas explicitamente.

```mermaid
flowchart TB
    subgraph U[RAIA 1 - EXPORTADOR]
        direction LR
        inicio([Inicio]) --> consulta[Informar consulta]
        exibir[Visualizar resposta] --> fim([Fim])
    end

    subgraph B[RAIA 2 - BACKEND]
        direction TB
        interpretar[Interpretar consulta e buscar regras]
        dados{Dados suficientes?}
        solicitar[Solicitar dados faltantes]
        fork["━━━━ FORK ━━━━"]
        join["━━━━ JOIN ━━━━"]
        fontes{Fontes verificadas?}
        responder[Gerar resposta com citacoes]
        incerto[Gerar resposta com incerteza]
        enviar[Enviar OrientacaoDTO]
    end

    subgraph N[RAIA 3 - VALIDACAO NORMATIVA]
        direction LR
        origem[Conferir origem oficial]
        vigencia[Conferir vigencia]
        pendencia[Registrar pendencia de revisao]
    end

    consulta --> interpretar --> dados
    dados -->|"[nao]"| solicitar --> enviar
    dados -->|"[sim]"| fork
    fork --> origem --> join
    fork --> vigencia --> join
    join --> fontes
    fontes -->|"[sim]"| responder --> enviar
    fontes -->|"[nao]"| incerto --> pendencia --> enviar
    enviar --> exibir

    classDef barra fill:#333,stroke:#333,color:#fff
    class fork,join barra
```

[Arquivo Mermaid](docs/diagrams/atividades.mmd)

## 3. Diagrama de estados
A entidade `FonteNormativa` passa por cadastro, verificação, revisão e arquivamento. O estado de verificação contém subestados; `Verificada` tem ações internas.

```mermaid
stateDiagram-v2
    direction LR
    [*] --> Cadastrada
    Cadastrada --> Verificacao: iniciarValidacao / conferirMetadados

    state Verificacao {
        direction TB
        [*] --> Origem
        Origem --> Vigencia: origemConfirmada [urlOficialValida]
        Origem --> Revisao: origemNaoConfirmada [urlOficialInvalida]
        Vigencia --> Concluida: vigenciaConfirmada [normaVigente]
        Vigencia --> Revisao: vigenciaNaoConfirmada [normaNaoConfirmada]
        Revisao --> Origem: novaEvidencia / revalidar
        Concluida --> [*]
    }

    Verificacao --> Verificada: validacaoAprovada / registrarEvidencias
    Verificacao --> Pendente: validacaoInconclusiva / sinalizarIncerteza
    Pendente --> Verificacao: novaEvidencia / reiniciarValidacao
    Verificada --> Desatualizada: mudancaDetectada / suspenderCitacao
    Desatualizada --> Verificacao: iniciarRevisao / consultarVersao
    Verificada --> Arquivada: revogacaoConfirmada / preservarHistorico
    Desatualizada --> Arquivada: revogacaoConfirmada / preservarHistorico
    Pendente --> Arquivada: descarteJustificado / registrarMotivo
    Arquivada --> [*]

    Verificada: entry / registrarDataValidacao
    Verificada: do / disponibilizarParaCitacao
    Verificada: exit / registrarMotivoSaida
```

[Arquivo Mermaid](docs/diagrams/estados.mmd)

## 4. C4 — Contêineres (Nível 2)
Apresenta a interface, API, validador, banco, fila e worker, com tecnologias e protocolos **propostos**.

```mermaid
C4Container
    title C4 Nivel 2 - Assistente de IA para Exportacao (proposta)
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")

    Person(usuario, "Exportador", "Consulta orientacoes")
    Person(curador, "Curador", "Revisa fontes")
    System_Ext(oficiais, "Fontes oficiais", "Normas e portais")
    System_Ext(llm, "LLM externo", "Gera texto")

    System_Boundary(fiep, "Assistente Fiep") {
        Container(web, "Interface Web", "React", "Consultas e revisao")
        Container(api, "API", "Python / FastAPI", "Orquestra orientacoes")
        Container(val, "Validador", "Python", "Confere fontes e vigencia")
        ContainerDb(db, "Base normativa", "PostgreSQL", "Regras e evidencias")
        ContainerQueue(q, "Fila", "RabbitMQ", "Pedidos de revisao")
        Container(worker, "Worker", "Python", "Atualiza pendencias")
    }

    Rel(usuario, web, "Consulta", "HTTPS")
    Rel(curador, web, "Revisa", "HTTPS")
    Rel(web, api, "ConsultaDTO", "HTTPS/JSON")
    Rel(api, db, "Busca regras", "SQL/TCP")
    Rel(api, val, "Valida regras", "HTTP/JSON")
    Rel(val, db, "Busca fontes", "SQL/TCP")
    Rel(api, llm, "Gera resposta", "HTTPS/JSON")
    Rel(val, q, "Solicita revisao", "AMQP")
    Rel(q, worker, "Entrega evento", "AMQP")
    Rel(worker, oficiais, "Consulta normas", "HTTPS")
    Rel(worker, db, "Registra evidencias", "SQL/TCP")
```

[Arquivo Mermaid](docs/architecture/workspace.mmd)

## 5. ADR 0001
A [decisão arquitetural](docs/architecture/adr-0001.md) propõe validar as fontes separadamente do LLM e registrar atualizações para revisão humana.

## Rastreabilidade

| Requisito | Atendimento |
|---|---|
| **RF10** | Fonte oficial, referência legal e citação na resposta |
| **RF11** | Resposta com ressalvas quando faltam dados ou validação |
| **RF12** | Evento assíncrono, revisão e revalidação de fontes |

## Estrutura

```text
aps3-nome-sobrenome/
├── README.md
└── docs/
    ├── architecture/
    │   ├── workspace.mmd
    │   └── adr-0001.md
    └── diagrams/
        ├── sequencia.mmd
        ├── atividades.mmd
        └── estados.mmd
```

**Observações:** a verificação de fonte não equivale a parecer jurídico. A infraestrutura (React, FastAPI, PostgreSQL e RabbitMQ) é ilustrativa e deve ser confirmada com a equipe. Para visualizar os diagramas, use GitHub ou um editor compatível com Mermaid, incluindo C4.
