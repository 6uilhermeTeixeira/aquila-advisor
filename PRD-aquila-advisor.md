# PRD: `aquila-advisor`

**Personal Advisor** · Setor Aquila (Visão, Alerta & Inovação) · Matriz dos 24 Anciãos
**Versão:** 0.1 (rascunho) · **Status:** Em definição · **Responsável:** _(preencher)_

---

## 1. Resumo

O `aquila-advisor` é o **assistente pessoal de IA** da plataforma: um copiloto privado que conhece o contexto de uma pessoa ou executivo (agenda, e-mails, tarefas, documentos, preferências, decisões passadas), **antecipa o que importa** e **delega a execução** às outras VMs.

No setor Aquila (visão, alerta e inovação), o Advisor é o **radar pessoal**: ele recebe os sinais produzidos pelo setor (alertas do `oculus`, previsões do `predict`, inteligência do `recon`) e os traduz em **briefings e recomendações priorizadas**. É também a **interface humana** mais natural para falar com o restante da plataforma.

**Premissa de produto:** assistente **pessoal/executivo**, privado e auditável, com autonomia graduada (nunca age de forma irreversível sem aprovação). Ver Q1 sobre o papel adicional de "relator de incidentes de segurança" que sugeri em conversa anterior.

**Nome comercial sugerido:** Executive AI Advisor (nome interno "Advisor" mantido como marca).

## 2. Objetivos

| # | Objetivo | Métrica de sucesso |
|---|---|---|
| O1 | Entregar um briefing diário priorizado e útil | ≥ 80% dos briefings avaliados como úteis pelo usuário no piloto |
| O2 | Manter memória de longo prazo **controlável pelo usuário** | Usuário consegue ver, editar e apagar qualquer memória; apagamento comprovável |
| O3 | Executar tarefas delegadas com segurança | 0 ações irreversíveis sem aprovação explícita |
| O4 | Integrar-se às ferramentas do dia a dia (agenda, e-mail, tarefas, mensagens) | Fluxos principais funcionando em ≥ 3 canais |
| O5 | Resistir a prompt injection vindo de e-mails, páginas e documentos | Suíte de testes adversariais aprovada a cada release |
| O6 | Operar como produto independente | Instalável sozinho, sem as demais VMs, em até 1 dia |

## 3. Escopo

### Dentro do escopo (v1)
- Conversa por **web e mensagens** (Telegram e/ou WhatsApp oficial), com voz opcional
- **Memória** em camadas (fatos e preferências, histórico, documentos pessoais) com painel de controle
- **Integrações pessoais:** agenda, e-mail, tarefas, notas, contatos
- **Briefing diário** e **alertas proativos** com regras de relevância e horário
- **Pesquisa e síntese** sob demanda (web e documentos pessoais), com fontes citadas
- **Delegação** a outras VMs por API/eventos, com níveis de autonomia
- **Aprovação humana** para ações sensíveis, com registro
- Trilha de auditoria de tudo o que o assistente leu, decidiu e fez

### Fora do escopo (v1)
- Aconselhamento jurídico, médico ou financeiro com pretensão de autoridade (encaminhar a `homo-lex` ou a profissionais)
- Execução financeira (pagamentos, transferências, compras) sem fluxo dedicado de aprovação forte
- Controle irrestrito de computador/terminal do usuário
- Análise de segurança de servidores, SIEM e threat intelligence (ficam em `oculus` e `recon`; o Advisor apenas as consome)
- Atendimento a clientes finais da empresa (isso é CRM/`homo-crm`)

> **Motivo:** assistentes com acesso a dados privados, conteúdo não confiável e capacidade de agir são o cenário de maior risco de segurança. A v1 limita o que pode agir sozinho.

## 4. Usuários e personas

| Persona | Necessidade |
|---|---|
| **Executivo / fundador** | Visão consolidada do dia, decisões priorizadas, delegação rápida |
| **Profissional autônomo** | Organização de agenda, e-mails, tarefas e pesquisa |
| **Gestor de equipe** | Resumos de projetos e pendências, acompanhamento de indicadores (`homo-bi`) |
| **Administrador de TI do cliente** | Provisionar usuários, definir políticas e conectores permitidos |
| **Outras VMs / Agent** | Consultar e acionar o Advisor via API, com escopo limitado |

## 5. Arquitetura

