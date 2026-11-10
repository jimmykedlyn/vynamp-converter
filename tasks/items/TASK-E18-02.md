# TASK-E18-02 — Entregar Vynamp Converter e a lib de encoders

> Gerado pelo passo 37 a partir de `tasks/backlog.json`. Os campos de estado vêm do
> backlog e não devem ser editados aqui. A coluna `Then` é o requisito aprovado,
> copiado literal do backlog; `Given` e `When` foram autorados na execução.

| Campo | Valor |
|---|---|
| ID | `TASK-E18-02` |
| Módulo dono | `vynamp-converter` |
| Repo | `jimmykedlyn/vynamp-converter` |
| REQ coberto | `REQ-E18-02` |
| Critérios da spec | pendente — `specs/converter.md` §5 |
| Fase | M3 |
| Prioridade | P0 |
| Estado | planned |
| Depende de | `TASK-E08-01`, `TASK-E01-07`, `TASK-E08-02`, `TASK-E11-03` |
| Bloqueia | `TASK-E19-04` |
| Modelo executor | Opus — áudio/RT, concorrência, segurança ou release |
| Esforço | 15–35 dias úteis |
| Risco | alto |

## Entrega

Ferramenta desktop `vynamp-converter` com fila de jobs persistida, apoiada na lib própria `vynamp-encode` como única fonte de codificadores.

## Boundary

- **Não** implementa o scheduler de jobs com checkpoints, leases e retomada pós-crash. Esse motor é `TASK-E08-01` em `vynamp-engine`; aqui a fila apenas consome a API dele, e duplicar a persistência faria dois donos do mesmo journal de fila.
- **Não** escreve os adapters de decodificação de AAC-LC, HE-AAC, ALAC, Vorbis e Opus. São de `TASK-E01-07` em `vynamp-decode`; a confusão é plausível porque converter para esses formatos exige ler os mesmos contêineres, mas ler é decode e aqui só se orquestra.
- **Não** decide licença, inventário de dependências nem política de distribuição de codec. Isso é `TASK-E11-03`; a ressalva de patente do AAC já está aceita e registrada na spec, e reabri-la na implementação seria decisão jurídica tomada no código.
- **Não** restaura áudio nem faz declip, denoise ou qualquer processamento de reparo. O restaurador é `TASK-E18-01`; o critério AC-1 existe justamente porque converter para lossy parece correção de qualidade e não é.
- **Não** edita tags, organiza pastas nem faz dedupe. É `TASK-E08-04` no player; aqui só se aplica o template de nome com whitelist e a sanitização, que reusa a regra de lá sem alterá-la.
- **Não** cria nem dota de governança os repositórios `vynamp-converter` e `vynamp-encode`. Isso foi `TASK-E19-04`; os repos já existem públicos com README e licença, e o trabalho aqui é código e fila, não infraestrutura de repo.

## Antes de começar

- [ ] Spec do módulo aprovada e o critério de aceite citado já existe nela
- [ ] Worktree própria criada (`cargo xtask repo worktree task-e18-02-claude-<slug>`)
- [ ] Fixtures/hardware necessários disponíveis ou limitação registrada
- [ ] Nenhuma TASK em execução tocando os mesmos arquivos

## Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| `TASK-E18-02-AC-1` | o fixture sine-24-stereo.flac (48 kHz, 24 bits, 2 canais, 4800 frames, 0,1 s, 21109 bytes) e a invocação vynamp-converter --format opus --bitrate 96 sem --overwrite | o job termina com exit 0 e tanto o relatório da conversão quanto os textos da interface do job são lidos por inteiro | Conversão lossy nunca é anunciada como restauração hi-res |
| `TASK-E18-02-AC-2` | o mesmo sine-24-stereo.flac como entrada e o plano com --format flac --bit-depth 16 --sample-rate 44100, ou seja, 24 para 16 bits e 48 kHz para 44,1 kHz | o plano é exibido antes de qualquer escrita, a confirmação explícita de degradação é exigida para o job seguir, o job termina e o relatório é aberto | degradação de profundidade ou taxa exige confirmação e aparece no relatório |
| `TASK-E18-02-AC-3` | duas cópias de sine-24-stereo.flac marcadas somente-leitura na origem, com o destino num volume cujo espaço livre termina durante a escrita do temporário de WAV 32 bits (cerca de 1,5 MiB por cópia), e cancelamento por arquivo disponível na fila | a escrita falha por disco cheio num job e o usuário cancela o outro job no meio da codificação | cancelamento e disco cheio preservam a source |
| `TASK-E18-02-AC-4` | um job --format flac sobre sine-24-stereo.flac cujo temporário já foi escrito, sincronizado com fsync e ainda não renomeado sobre o destino | a etapa de validação pós-escrita relê e decodifica o temporário e confronta contêiner, taxa, canais e duração com o plano antes do rename, com uma variante em que o temporário é truncado de propósito | saída é validada após escrita |
| `TASK-E18-02-AC-5` | o diretório de destino já contém sine-24-stereo.flac, de 21109 bytes, e um conteúdo distinto do que o job produziria, e o job aponta para esse mesmo nome sem --overwrite | o job é enfileirado com dez arquivos e chega à detecção de colisão antes da escrita do destino | colisão de nome não sobrescreve. |

## Verificação

| Item | Comando / fixture | Evidência esperada |
|---|---|---|
| Teste automatizado | `cargo test` no repo `vynamp-converter` | verde no CI Gate |
| Oráculo independente | `cargo test` no repo `apps/converter` com os fixtures `sine-24-stereo.flac` (48 kHz, 24 bits, 4800 frames, 21109 bytes), `sine-16-stereo.flac` e `sine-16-mono-tail.flac`, mais um `--dry-run` e um par de arquivos homônimos no destino; a source é hasheada com SHA-256 antes e depois de cada job | o relatório de cada job traz origem e destino com taxa e profundidade, rotula o job Opus como lossy sem a expressão restauração hi-res, lista 24 para 16 bits e 48 para 44,1 kHz como degradações confirmadas, remove o temporário truncado e deixa o SHA-256 da origem idêntico; hoje `apps/converter/src/main.rs` tem 3 linhas com um `println!` de versão, não há pasta `tests/`, e `libs/encode/src/lib.rs` só declara `VERSION` — nenhuma dessas evidências existe ainda |
| Hardware | nenhum: os jobs leem e escrevem arquivos de fixture e não abrem dispositivo de áudio; a ausência neste ambiente de placa pro, Android, iOS e macOS não afeta nenhum dos cinco critérios | ausência é limitação registrada, não aprovação |

## DoD — pronto para review

- [ ] Todos os critérios de aceite com evidência
- [ ] `cargo fmt --check` e `cargo clippy --all-targets -- -D warnings` limpos
- [ ] `cargo test` verde; testes ignorados listados com motivo
- [ ] Spec do módulo atualizada se a implementação revelou algo novo
- [ ] `backlog.json` com o estado correto
- [ ] Revisão feita por contexto limpo, não pelo implementador
- [ ] PR aberto com template preenchido; CI Gate verde no SHA final
## Notas de execução

Preenchido durante a execução, não antes.
