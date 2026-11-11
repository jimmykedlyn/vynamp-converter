# Spec — vynamp-converter · converter

| Campo | Valor |
|---|---|
| Módulo | `vynamp-converter` |
| Crate(s) | `vynamp-converter` (`[lib]` + `[[bin]]`; crate único até uma TASK justificar divisão) |
| Tipo | app |
| Versão da spec | 0.2.0 |
| Data | 2026-11-11 |
| Estado | **rascunho autorado, não aprovado** — §3 a §6, §9 e §10 escritos a partir de `reference/tools/02-vynamp-converter.md` §1, §2, §4, §5 e §8 e das TASKs E08-02 e E18-02. A aprovação é do mantenedor, por `GOVERNANCE.md` |
| Dono | mantenedor do `vynamp-converter` |
| Depende de | `vynamp-engine`, `vynamp-encode` |
| É dependido por | ninguem |

> §1, §2 e §7 foram decididos em `01-arvore-final.md` §2 e valem como norma.
> §3 a §6, §9 e §10 foram autorados em 2026-11-11 a partir de
> `reference/tools/02-vynamp-converter.md`, que continua sendo a fonte normativa do
> produto. Estar escrito não é estar aprovado: até o mantenedor aprovar, nenhum número
> aqui é contrato, e toda TASK que dependa dele continua `planned`. Esta spec cobre a
> interface de linha de comando, os códigos de saída, o relatório por arquivo e a
> rotulagem honesta; fila, escrita atômica e cancelamento vivem em `lote.md`.
>
> **A fonte normativa é o repositório principal, não este arquivo.** Ver
> [`specs/README.md`](README.md) para o documento canônico deste módulo.

## 1. Propósito

App desktop de conversao em lote, com fila persistida e preview.

**Não faz:** Nao toca audio, nao restaura, nao gerencia biblioteca.

## 2. Fronteira

| Permitido | Proibido |
|---|---|
| vynamp-engine, vynamp-encode | acessar decode/output/dsp direto quando houver API no engine; depender do player ou do restore |

## 3. API pública

O contrato primário é a linha de comando. O crate expõe também um `[lib]` fino para
que os testes de integração dirijam o app sem subprocesso; o binário e a GUI só
chamam `run`.

### Linha de comando

```
vynamp-converter [opções] <caminhos...>
```

| Opção | Valor | Padrão |
|---|---|---|
| `--format` | `flac`, `wav`, `aiff`, `alac`, `opus`, `mp3`, `aac`, `vorbis` | `flac` |
| `--bitrate` | kbps, só formatos lossy | por formato, ver `02-vynamp-converter.md` §2 |
| `--quality` | qualidade Vorbis | por formato |
| `--sample-rate` | Hz | a da fonte |
| `--bit-depth` | `16`, `24`, `32`, `32f` | a da fonte, limitada pela matriz |
| `--channels` | `1` ou `2` | a da fonte |
| `--out` | diretório de destino | ao lado da fonte |
| `--name-template` | template da lista branca de `lote.md` §3 | `{title}` |
| `--overwrite` | sem valor; permite sobrescrever colisão | desligado |
| `--confirm-degradation` | sem valor; confirma perda de taxa, profundidade ou canais | desligado |
| `--dry-run` | sem valor; mostra o plano de nomes e o relatório previsto, não escreve | desligado |
| `--preset` | nome de preset salvo | nenhum |
| `--report` | caminho do relatório JSON | `<out>/vynamp-converter-report.json` |
| `--version`, `--help` | — | — |

`--confirm-degradation` é a forma não interativa da confirmação explícita que
TASK-E18-02 AC-2 exige; a GUI pergunta ao usuário e passa o mesmo sinal para `run`.
Os caminhos entram na fila na ordem recebida (lote atômico do shell, ADR-012).

### Códigos de saída

| Código | Significado |
|---|---|
| 0 | todos os arquivos convertidos e validados |
| 1 | erro de uso: opção inválida, formato inexistente (inclusive DSF/DFF), degradação sem `--confirm-degradation` |
| 2 | entrada ilegível ou formato de entrada não suportado |
| 3 | falha de escrita (inclusive disco cheio e validação pós-escrita reprovada) |
| 4 | cancelado |
| 5 | erro interno do encoder |

Em lote misto vale o código do pior arquivo, na ordem 5 > 3 > 2 > 4 > 0; erro de uso
(1) aborta antes de qualquer escrita.

### Biblioteca