### 5.1 Base técnica
- **SO:** Ubuntu Server 24.04 LTS, instalação mínima (derivada do `taurus-seed`).
- **Execução:** cada usuário roda em **workspace isolado** (container/namespace próprio, volume criptografado). Serviços compartilhados mínimos.
- **Inferência de LLM:** modelo de pesos abertos em servidor próprio (GPU em `leo-compute` quando houver) ou API externa **somente** se a política do usuário permitir.
- **Segredos e tokens OAuth:** sempre no cofre do `aquila-sentinel`; o Advisor recebe credenciais efêmeras com escopo mínimo.
- **Configuração como código:** Ansible para o host, Compose versionado neste repositório.

### 5.2 Componentes

| Função | Componente recomendado | Alternativa |
|---|---|---|
| Orquestração do agente | **LangGraph** | CrewAI, código próprio |
| Camada de memória | **Mem0** ou **Letta** | Graphiti (grafo temporal), pgvector próprio |
| Banco e vetores | **PostgreSQL + pgvector** | Qdrant |
| Ferramentas e conectores | **MCP (Model Context Protocol)** + servidores MCP | Conectores próprios |
| Automação de fluxos | **n8n** ou **Activepieces** | Windmill |
| Gateway de canais (mensagens) | Gateway próprio ou **OpenClaw** como base de referência | Bots oficiais (Telegram, WhatsApp Business API) |
| LLM local | **vLLM** ou **Ollama** | API externa sob política |
| Voz (entrada e saída) | **faster-whisper** + **Piper** | Whisper, Coqui |
| Agenda, contatos e tarefas (opcional, self-hosted) | **Radicale** / **Nextcloud**, **Vikunja** | Google/Microsoft via OAuth |
| Guardrails e filtros | **LLM Guard** / **NeMo Guardrails** + OPA | Llama Guard |
| Observabilidade de LLM | **Langfuse** | OpenTelemetry |
| Avaliação e testes adversariais | **promptfoo** | Garak |
| Autenticação | OIDC via `aquila-sentinel` (Keycloak) | — |

### 5.3 Diagrama lógico

```
 Canais: Web · Telegram · WhatsApp · Voz
        │  (toda entrada é dado não confiável)
        ▼
  Gateway de canais ──► Autenticação (SSO) ──► Guardrails de entrada
        │
        ▼
  ┌───────────────────── Workspace do usuário (isolado) ─────────────────────┐
  │  Orquestrador do agente  ◄──►  Memória (fatos · histórico · documentos)  │
  │        │                                                                 │
  │        ├─ Ferramentas (MCP): agenda · e-mail · tarefas · web · arquivos  │
  │        │      └─ credenciais efêmeras ◄── aquila-sentinel (Vault)        │
  │        ├─ Motor de políticas (OPA): nível de autonomia por ação          │
  │        └─ Fila de aprovações ──► usuário (confirma / nega)               │
  └───────────┬──────────────────────────────────────────────────────────────┘
              │ eventos/APIs
              ▼
   Consulta: oculus · predict · recon · rag · lex · bi · crm
   Execução: via event bus (leo-logistics) → leo-media · leo-marketing · leo-devops
              │
   Logs de auditoria ──► aquila-oculus (somente envio)
```

## 6. Requisitos funcionais

### 6.1 Conversa e canais (CHN)
- **CHN-01** Interface web (PWA) responsiva e canal de mensagens (Telegram e/ou WhatsApp pela API oficial).
- **CHN-02** Voz opcional (transcrição e resposta falada) com processamento local.
- **CHN-03** Pareamento e **lista de remetentes permitidos**: mensagens de desconhecidos são ignoradas ou exigem aprovação.
- **CHN-04** Idioma principal português do Brasil, com suporte a inglês.

### 6.2 Memória (MEM)
- **MEM-01** Memória em camadas: preferências e fatos estáveis; histórico de interações; documentos pessoais.
- **MEM-02** **Painel de memória:** o usuário vê, corrige, exporta e apaga qualquer item.
- **MEM-03** Cada memória guarda origem, data e grau de confiança; informação sensível exige confirmação antes de ser salva.
- **MEM-04** "Esquecer" é real: remove do banco, dos vetores e dos backups conforme política de retenção.
- **MEM-05** Memória isolada por usuário e por tenant; nunca usada para treinar modelos.

