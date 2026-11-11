# Spec — vynamp-converter · lote

| Campo | Valor |
|---|---|
| Módulo | `vynamp-converter` |
| Crate(s) | `vynamp-converter` (módulo `lote`; crate único até uma TASK justificar divisão) |
| Tipo | app |
| Versão da spec | 0.2.0 |
| Data | 2026-11-11 |
| Estado | **rascunho autorado, não aprovado** — §3 a §6, §9 e §10 escritos a partir de `reference/tools/02-vynamp-converter.md` §4, §5 e §8 e das TASKs E08-01, E08-02 e E18-02. A aprovação é do mantenedor, por `GOVERNANCE.md` |
| Dono | mantenedor do `vynamp-converter` |
| Depende de | `vynamp-engine`, `vynamp-encode` |
| É dependido por | ninguem |

> §1, §2 e §7 foram decididos em `01-arvore-final.md` §2 e valem como norma.
> §3 a §6, §9 e §10 foram autorados em 2026-11-11 a partir de
> `reference/tools/02-vynamp-converter.md`, que continua sendo a fonte normativa do
> produto. Estar escrito não é estar aprovado: até o mantenedor aprovar, nenhum número
> aqui é contrato, e toda TASK que dependa dele continua `planned`. Esta spec cobre a
> fila, a escrita atômica, a validação pós-escrita, a colisão de nomes e o
> cancelamento; a linha de comando e o relatório vivem em `converter.md`.
>
> **A fonte normativa é o repositório principal, não este arquivo.** Ver
> [`specs/README.md`](README.md) para o documento canônico deste módulo.

## 1. Propósito

App desktop de conversao em lote, com fila persistida e preview.

**Não faz:** Nao toca audio, nao restaura, nao gerencia biblioteca.

## 2. Fronteira

| Permitido | Proibido |
|---|---|
| herda a fronteira de `specs/converter.md` | herda as proibicoes de `specs/converter.md` |

## 3. API pública

```rust
/// Sinal de cancelamento compartilhado entre a CLI/GUI e os workers.
#[derive(Clone)]
pub struct CancelToken { /* Arc interno */ }

impl CancelToken {
    pub fn new() -> Self;
    /// Cancela o lote inteiro.
    pub fn cancel_all(&self);
    /// Cancela só o arquivo de índice `index` na fila.
    pub fn cancel_file(&self, index: usize);
    pub fn is_cancelled(&self, index: usize) -> bool;
}

/// Plano do lote, montado antes de qualquer escrita.
pub struct BatchPlan {
    pub items: Vec<PlannedItem>, // na ordem recebida
}

pub struct PlannedItem {
    pub source: PathBuf,
    pub destination: PathBuf,    // já resolvido pelo template e sanitizado
    pub params: vynamp_encode::ValidatedPlan,
}

/// Monta o plano: aplica o template, sanitiza, detecta colisões e duplicatas.
pub fn plan(sources: &[PathBuf], opts: &BatchOptions) -> Result<BatchPlan, ConvertError>;

/// Executa o plano pela fila persistida do engine (E08-01).
pub fn execute(plan: BatchPlan, cancel: CancelToken) -> Vec<FileReport>;
```

### Fila

- Até **100 arquivos** por lote; acima disso `plan` recusa com `Usage`.
- Workers: no máximo `núcleos − 1`, mínimo 1.
- A fila é a fila persistida de jobs do `vynamp-engine` (E08-01); o converter não
  mantém fila própria.
- Os itens entram na ordem recebida; a ordem de término pode variar, a do relatório
  não.

### Escrita atômica

Cada arquivo segue, nesta ordem:

1. escreve em `<destino>.part` no **mesmo volume** do destino;
2. `fsync` do temporário;
3. relê e decodifica o temporário: contêiner, taxa, canais e duração precisam bater
   com o plano (validação pós-escrita);
4. `rename` atômico para o nome final.

Em qualquer falha ou cancelamento, o `.part` é removido e a fonte não é tocada.

### Template de nome

Lista branca de campos: `{title}`, `{artist}`, `{album}`, `{track}`, `{year}`. Campo
fora da lista é erro de uso. O resultado é sanitizado (caracteres proibidos no sistema
de arquivos, nomes reservados do Windows, comprimento) e o plano completo de nomes é
mostrado antes de escrever.

