# TASK-E08-02 — Entregar conversor e encoder registry

> Gerado pelo passo 37 a partir de `tasks/backlog.json`. Os campos de estado vêm do
> backlog e não devem ser editados aqui. Critério de aceite: **PENDENTE** de autoria —
> a coluna `Then` é o requisito aprovado, copiado literal; `Given` e `When` ainda
> não existem.

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

<!-- PENDENTE: o que esta TASK não faz, mesmo parecendo relacionado. -->

## Antes de começar

- [ ] Spec do módulo aprovada e o critério de aceite citado já existe nela
- [ ] Worktree própria criada (`cargo xtask repo worktree task-e08-02-claude-<slug>`)
- [ ] Fixtures/hardware necessários disponíveis ou limitação registrada
- [ ] Nenhuma TASK em execução tocando os mesmos arquivos

## Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| `TASK-E08-02-AC-1` | PENDENTE | PENDENTE | Validar output após escrita |
| `TASK-E08-02-AC-2` | PENDENTE | PENDENTE | cancelamento/disco cheio preservam source |
| `TASK-E08-02-AC-3` | PENDENTE | PENDENTE | conversão lossy não é anunciada como restauração hi-res. |

## Verificação

| Item | Comando / fixture | Evidência esperada |
|---|---|---|
| Teste automatizado | `cargo test` no repo `vynamp-converter` | verde no CI Gate |
| Oráculo independente | PENDENTE | comparação numérica, não "parece certo" |
| Hardware | PENDENTE | ausência é limitação registrada, não aprovação |

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