### 6.3 Briefing e alertas (BRF)
- **BRF-01** Briefing diário configurável (horário, canal, tamanho) com agenda, pendências, mensagens importantes e riscos.
- **BRF-02** Alertas proativos por regras de relevância (prazo próximo, conflito de agenda, mudança em indicador, sinal vindo do setor Aquila).
- **BRF-03** Controle de **fadiga de alertas:** limites por período, agrupamento e modo "não perturbe".
- **BRF-04** Todo item do briefing mostra por que foi priorizado e de onde veio.

### 6.4 Tarefas e produtividade (TSK)
- **TSK-01** Leitura e proposta de ações em agenda, e-mail, tarefas e notas.
- **TSK-02** Rascunhos de e-mails, respostas e resumos; envio somente após aprovação (ver autonomia).
- **TSK-03** Pesquisa na web e em documentos pessoais, com fontes citadas e distinção entre fato e inferência.
- **TSK-04** Resumo de reuniões e documentos enviados pelo usuário.

### 6.5 Delegação a outras VMs (DLG)
- **DLG-01** Consultas a `homo-rag`, `homo-lex`, `homo-bi`, `homo-crm`, `aquila-predict` e `aquila-recon` por API.
- **DLG-02** Ordens de execução (mídia, campanhas, deploy) **apenas via event bus**, com contrato e identidade verificados.
- **DLG-03** Cada VM recebe somente o contexto mínimo necessário para a tarefa.

### 6.6 Autonomia e aprovação (AUT)

| Nível | Tipo de ação | Exemplos | Regra |
|---|---|---|---|
| **N0** | Leitura | Consultar agenda, buscar documento | Automático |
| **N1** | Proposta | Rascunhar e-mail, sugerir reunião | Automático, nada sai sem confirmação |
| **N2** | Ação reversível e interna | Criar tarefa, mover evento próprio | Automático com notificação e desfazer |
| **N3** | Ação externa, irreversível, financeira ou destrutiva | Enviar e-mail, aceitar convite de terceiros, pagar, apagar, acionar deploy | **Aprovação humana explícita** a cada vez |

- **AUT-01** O nível de cada ação é decidido por política (OPA), não pelo modelo.
- **AUT-02** O usuário pode **reduzir** o nível de autonomia, e só um administrador pode elevá-lo.
- **AUT-03** Fila de aprovações com resumo claro do que será feito, com quais dados e para quem.
- **AUT-04** Botão de **parada de emergência** que suspende todas as ações do agente.

### 6.7 Segurança do agente (SEG)
- **SEG-01** Todo conteúdo externo (e-mail, web, anexos, mensagens) é tratado como **dado não confiável**, nunca como instrução.
- **SEG-02** **Regra dos três fatores:** o agente não deve combinar, na mesma execução, acesso a dados privados, leitura de conteúdo não confiável e capacidade de comunicação externa sem aprovação.
- **SEG-03** Ferramentas com permissão mínima, por tarefa, com credencial efêmera.
- **SEG-04** Filtros de saída para evitar vazamento de dados sensíveis em respostas ou chamadas de ferramenta.
- **SEG-05** Suíte de testes adversariais (injection direta e indireta, exfiltração) obrigatória em cada release.
- **SEG-06** Ferramentas e plugins de terceiros só entram por lista aprovada, com versão fixa e revisão.

### 6.8 Auditoria (AUD)
- **AUD-01** Registro de entradas, fontes consultadas, ferramentas chamadas, decisões, aprovações e resultados.
- **AUD-02** O usuário acessa o próprio histórico; o administrador acessa apenas metadados, nunca o conteúdo pessoal, salvo política explícita.
- **AUD-03** Logs enviados ao `aquila-oculus` e a cópia imutável.

### 6.9 API (API)
- **API-01** API REST/OpenAPI e eventos para outras VMs consultarem o Advisor e receberem notificações.
- **API-02** Autenticação por token OIDC com escopo por usuário e por tenant.
- **API-03** Modo standalone sem dependência das demais VMs.

## 7. Requisitos não funcionais