### Colisão e idempotência

- Colisão com arquivo existente ou entre itens do mesmo lote é detectada em `plan`,
  antes de qualquer escrita; sem `--overwrite`, o item vira `Skipped` com o motivo.
- Job duplicado (mesma fonte, mesmos parâmetros, mesmo destino) é detectado e
  executado uma vez.
- Reexecutar um lote já concluído não muda nenhum arquivo.

## 4. Requisitos

| ID | Requisito | REQ de produto | TASK | Teste |
|---|---|---|---|---|
| LOTE-REQ-01 | O lote aceita até 100 arquivos, usa no máximo `núcleos − 1` workers e roda pela fila persistida do engine — planned (M3) | fila persistida | E08-01, E08-02 | — (LOTE-AC-01 planejado) |
| LOTE-REQ-02 | Toda escrita passa por `.part` no mesmo volume, `fsync`, validação pós-escrita e `rename`; nenhum arquivo final existe sem ter sido validado — planned (M3) | escrita atômica | E08-02, E18-02 | — (LOTE-AC-02 planejado) |
| LOTE-REQ-03 | Cancelamento por arquivo ou por lote, e disco cheio, preservam SHA-256 e mtime de todas as fontes e deixam zero `.part` — planned (M3) | fonte intocável | E08-02, E18-02 | — (LOTE-AC-03 planejado) |
| LOTE-REQ-04 | Colisão de nome é detectada antes de escrever e nunca sobrescreve sem `--overwrite` — planned (M3) | sem sobrescrita silenciosa | E18-02 | — (LOTE-AC-04 planejado) |
| LOTE-REQ-05 | O template só aceita campos da lista branca, sanitiza o resultado e mostra o plano completo de nomes antes de escrever — planned (M3) | nomes previsíveis | E18-02 | — (LOTE-AC-05 planejado) |
| LOTE-REQ-06 | Processamento é idempotente e job duplicado roda uma vez — planned (M3) | idempotência | E08-01, E18-02 | — (LOTE-AC-06 planejado) |
| LOTE-REQ-07 | Temporário truncado ou com duração, taxa ou canais divergentes reprova a validação, não é renomeado e vira `Failed(Validation)` — planned (M3) | validação pós-escrita | E18-02 | — (LOTE-AC-07 planejado) |

Os rótulos da coluna “REQ de produto” são descritivos; o ID de produto correspondente
é o da TASK citada no backlog.

## 5. Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| LOTE-AC-01 | um lote de 101 caminhos e outro de 3, numa máquina de N núcleos | `plan` e `execute` rodam | o de 101 é recusado com `Usage`; o de 3 usa no máximo `max(1, N − 1)` workers e o relatório sai na ordem recebida |
| LOTE-AC-02 | um lote com FLAC 24/96 estéreo de 2 480 000 quadros, WAV 16/44,1 mono e Opus 128 | `execute` termina | os três destinos existem, cada um decodifica com contêiner, taxa, canais e duração do plano, e não sobra nenhum `.part` |
| LOTE-AC-03 | o lote de LOTE-AC-02 com SHA-256 e mtime das fontes registrados, executado três vezes: cancelando o lote no meio, cancelando só o segundo arquivo, e com destino que devolve `StorageFull` | cada execução termina | em todas, SHA-256 e mtime das fontes são idênticos aos registrados e não existe nenhum `.part`; os itens afetados são `Cancelled` ou `Failed(Write{kind: StorageFull})` |
| LOTE-AC-04 | um lote de 10 arquivos cujos nomes planejados colidem entre si e com um arquivo existente em `--out` | `plan` roda sem e depois com `--overwrite` | sem a opção, todas as colisões são listadas antes de qualquer escrita e os itens viram `Skipped`; com a opção, só o arquivo existente é sobrescrito, uma vez |
| LOTE-AC-05 | templates `{artist} - {title}`, `{track}/{title}`, `{title}:{year}`, `CON` e `{genre}` | `plan` roda | os três primeiros geram nomes sanitizados e válidos no SO; `CON` é ajustado; `{genre}` sai com `Usage` |
| LOTE-AC-06 | o mesmo lote executado duas vezes, e um lote com o mesmo item repetido | `execute` roda | a segunda execução não altera nenhum byte nem mtime de destino; o item repetido é convertido uma vez |
| LOTE-AC-07 | um encoder de teste que trunca o temporário antes do `fsync` | `execute` roda | o item vira `Failed(Validation)`, o destino final não existe e o `.part` foi removido |

