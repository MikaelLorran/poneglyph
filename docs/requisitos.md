# Poneglyph — Especificação original (Fase 3)

> Documento fornecido pelo mantenedor em 2026-11-06. Mantido como referência; as decisões tomadas
> depois dele estão em `docs/superpowers/specs/2026-11-07-poneglyph-design.md`, que prevalece em caso
> de conflito.

## 1. Visão Geral e Escopo Acordado

O Poneglyph é um harness e orquestrador open-source de agentes de Inteligência Artificial voltado para engenharia de software autônoma e assistida, inspirado em ferramentas como Claude Code e Alethe Agents. Ele resolve o problema da perda de contexto (context pollution), falta de determinismo em tarefas longas de codificação e ausência de governança financeira na execução de múltiplos modelos.

Justificativa de Escopo e Prazo (Acordo da Fase 3):
Considerando que o desenvolvimento será conduzido por um único desenvolvedor (1 Dev Full Stack) em paralelo a outras demandas e com meta inicial de entrega funcional até Dezembro de 2026 (~8 semanas), o escopo completo foi fatiado em 4 Milestones evolutivas.

- **MVP 1.0 (Até Dez/2026 - Milestones 1 e 2):** Focado exclusivamente na construção do pacote @poneglyph/core e da interface de linha de comando (@poneglyph/cli), integrando banco SQLite via Prisma, containerização Docker para sandbox, servidores MCP nativos (filesystem, bash, git), hierarquia de agentes consumindo APIs de prateleira (OpenAI/Anthropic/OpenRouter) e pipeline de validação via LLM-as-a-Judge.
- **Pós-MVP / Pesquisa (2027 - Milestones 3 e 4):** Construção da interface Desktop em Electron + React, implementação da camada semântica de versionamento (AI-Git) e treinamento/hospedagem de modelos próprios com Fine-Tuning (LoRA) em infraestrutura Serverless GPU (vLLM/Modal/RunPod).

### Roadmap de Milestones

**Milestone 1: Fundação, Persistência e Sandbox (Semanas 1 a 4 — Nov/2026)**
- Setup do monorepo (pnpm workspaces + Turborepo + tsup + Vitest).
- Modelagem do SQLite com Prisma (Sessões, Traces de Telemetria, Custos) e implementação do Circuit Breaker financeiro.
- Gerenciador de Containers Docker (criação, standby e rotina de limpeza prune) integrado aos servidores MCP (filesystem, bash, git).

**Milestone 2: Motor de Orquestração e CLI MVP 1.0 (Semanas 5 a 8 — Dez/2026)**
- Implementação da hierarquia em 3 níveis (Orquestrador, Especialistas, Assistentes) com empacotamento via Prompt Estruturado.
- Pipeline LLM-as-a-Judge acoplado ao resultado de testes (TDD) + Agente Especialista de Troubleshooting com loop max_retries.
- CLI interativa (init, task, chat, resume, metrics) com fallback de reidratação de estado e publicação no NPM.