| Área | Requisito |
|---|---|
| **Privacidade (LGPD)** | Dados pessoais minimizados, base legal e finalidade documentadas, direito de acesso, correção e exclusão atendidos pelo painel |
| **Segurança** | SSO + MFA, criptografia em trânsito e em repouso (volume por usuário), segredos no `aquila-sentinel`, isolamento entre usuários |
| **Desempenho** | Resposta conversacional < 5 s (p95) para consultas simples; briefing pronto antes do horário configurado |
| **Disponibilidade** | Meta de 99,5% na v1; falha do Advisor não pode afetar as demais VMs |
| **Custo** | Orçamento de tokens por usuário, cache de contexto e modelos menores para tarefas simples |
| **Qualidade** | Conjunto de avaliação com tarefas reais e casos adversariais; release bloqueado em regressão |
| **Recuperação** | Backup do banco e dos volumes em `taurus-hub`, RPO ≤ 24 h, RTO ≤ 8 h |
| **Observabilidade** | Métricas de latência, custo, taxa de aprovação/recusa e alertas de comportamento anômalo |
| **Portabilidade** | Entrega reprodutível (imagem + repositório) e instalação standalone documentada |
| **Licenciamento** | Registrar licença de cada componente e de cada modelo antes da comercialização |

## 8. Integrações com outras VMs

| VM | Direção | Integração |
|---|---|---|
| `aquila-sentinel` | Advisor → Sentinel | SSO, MFA, tenants, **cofre de tokens OAuth** e credenciais efêmeras |
| `aquila-oculus` | Oculus → Advisor / Advisor → Oculus | Recebe alertas para o briefing; envia logs de auditoria (somente envio) |
| `aquila-predict` | Predict → Advisor | Previsões e anomalias incorporadas ao briefing |
| `aquila-recon` | Recon → Advisor | Inteligência de ameaças e superfície de ataque resumida para o usuário |
| `homo-rag` | Advisor → RAG | Base de conhecimento corporativa para perguntas e respostas |
| `homo-lex` | Advisor → Lex | Perguntas jurídicas encaminhadas com as citações preservadas |
| `homo-bi` | BI → Advisor | Indicadores para acompanhamento e alertas de métricas |
| `homo-crm` | Advisor ↔ CRM | Contexto de clientes e próximos passos de relacionamento |
| `homo-ethic` | Ethic ↔ Advisor | Políticas de autonomia, auditoria e revisão de conformidade |
| `leo-logistics` | Advisor → Event bus | Canal oficial para ordens de execução |
| `leo-media`, `leo-marketing`, `leo-devops` | Advisor → (via bus) | Execução delegada, sempre com aprovação conforme o nível |
| `leo-compute` | Advisor → Compute | Inferência de LLM e voz em GPU |
| `taurus-hub` | Advisor → Storage | Documentos pessoais e backups |
| `taurus-portal` | Portal → Advisor | Contratação, criação de usuários e painel do cliente |

## 9. Modo standalone (produto)
- Instalação independente, com SSO local opcional (Keycloak embutido ou login simples com MFA).
- Conectores pessoais prontos (Google, Microsoft, CalDAV/IMAP) e catálogo de ferramentas aprovadas.
- Assistente de configuração: modelo (local ou externo), canais, horário do briefing, níveis de autonomia, política de retenção.
- Documentação para o usuário final e para o administrador.

## 10. Riscos

| Risco | Impacto | Mitigação |
|---|---|---|
| **Prompt injection indireto** (instrução escondida em e-mail, página ou anexo) | Crítico | Conteúdo externo como dado, regra dos três fatores, aprovação para ações externas, testes adversariais |
| **Vazamento de dados pessoais** | Crítico | Isolamento por usuário, credenciais efêmeras, filtros de saída, auditoria |
| **Ação irreversível indevida** | Crítico | Níveis de autonomia por política, aprovação N3, botão de emergência |
| **Tokens OAuth e acesso a e-mail como alvo de alto valor** | Alto | Cofre no Sentinel, escopos mínimos, rotação, alertas de uso anômalo |
| **Extensões e plugins de terceiros maliciosos** | Alto | Lista aprovada, versão fixa, revisão, execução em sandbox |
| **Gateway de canais exposto na internet** | Alto | Ligar ao loopback/rede privada, acesso por VPN/overlay, allowlist de remetentes |
| **Alucinação ou erro com ar de certeza** | Médio | Citação de fontes, distinção entre fato e inferência, confirmação antes de agir |
| **Fadiga de alertas e baixa utilidade** | Médio | Regras de relevância, limites, feedback do usuário, métricas de utilidade |
| **Custo de inferência** | Médio | Orçamento por usuário, cache, modelos menores para tarefas simples |
| **Regras do WhatsApp e de APIs de terceiros** | Médio | Usar APIs oficiais, evitar automação não suportada |
| **Conflito com os demais "advisors"** (`homo-lex`, `homo-arch`, `leo-ea`) | Baixo | Definir o Advisor como interface pessoal e roteador; especialistas respondem sobre seu domínio |