**Nenhum destes testes existe ainda.** O nome é o compromisso; a TASK citada em §4 é
quem os escreve, e o teste nasce vermelho antes da implementação.

### Critérios que esta spec **não** fixa

- Formato de persistência da fila: é do `vynamp-engine` (E08-01).
- Política de retomada após queda do processo: depende da fila do engine; esta spec só
  exige que nenhum `.part` seja promovido sem validação.
- Regras exatas de sanitização por SO: a spec exige nome válido e plano visível, não a
  tabela de substituição.

## 6. Erros e cancelamento

**Erro é dado, não `panic!`.**

```rust
pub enum ConvertError {
    Usage { reason: String },
    UnreadableInput { path: PathBuf, reason: String },
    Write { path: PathBuf, kind: std::io::ErrorKind },
    Validation { path: PathBuf, expected: String, found: String },
    Encoder(vynamp_encode::EncodeError),
    Cancelled,
    Collision { destination: PathBuf, with: PathBuf },
}
```

| Situação | Erro | Efeito |
|---|---|---|
| Lote acima de 100, campo de template fora da lista | `Usage` | nada escrito, código 1 |
| Fonte ilegível | `UnreadableInput` | item `Failed`, segue o lote |
| Disco cheio, permissão, volume diferente | `Write { kind }` | `.part` removido, item `Failed` |
| Temporário diverge do plano | `Validation` | `.part` removido, item `Failed` |
| Encoder falhou | `Encoder` | `.part` removido, item `Failed` |
| Colisão sem `--overwrite` | `Collision` | item `Skipped` |

**Cancelamento é explícito, não timeout.** `CancelToken::cancel_file` e `cancel_all`
são checados entre blocos de encode; ao ver o cancelamento, o worker para, remove o
`.part` e marca o item `Cancelled`. Arquivos já renomeados permanecem. A fonte nunca é
aberta para escrita.

## 7. Fora de escopo

Codificadores proprios: a unica fonte e a lib `vynamp-encode`. Jobs persistidos vem do engine (E08-01)

## 8. Dependências

Depende de: `vynamp-engine`, `vynamp-encode`

É dependido por: ninguem

## 9. Testes e fixtures

| Fixture | Origem | Uso |
|---|---|---|
| `sine-24-stereo.flac`, `sine-16-stereo.flac`, `sine-16-mono-tail.flac` | cópia de `vynamp-decode` `crates/vynamp-decode-flac/tests/fixtures/` | LOTE-AC-01, LOTE-AC-04 a LOTE-AC-06 |
| FLAC 24/96 estéreo de 2 480 000 quadros, WAV 16/44,1 mono | gerados no teste (seno) | LOTE-AC-02, LOTE-AC-03 |
| destino que devolve `StorageFull` | sistema de arquivos falso no teste | LOTE-AC-03 |
| encoder que trunca | dublê de teste | LOTE-AC-07 |

Oráculo: SHA-256 e mtime das fontes, listagem do diretório de destino (incluindo
`.part`) e decodificação do arquivo final. O FLAC de 2 480 000 quadros é gerado em
tempo de teste e cabe no tempo de CI, então nenhum teste fica atrás de `#[ignore]`.

```
cargo test --test escrita_atomica --locked
cargo test --test fila_lote --locked
```

## 10. Limitações conhecidas

Limitação declarada não reprova a entrega; limitação omitida reprova.

| Limitação | Efeito | Quem cobre |
|---|---|---|
| Nenhum tipo desta spec existe em `src/` | nenhum critério verificável hoje | E18-02 |
| A fila persistida do engine ainda não existe | `execute` não tem onde rodar | E08-01 |
| `rename` atômico exige mesmo volume | destino em outro volume exige `.part` no volume de destino, não no da fonte | regra de §3 |
| Cancelamento é checado entre blocos | um bloco em andamento termina antes de parar | declarado aqui |
