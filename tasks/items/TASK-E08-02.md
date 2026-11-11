# TASK-E08-02 — Entregar conversor e encoder registry

> Gerado pelo passo 37 a partir de `tasks/backlog.json`. Os campos de estado vêm do
> backlog e não devem ser editados aqui. A coluna `Then` é o requisito aprovado,
> copiado literal do backlog; `Given` e `When` foram autorados na execução.

| Campo | Valor |
|---|---|
| ID | `TASK-E08-02` |
| Módulo dono | `vynamp-converter` |
| Repo | `jimmykedlyn/vynamp-converter` |
| REQ coberto | `REQ-E08-02` |
| Critérios da spec | pendente — `specs/converter.md` §5 |
| Fase | M3 |
| Prioridade | P0 |
| Estado | planned |
| Depende de | `TASK-E08-01`, `TASK-E01-07`, `TASK-E11-03` |
| Bloqueia | `TASK-E05-08`, `TASK-E12-06`, `TASK-E18-02` |
| Modelo executor | Opus — áudio/RT, concorrência, segurança ou release |
| Esforço | 5–15 dias úteis |
| Risco | médio |

## Entrega

Conversão batch para FLAC/WAV/Opus/Vorbis e MP3 quando permitido, com preview.

## Boundary

- **Não** implementa codificador nenhum: FLAC, WAV, Opus, Vorbis e MP3 saem da lib `vynamp-encode`, entregue por `TASK-E18-02`, no mesmo módulo, e hoje reduzida a um stub de 6 linhas com `pub const VERSION`. Aqui se entrega o registry que escolhe o encoder, a orquestração do lote e a validação pós-escrita; escrever frame FLAC neste app faria dois donos do formato e divergiria do muxing de priming e padding, que é do encoder.
- **Não** implementa a fila persistida com checkpoints, leases, progresso e pools limitados: o scheduler de jobs é `TASK-E08-01`, em vynamp-engine, e é dependência desta TASK. O lote aqui é consumido pela API que ele expõe; guardar estado de job em disco neste repo duplicaria o lease e quebraria a retomada depois de queda.
- **Não** implementa o registry de decodificação das entradas: ele já existe em `vynamp-decode-core` (`Registry::probe`) e os adapters dos 17 formatos de áudio do produto vêm de `TASK-E01-04`, `TASK-E01-07`, `TASK-E01-08` e `TASK-E17-05`, em vynamp-decode. O registry desta TASK é só do lado da escrita, e a confusão é plausível porque o título diz encoder registry enquanto o app lê todos os formatos de entrada.
- **Não** faz restauração hi-res — upscale de profundidade, denoise ou recuperação de conteúdo perdido: essa ferramenta é o Vynamp Restore, `TASK-E18-01`. A AC-3 é regra de rótulo, não de processamento: a conversão lossy tem perda de projeto e o relatório diz isso; melhorar o sinal para caber em outro rótulo seria outro produto.
- **Não** decide se MP3 e AAC entram na build nem resolve patente e licença: o inventário de codecs, fontes e políticas de distribuição é `TASK-E11-03`, e a decisão de licença não é tomada aqui, apenas consumida. Nesta TASK, quando permitido significa quando o encoder estiver presente na build, e o registry apenas o expõe ou o declara ausente.
- **Não** define template de nome de saída, organização de pastas nem dedupe: a whitelist de campos e a sanitização de separadores e nomes reservados são `TASK-E08-04`. Nome de saída aqui é parâmetro recebido do chamador; criar template próprio daria duas regras de colisão de nome para o mesmo diretório.

## Antes de começar

- [ ] Spec do módulo aprovada e o critério de aceite citado já existe nela
- [ ] Worktree própria criada (`cargo xtask repo worktree task-e08-02-claude-<slug>`)
- [ ] Fixtures/hardware necessários disponíveis ou limitação registrada
- [ ] Nenhuma TASK em execução tocando os mesmos arquivos

## Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| `TASK-E08-02-AC-1` | um lote de 3 originais com SHA-256 registrado antes do job — FLAC 24 bits 96 kHz estéreo de 2 480 000 frames (25,833 s), WAV PCM 16 bits 44,1 kHz mono e Opus 128 kbps — com destino em diretório vazio do mesmo volume da origem, `--overwrite` desligado e nenhum nome de saída já existente no destino | o worker converte cada arquivo no temporário do mesmo volume, faz fsync, relê e decodifica o temporário, compara contêiner, taxa, canais e duração com o plano do job e só então renomeia sobre o destino; o harness lê o temporário antes do rename e a linha de relatório de cada arquivo | Validar output após escrita |
| `TASK-E08-02-AC-2` | um lote de 3 FLAC 24 bits 96 kHz com SHA-256 e mtime das origens registrados antes do job, um diretório de destino com espaço livre medido antes do job, acima dos 3 temporários de 15 MiB, o cancelamento por arquivo exposto pela fila de `TASK-E08-01` e um `Write` de teste que devolve `io::ErrorKind::StorageFull` na metade da escrita do arquivo 2 | o cancelamento do arquivo 2 é entregue enquanto o temporário de 15 MiB está parcialmente escrito e, em outra execução, o mesmo ponto falha com StorageFull injetado; depois o harness lista o diretório de destino, conta temporários restantes e recalcula SHA-256 e mtime das 3 origens | cancelamento/disco cheio preservam source |
| `TASK-E08-02-AC-3` | um FLAC 16 bits 44,1 kHz estéreo convertido para Opus 128 kbps VBR, o mesmo original convertido para MP3 320 kbps CBR e um FLAC 24 bits 96 kHz convertido para Opus 256 kbps, com relatório por arquivo em JSON e a tela do lote mostrando uma linha de status por saída | o lote termina com os 3 jobs em sucesso e o harness lê o relatório e os textos de interface de cada saída, colhendo o tipo declarado, a taxa alvo e qualquer menção a restauração ou ausência de perda, e compara com a lista de termos proibidos do glossário do produto | conversão lossy não é anunciada como restauração hi-res. |

## Verificação

| Item | Comando / fixture | Evidência esperada |
|---|---|---|
| Teste automatizado | `cargo test` no repo `vynamp-converter` | verde no CI Gate |
| Oráculo independente | um teste de integração novo em `apps/converter/tests/escrita_atomica.rs`, com um `std::io::Write` de teste que devolve `io::ErrorKind::StorageFull`, rodado com `cargo test --test escrita_atomica` sobre 3 fixtures pequenos — FLAC 24 bits 96 kHz estéreo de 2 480 000 frames (25,833 s), WAV PCM 16 bits 44,1 kHz mono e Opus 128 kbps — com destino em diretório vazio do mesmo volume, mais o cancelamento do arquivo 2 de 3 disparado no meio do temporário. O repo `apps/converter` tem hoje 1 arquivo .rs (`src/main.rs`, 3 linhas imprimindo `vynamp_converter 0.1.0`), 0 arquivos em `tests/`, `[dependencies]` vazio e só o alvo `[[bin]]`, sem `[lib]`; teste em `tests/` não importa a API de um binário, então nem a biblioteca nem o teste existem e a evidência não compila | por arquivo: o temporário relido declara e entrega o mesmo contêiner, taxa, canais e duração do plano (2 480 000 frames a 96 kHz, 1 canal em 44,1 kHz), e o SHA-256 e o mtime das 3 origens ficam idênticos aos registrados antes do job; 0 temporários `.part` sobrando no destino depois do cancelamento e depois do StorageFull injetado, com código de saída 4 para cancelado e 3 para falha de escrita, como a spec §2 define; as 3 saídas lossy do AC-3 saem rotuladas como lossy com taxa alvo e sem as expressões de restauração ou sem perda. O que ainda não existe hoje: nenhuma linha de conversão de áudio no repo, nenhum teste no repo, `libs/encode/src/lib.rs` é só `pub const VERSION`, e nenhum crate do umbrella tem encoder de produto — o único codificador no código é `vynamp-decode-mp3/src/layer3/testenc.rs`, 279 linhas atrás de `#[cfg(test)]`, e não há FLAC, Opus, Vorbis nem muxer; reler um temporário Opus ainda depende do adapter de Opus de `TASK-E01-07`, que segue planned |
| Hardware | nenhum: entrada e saída dos três critérios são arquivos em disco e nenhum dispositivo de áudio é aberto; o disco cheio real pediria um volume de 64 MiB ou um VHD, que a suíte não cria por não ser portátil, por isso o ENOSPC entra por um `Write` que devolve StorageFull. Esta máquina é Windows sem placa pro, sem Android, sem iOS e sem macOS, e nenhum dos critérios os exige | ausência é limitação registrada, não aprovação |

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