## 11. Repositórios de referência (projetos existentes)

> **Verifique licença, manutenção e versão atual de cada projeto antes de adotar.** Projetos de assistentes de IA mudam rápido e vários têm histórico recente de falhas de segurança.

### Assistentes pessoais completos
- **OpenClaw**: https://github.com/openclaw/openclaw (assistente pessoal self-hosted, multicanal, com gateway, memória e skills; licença MIT. Útil como referência de arquitetura. Por ser um projeto de uso pessoal e de evolução rápida, avalie com rigor o modelo de segurança antes de usá-lo como base de produto)
- **Khoj**: https://github.com/khoj-ai/khoj ("segundo cérebro" de IA sobre notas e documentos pessoais; verifique a licença)
- **Open WebUI**: https://github.com/open-webui/open-webui (interface de chat para modelos locais e externos)
- **AnythingLLM**: https://github.com/Mintplex-Labs/anything-llm
- **LibreChat**: https://github.com/danny-avila/LibreChat
- **Home Assistant (Assist)**: https://github.com/home-assistant/core (referência de assistente de voz local e automação)

### Memória e agentes
- **Mem0**: https://github.com/mem0ai/mem0 (camada de memória para agentes)
- **Letta (MemGPT)**: https://github.com/letta-ai/letta (agentes com memória gerenciada)
- **Graphiti**: https://github.com/getzep/graphiti (grafo de conhecimento temporal para memória)
- **LangGraph**: https://github.com/langchain-ai/langgraph (orquestração de agentes com estado)
- **CrewAI**: https://github.com/crewAIInc/crewAI
- **Model Context Protocol (servidores de referência)**: https://github.com/modelcontextprotocol/servers

### Automação e conectores
- **n8n**: https://github.com/n8n-io/n8n (licença fair-code; verifique o uso comercial)
- **Activepieces**: https://github.com/activepieces/activepieces
- **Windmill**: https://github.com/windmill-labs/windmill

### Dados pessoais self-hosted
- **Radicale** (agenda e contatos): https://github.com/Kozea/Radicale
- **Nextcloud**: https://github.com/nextcloud/server
- **Vikunja** (tarefas): https://github.com/go-vikunja/vikunja

### Voz
- **faster-whisper**: https://github.com/SYSTRAN/faster-whisper
- **Piper TTS**: https://github.com/rhasspy/piper (confirme o repositório mantido atualmente)

### Segurança e qualidade
- **LLM Guard**: https://github.com/protectai/llm-guard
- **NeMo Guardrails**: https://github.com/NVIDIA/NeMo-Guardrails
- **Garak** (varredura de vulnerabilidades em LLMs): https://github.com/NVIDIA/garak
- **promptfoo**: https://github.com/promptfoo/promptfoo
- **Langfuse**: https://github.com/langfuse/langfuse
- **Open Policy Agent**: https://github.com/open-policy-agent/opa

## 12. Roadmap de construção

| Fase | Entregas | Critério de aceite |
|---|---|---|
| **0. Base** | Ubuntu 24.04 mínimo (seed), Compose, PostgreSQL + pgvector, SSO via Sentinel, modelo local ou externo configurado | Login com MFA; chat web funcionando |
| **1. Conversa e memória** | Chat web, memória em camadas, painel de memória, trilha de auditoria | Usuário vê, edita e apaga memórias; apagamento verificado |
| **2. Conectores de leitura** | Agenda, e-mail e tarefas em modo somente leitura (N0), tokens no cofre | Resumo correto da agenda e dos e-mails em teste |
| **3. Briefing e alertas** | Briefing diário, alertas proativos, controle de fadiga | Briefing avaliado como útil no piloto |
| **4. Ações com aprovação** | Rascunhos (N1), ações internas (N2), fila de aprovação e N3, botão de emergência, política OPA | Nenhuma ação externa sem aprovação em testes |
| **5. Segurança adversarial** | Suíte de injection e exfiltração, filtros de entrada e saída, isolamento por usuário | Suíte aprovada; nenhum vazamento entre usuários |
| **6. Delegação** | Consultas a RAG, Lex, BI, CRM, Predict, Recon; ordens via event bus | Fluxo ponta a ponta: pergunta → consulta → resposta com fonte |
| **7. Canais e voz** | Telegram/WhatsApp oficial, voz local, allowlist | Mensagens de remetentes não autorizados ignoradas |
| **8. Standalone** | Instalador, assistente de configuração, documentação | Instalação limpa em até 1 dia |