```rust
/// Executa o converter com os argumentos já separados (sem o nome do programa).
pub fn run(args: &[OsString], cancel: CancelToken) -> Outcome;

pub struct Outcome {
    pub exit_code: u8,
    pub report: Report,
}

/// Relatório por lote; serializado em JSON no caminho de `--report`.
pub struct Report {
    pub files: Vec<FileReport>,
}

pub struct FileReport {
    pub source: PathBuf,
    pub destination: Option<PathBuf>,
    pub source_format: String,       // contêiner + codec, ex. "FLAC"
    pub source_rate: u32,
    pub source_bits: Option<u8>,
    pub source_channels: u16,
    pub target_format: vynamp_encode::Format,
    pub params: String,              // ex. "Opus 96 kbps VBR"
    pub gain_db: Option<f32>,
    pub degradations: Vec<String>,   // vem de `ValidatedPlan::degradations`
    pub label: QualityLabel,
    pub metadata_warning: Option<String>,
    pub status: FileStatus,
}

pub enum QualityLabel {
    /// Saída sem perdas na resolução da fonte ou na degradação declarada.
    Lossless,
    /// Saída com perdas: o texto declara a perda esperada e que a qualidade
    /// perceptual é estimativa.
    Lossy { expected_loss: String },
}

pub enum FileStatus { Converted, Skipped { reason: String }, Failed(ConvertError), Cancelled }
```

`CancelToken` e `ConvertError` estão em `lote.md` §3 e §6.

## 4. Requisitos

| ID | Requisito | REQ de produto | TASK | Teste |
|---|---|---|---|---|
| CONVERTER-REQ-01 | A CLI aceita exatamente as opções de §3 e devolve os códigos de saída de §3; opção desconhecida ou valor fora da matriz de `vynamp-encode` (`formatos.md` §3) sai com 1 antes de qualquer escrita — planned (M3) | CLI estável | E18-02 | — (CONVERTER-AC-01 planejado) |
| CONVERTER-REQ-02 | Saída de formato lossy é rotulada `Lossy` no relatório e nunca é chamada de “restauração”, “hi-res” ou “sem perdas”; o rótulo vem só de `Format::is_lossy` — planned (M3) | rótulo honesto | E08-02, E18-02 | — (CONVERTER-AC-02 planejado) |
| CONVERTER-REQ-03 | Redução de taxa, profundidade ou canais em relação à fonte exige confirmação explícita; sem ela sai com 1, com ela a degradação aparece no relatório — planned (M3) | sem degradação implícita | E18-02 | — (CONVERTER-AC-03 planejado) |
| CONVERTER-REQ-04 | O relatório registra, por arquivo, formato, taxa, profundidade e canais da fonte, formato e parâmetros do destino, ganho aplicado e cada degradação — planned (M3) | relatório por arquivo | E18-02 | — (CONVERTER-AC-04 planejado) |
| CONVERTER-REQ-05 | Metadados e capa são extraídos pelas regras de `restore` §7 e reembutidos no destino; campo desconhecido é preservado quando o contêiner suporta; falha de metadados vira `metadata_warning` e não bloqueia a conversão — planned (M3) | metadados preservados | E18-02 | — (CONVERTER-AC-05 planejado) |
| CONVERTER-REQ-06 | `--dry-run` não cria, altera nem apaga nenhum arquivo e mostra o plano completo de nomes e o relatório previsto — planned (M3) | prévia sem efeito | E18-02 | — (CONVERTER-AC-06 planejado) |
| CONVERTER-REQ-07 | O app é só desktop: não há alvo mobile, nem compartilhamento Android ou deep link — planned (M3) | escopo desktop | E18-02 | — (CONVERTER-AC-07 planejado) |

Os rótulos da coluna “REQ de produto” são descritivos; o ID de produto correspondente
é o da TASK citada no backlog.

## 5. Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| CONVERTER-AC-01 | a tabela de casos de opção: cada opção de §3 com valor válido, valor fora da matriz (ex.: `--bitrate 300` para Opus), opção desconhecida e `--format dsf` | `run` executa cada caso com `--dry-run` | válidos saem 0; os demais saem 1 e nenhum arquivo aparece em `--out` |
| CONVERTER-AC-02 | `sine-24-stereo.flac` (48 kHz, 24 bits, 2 canais, 4 800 quadros) convertido para Opus 96, Opus 128, Opus 256, MP3 320 CBR e FLAC | `run` termina cada conversão | os quatro lossy têm `label = Lossy` com `expected_loss` não vazio, o FLAC tem `Lossless`, e nem o relatório nem a saída de terminal contêm “restaura”, “hi-res” ou “sem perdas” para os lossy |
| CONVERTER-AC-03 | `sine-24-stereo.flac` e o pedido `--format flac --bit-depth 16 --sample-rate 44100`, primeiro sem e depois com `--confirm-degradation` | `run` executa os dois | o primeiro sai 1 sem escrever; o segundo sai 0 e `degradations` contém “24→16 bits” e “48 000→44 100 Hz” |
| CONVERTER-AC-04 | `sine-16-mono-tail.flac` convertido para FLAC estéreo com `--channels 2 --confirm-degradation` | `run` termina | o relatório tem os campos de CONVERTER-REQ-04 preenchidos e `gain_db` igual ao ganho declarado |
| CONVERTER-AC-05 | um FLAC com título, artista, capa e um campo Vorbis desconhecido, e outro com bloco de metadados corrompido | ambos são convertidos para FLAC e para Opus | o primeiro preserva título, artista, capa e o campo desconhecido; o segundo converte com `metadata_warning` preenchido e sai 0 |
| CONVERTER-AC-06 | uma pasta com os três fixtures de §9 e o hash e a lista de arquivos de `--out` antes da execução | `run` executa com `--dry-run` | a pasta e `--out` ficam byte a byte iguais e a saída lista um nome planejado por fonte |
| CONVERTER-AC-07 | o `Cargo.toml` e os alvos de build do crate | o gate de alvos lê os destinos declarados | não existe alvo Android/iOS nem registro de share target ou deep link |

