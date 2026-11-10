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
- **Não** implementa o encoder registry nem mapeia formato para codificador: o registry e a orquestração do lote são `TASK-E08-02`, no mesmo módulo, que bloqueia esta TASK e já declara que os codificadores saem daqui. O título desta TASK diz lib de encoders, então escrever o registry junto é a confusão mais provável; `vynamp-encode` publica o codificador, o registry de `TASK-E08-02` escolhe qual atende cada formato.
- **Não** decide licença, inventário de dependências nem política de distribuição de codec. Isso é `TASK-E11-03`; a ressalva de patente do AAC já está aceita e registrada na spec, e reabri-la na implementação seria decisão jurídica tomada no código. Também não cria nem dota de governança os repositórios `vynamp-converter` e `vynamp-encode`: eles já existem públicos com README, LICENSE e SECURITY.md, entregues por `TASK-E19-04`.
- **Não** restaura áudio nem faz declip, denoise ou qualquer processamento de reparo. O restaurador é `TASK-E18-01`; o critério AC-1 existe justamente porque converter para lossy parece correção de qualidade e não é.
- **Não** edita tags, organiza pastas nem faz dedupe. É `TASK-E08-04` no player; aqui só se aplica o template de nome com whitelist e a sanitização, que reusa a regra de lá sem alterá-la.

## Antes de começar

- [ ] Spec do módulo aprovada e o critério de aceite citado já existe nela
- [ ] Worktree própria criada (`cargo xtask repo worktree task-e18-02-claude-<slug>`)
- [ ] Fixtures/hardware necessários disponíveis ou limitação registrada
- [ ] Nenhuma TASK em execução tocando os mesmos arquivos

## Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| `TASK-E18-02-AC-1` | o fixture sine-24-stereo.flac (48 kHz, 24 bits, 2 canais, 4800 frames, 0,1 s, 21109 bytes), destino num diretório vazio do mesmo volume e a invocação vynamp-converter --format opus --bitrate 96 sem --overwrite | o comando roda até o job sair, e em seguida o harness le por inteiro o relatório gravado pela conversão e todos os textos de interface que o job exibiu | Conversão lossy nunca é anunciada como restauração hi-res |
| `TASK-E18-02-AC-2` | o mesmo sine-24-stereo.flac como entrada e o plano com --format flac --bit-depth 16 --sample-rate 44100, ou seja, 24 para 16 bits e 48 kHz para 44,1 kHz | o usuário dispara o job com esse plano, responde a um pedido explícito de confirmação de degradação, o job termina e o harness abre o relatório gerado | degradação de profundidade ou taxa exige confirmação e aparece no relatório |
| `TASK-E18-02-AC-3` | duas cópias de sine-24-stereo.flac (4800 frames, 2 canais) marcadas somente-leitura na origem, destino em diretório vazio do mesmo volume, o cancelamento por arquivo exposto pela fila de `TASK-E08-01` e um `Write` de teste que devolve `io::ErrorKind::StorageFull` na metade da escrita do segundo temporário | o job 1 falha com o StorageFull injetado no meio da escrita do temporário, o job 2 e cancelado pela fila no meio da codificação, e depois o harness lista o diretorio de destino e recalcula SHA-256 e atributos de cada origem | cancelamento e disco cheio preservam a source |
| `TASK-E18-02-AC-4` | um job --format flac sobre sine-24-stereo.flac cujo temporário de destino já foi escrito, sincronizado com fsync e ainda não renomeado sobre o destino | o harness lê o temporário antes do rename e aciona a validação pós-escrita sobre ele, repetindo com uma variante em que o temporário é truncado de propósito antes da releitura | saída é validada após escrita |
| `TASK-E18-02-AC-5` | o diretorio de destino já contém sine-24-stereo.flac, de 21109 bytes, com conteúdo distinto do que o job produziria, e o job aponta para esse mesmo nome sem --overwrite | o lote de dez arquivos que inclui esse nome é enfileirado e disparado, e o harness observa o destino no instante em que a colisão é detectada, antes de qualquer escrita | colisão de nome não sobrescreve. |

## Verificação

| Item | Comando / fixture | Evidência esperada |
|---|---|---|
| Teste automatizado | `cargo test` no repo `vynamp-converter` | verde no CI Gate |
| Oráculo independente | `cargo test` no repo `apps/converter` com os fixtures `sine-24-stereo.flac` (48 kHz, 24 bits, 4800 frames, 21109 bytes), `sine-16-stereo.flac` e `sine-16-mono-tail.flac`, mais um `--dry-run` e um par de arquivos homônimos no destino; a source é hasheada com SHA-256 antes e depois de cada job | o relatório de cada job traz origem e destino com taxa e profundidade, rotula o job Opus como lossy sem a expressão restauração hi-res, lista 24 para 16 bits e 48 para 44,1 kHz como degradações confirmadas, remove o temporário truncado e deixa o SHA-256 da origem idêntico; a falha de disco entra por um `Write` que devolve `StorageFull`, e não por um volume cheio de verdade, porque encher um volume não é portátil numa suíte de teste; hoje `apps/converter/src/main.rs` tem 3 linhas com um `println!` de versão e `apps/converter/Cargo.toml` declara só `[[bin]]`, sem `[lib]`, de modo que um teste em `tests/` nem conseguiria importar a API; não há pasta `tests/` em `apps/converter` nem em `libs/encode`, e `libs/encode/src/lib.rs` só declara `VERSION` — nenhuma dessas evidências existe ainda |
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