## 13. Estrutura sugerida do repositório

```
aquila-advisor/
├── README.md
├── PRD.md                      # este arquivo
├── docs/
│   ├── adr/                    # decisões de arquitetura
│   ├── modelo-de-ameacas.md
│   ├── niveis-de-autonomia.md
│   └── runbooks/               # parada de emergência, restauração, rotação de tokens
├── ansible/                    # host, hardening, containers
├── compose/                    # postgres, llm, voz, gateway, api, ui
├── app/
│   ├── api/
│   ├── agent/                  # orquestração, planejamento
│   ├── memory/                 # camadas, painel, esquecimento
│   ├── tools/                  # conectores MCP (agenda, e-mail, tarefas...)
│   ├── approvals/              # fila e níveis de autonomia
│   ├── guardrails/             # entrada, saída, injection
│   └── briefing/
├── channels/                   # web, telegram, whatsapp, voz
├── policies/                   # OPA/Rego (níveis N0 a N3, escopos por ferramenta)
├── eval/
│   ├── tasks/                  # tarefas reais
│   └── adversarial/            # injection, exfiltração
├── tests/
└── .gitignore                  # nunca versionar segredos, tokens, memórias ou dados pessoais
```

**Regras do repositório:** nenhum segredo, token ou dado pessoal versionado; mudanças em `app/guardrails`, `app/approvals` e `policies/` exigem revisão obrigatória; release bloqueado se a suíte adversarial regredir; plugins de terceiros com versão e digest fixos.

## 14. Questões em aberto

- **Q1.** O Advisor é **assistente pessoal/executivo** (premissa deste PRD) ou também **analista e relator de incidentes de segurança**, como sugeri antes? Se o segundo, isso vira um módulo opcional separado.
- **Q2.** Modelo de implantação: **um usuário por instância** (mais seguro e simples) ou **multiusuário** na mesma VM com isolamento?
- **Q3.** Canais prioritários: web, Telegram, WhatsApp, voz? O WhatsApp exige conta e API oficiais.
- **Q4.** Provedores de agenda e e-mail dos clientes: Google, Microsoft ou self-hosted?
- **Q5.** LLM local obrigatório, ou API externa permitida por política do usuário?
- **Q6.** Haverá ações financeiras no futuro? Se sim, exigem fluxo próprio de aprovação forte.
- **Q7.** Como o Advisor se relaciona com `homo-lex`, `homo-arch` e `leo-ea`? Proposta: ele roteia a pergunta e apresenta a resposta do especialista.
- **Q8.** O monitoramento do uso pelo administrador (quanto e o quê) é aceitável para o cliente, dentro da LGPD?
- **Q9.** O Advisor pode falar em nome do usuário (e-mails, mensagens)? Em quais condições e com qual identificação?

## 15. Definição de pronto (v1)

- [ ] Fases 0 a 5 concluídas e testadas
- [ ] Nenhuma ação N3 executada sem aprovação em todos os testes
- [ ] Suíte adversarial (injection direta e indireta, exfiltração) aprovada
- [ ] Painel de memória permite ver, editar e apagar, com exclusão comprovada
- [ ] Tokens OAuth guardados no `aquila-sentinel`; nenhum segredo no repositório
- [ ] Isolamento entre usuários comprovado por teste
- [ ] Logs e auditoria chegando ao Oculus e à cópia imutável
- [ ] Botão de parada de emergência testado
- [ ] Política de privacidade, termos de uso e bases legais (LGPD) publicados
- [ ] Backup e restauração testados
- [ ] Runbooks escritos

---
_Documento vivo: atualize a versão e registre decisões em `docs/adr/` a cada mudança relevante._