**Milestone 3: Interface Desktop Electron & Chat Remoto (Q1 2027)**
- Aplicação Desktop (@poneglyph/desktop) em Electron + React + Vite + Tailwind CSS (Dark Mode, paleta Dark Slate #0F172A, Space Grotesk e Inter), consumindo o @poneglyph/core via IPC.
- Interface Web/Chat remoto conectado ao endpoint de inferência.

**Milestone 4: AI-Git & Fine-Tuning Serverless GPU (Q2 2027 — Tese)**
- Desenvolvimento da abstração semântica AI-Git (substituindo comandos Git tradicionais por payloads estruturados para agentes).
- Criação do dataset e Fine-Tuning (LoRA) em modelos abertos de código (ex: Qwen2.5-Coder / DeepSeek), hospedados sob demanda via Serverless GPU com vLLM.

## 2. Requisitos Funcionais (RFs)

- **RF-01 (Configuração de Intervenção):** Como usuário, eu quero configurar o nível de autonomia do harness (execução automática contínua vs. aprovação manual Y/n antes de ações críticas) para ter controle total sobre a segurança das operações no terminal e sistema de arquivos.
- **RF-02 (Gestão de Perfis de Agentes):** Como usuário, eu quero cadastrar e editar perfis de agentes no arquivo de configuração, definindo nome, camada hierárquica, endpoint da API, modelo e system prompt para personalizar minha equipe de agentes.
- **RF-03 (Orquestração Autônoma e Manual):** Como usuário, eu quero alternar entre o modo autônomo (onde o Agente Orquestrador decompõe a tarefa e aciona subagentes) e o modo manual (onde seleciono diretamente qual agente responderá ao prompt) para adaptar a ferramenta à complexidade de cada problema.
- **RF-04 (Hierarquia em Três Camadas):** Como Orquestrador do sistema, eu quero gerenciar agentes divididos em três níveis — Orquestrador (alto raciocínio), Especialistas (foco em domínio/fine-tuning) e Assistentes (modelos rápidos/baratos para sub-rotinas) — permitindo que tanto o Orquestrador quanto os Especialistas acionem Assistentes.
- **RF-05 (Delegação por Prompt Estruturado):** Como Orquestrador, eu quero enviar tarefas aos subagentes encapsuladas em um protocolo estrito de Prompt Estruturado (contendo apenas objetivo, contexto isolado da subtarefa, regras de SDD/TDD e schema de saída) para evitar poluição de contexto e economizar tokens.
- **RF-06 (Sinalização e Avaliação LLM-as-a-Judge):** Como subagente, eu quero sinalizar formalmente a conclusão da minha tarefa para que o Orquestrador acione o LLM-as-a-Judge, que validará o código gerado cruzando a especificação inicial com o stdout/stderr dos testes automatizados, retornando Pass/Fail e o motivo detalhado.
- **RF-07 (Troubleshooting e Reconstrução de Tarefa):** Como Orquestrador, ao receber um veredito de falha (Fail) do Juiz, eu quero acionar automaticamente um Agente Especialista de Investigação padrão para analisar o código, o erro dos testes e o parecer do Juiz, gerando uma nova tarefa estruturada corrigida até o limite de max_retries.
- **RF-08 (Execução em Sandbox Docker via MCP):** Como sistema, eu quero executar todas as ferramentas do Model Context Protocol (filesystem, bash e git) dentro de um container Docker isolado para proteger o sistema operacional host contra comandos destrutivos.
- **RF-09 (Persistência de Sessão, Standby e Fallback):** Como usuário, eu quero pausar ou retomar (poneglyph resume <id>) qualquer sessão de trabalho, reconectando ao container Docker mantido em standby ou, caso o container não exista mais, reconstruindo automaticamente o estado do projeto a partir do histórico de prompts e commits registrados no SQLite.
- **RF-10 (Observabilidade e Circuit Breaker):** Como usuário, eu quero visualizar métricas detalhadas (poneglyph metrics) de latência, tokens e custos armazenados no SQLite, além de definir travas financeiras (por tarefa, por sessão e mensal) que interrompam imediatamente a execução caso o orçamento seja atingido.
- **RF-11 (Camada de Abstração AI-Git — [Milestone 4 / Pendência de Design]):** Como agente de IA, eu quero versionar minhas alterações através de uma interface semântica simplificada (AI-Git) que registre o diff do código junto ao raciocínio utilizado, sem precisar manipular comandos Git complexos diretamente no terminal.
- **RF-12 (Interface Gráfica Desktop — [Milestone 3]):** Como usuário, eu quero operar todas as funcionalidades de orquestração, visualização de árvores de agentes, diffs de código e métricas através de uma aplicação Desktop em Electron.

## 3. Requisitos Não Funcionais (RNFs)

- **RNF-01 (Arquitetura e Stack Tecnológica):** O projeto deve ser estruturado em um Monorepo TypeScript (Node.js LTS) utilizando pnpm workspaces e Turborepo, dividido em pacotes desacoplados: @poneglyph/core (motor, Prisma/SQLite, cliente MCP), @poneglyph/cli (interface de terminal com commander e @clack/prompts, empacotada via tsup) e futuramente @poneglyph/desktop (Electron + React + Vite + Tailwind CSS).
- **RNF-02 (Compatibilidade de Inferência):** O módulo de conexão com LLMs deve seguir estritamente o contrato da API da OpenAI (SDK oficial/compatível), permitindo apontar de forma transparente tanto para provedores comerciais quanto para motores de inferência open-source (vLLM / TGI / Ollama) rodando em Serverless GPU.
- **RNF-03 (Segurança de Credenciais e Governança de Repositório):** Chaves de API nunca devem ser salvas em texto puro dentro do diretório do repositório alvo; devem utilizar variáveis de ambiente locais ou configuração global segura do usuário. O repositório público no GitHub deve operar com Branch Protection Rules na main, exigindo Pull Request, aprovação do mantenedor e sucesso na pipeline de CI (Vitest + Build) antes do merge.
- **RNF-04 (Gestão de Recursos Host):** O sistema deve prover um comando de limpeza (poneglyph prune) e verificação de saúde para encerrar containers Docker em standby órfãos ou expirados, evitando vazamento de memória RAM e disco na máquina local.
- **RNF-05 (Padronização Visual Desktop — [Milestone 3]):** A interface Electron deverá operar nativamente em Dark Mode com alto contraste (fundo base #0F172A), utilizando a fonte Space Grotesk para cabeçalhos, Inter para logs/corpo de texto e JetBrains Mono (ou equivalente monoespaçada) para blocos de código e terminal integrado.

## 4. Regras de Negócio (RNs)

- **RN-01 (Obrigatoriedade de SDD e TDD):** Nenhuma alteração de código produzida por Especialistas ou Assistentes pode ser considerada concluída pelo LLM-as-a-Judge apenas por inspeção visual/estática do LLM; o veredito de aprovação exige que a suíte de testes correspondente seja executada via MCP bash dentro do sandbox e retorne código de saída 0 (sucesso).
- **RN-02 (Atomicidade de Versionamento por Agente):** Cada intervenção validada de um agente ou subagente no código deve gerar um registro/commit atômico identificando nos metadados qual perfil de agente realizou a ação, o ID da tarefa e o custo em tokens daquela operação.
- **RN-03 (Precedência do Circuit Breaker):** As travas de orçamento financeiro (limite por tarefa, limite por sessão e teto mensal) têm precedência absoluta sobre o loop de max_retries. Se uma tentativa de refatoração atingir 100% do teto configurado, o Orquestrador deve abortar a chamada de API imediatamente, colocar o container Docker em standby, salvar o estado no SQLite e devolver o controle ao usuário.
- **RN-04 (Isolamento Estrito de Contexto):** Subagentes (Especialistas e Assistentes) são proibidos de acessar o histórico completo do chat do usuário no SQLite; eles recebem única e exclusivamente o payload do Prompt Estruturado montado pelo agente superior que os invocou.