**Nenhum destes testes existe ainda.** O nome é o compromisso; a TASK citada em §4 é
quem os escreve, e o teste nasce vermelho antes da implementação.

### Critérios que esta spec **não** fixa

- Texto exato de `expected_loss`: a spec exige que exista e que não prometa
  transparência; a redação é de UX.
- Interface gráfica: a GUI desktop (ícone C/Pulso com selo lima `#B8F35A`) chama a
  mesma `run`; layout e preview visual são do design, não desta spec.
- Valores padrão de bitrate por formato: estão em `02-vynamp-converter.md` §2 e podem
  mudar sem nova versão desta spec.

## 6. Erros e cancelamento

**Erro é dado, não `panic!`.** Cada falha de arquivo vira `FileStatus::Failed` com um
`ConvertError` (`lote.md` §6) e o lote segue; a CLI traduz o pior status no código de
saída de §3.

| Situação | Resultado |
|---|---|
| Opção inválida, formato inexistente, degradação não confirmada | código 1, nada escrito |
| Fonte ilegível ou formato de entrada não suportado | arquivo `Failed`, código 2 |
| Disco cheio, permissão, validação pós-escrita reprovada | arquivo `Failed`, código 3, fonte intacta |
| Erro do encoder | arquivo `Failed`, código 5 |
| Falha de metadados | arquivo `Converted` com `metadata_warning` |

**Cancelamento é explícito, não timeout.** O `CancelToken` cancela um arquivo ou o
lote; o arquivo cancelado vira `Cancelled` e o código de saída é 4 quando não houver
erro pior. A garantia sobre fonte e temporários está em `lote.md` §6.

## 7. Fora de escopo

Codificadores proprios: a unica fonte e a lib `vynamp-encode`. Jobs persistidos vem do engine (E08-01)

## 8. Dependências

Depende de: `vynamp-engine`, `vynamp-encode`

É dependido por: ninguem

## 9. Testes e fixtures

| Fixture | Origem | Uso |
|---|---|---|
| `sine-24-stereo.flac` (48 kHz, 24 bits, 2 canais, 4 800 quadros, 21 109 bytes) | cópia de `vynamp-decode` `crates/vynamp-decode-flac/tests/fixtures/` para `tests/fixtures/` deste repo | CONVERTER-AC-02, CONVERTER-AC-03, CONVERTER-AC-06 |
| `sine-16-stereo.flac` | idem | CONVERTER-AC-01, CONVERTER-AC-06 |
| `sine-16-mono-tail.flac` | idem | CONVERTER-AC-04, CONVERTER-AC-06 |
| FLAC com metadados ricos e FLAC com bloco corrompido | gerados no teste a partir de `sine-16-stereo.flac` | CONVERTER-AC-05 |

Oráculo: o relatório JSON e os códigos de saída, comparados com tabelas literais no
teste; o conteúdo de áudio de saída é verificado pela validação pós-escrita de
`lote.md`. Todos os fixtures são pequenos, então nenhum teste fica atrás de
`#[ignore]`.

```
cargo test --test cli --locked
cargo test --test rotulo_relatorio --locked
```

## 10. Limitações conhecidas

Limitação declarada não reprova a entrega; limitação omitida reprova.

| Limitação | Efeito | Quem cobre |
|---|---|---|
| `src/main.rs` tem três linhas e o crate não tem `[lib]` | nenhum critério verificável hoje | E18-02 cria o `[lib]` |
| Os fixtures ainda não foram copiados para este repo | §9 depende de cópia | E18-02 |
| MP3 só existe se o encoder estiver na build | `--format mp3` sai 1 em build sem ele | `vynamp-encode` `FormatAbsent` |
| Estimativa de perda é texto, não medida | o usuário recebe aviso, não número perceptual | declarado no próprio rótulo |
| Edição de tags, organização/deduplicação (E08-04), CUE e gravação de disco (E08-05) | não existem no converter | TASKs citadas |
