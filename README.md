# Relatório de análise — resbra.zip 

Data da análise: 2026-05-14  
Arquivo analisado: `resbra.zip`  
Tipo de análise: auditoria estática de documentação, consistência interna, SQL documentado e modelagem relacional.

> Observação: esta análise não executou migrations em um PostgreSQL real, porque o pacote contém documentação Markdown e snippets SQL, não um conjunto canônico de DDL completo. As conclusões abaixo são baseadas em leitura estática, parsing de Markdown, validação de links, checagem de snippets e comparação entre tabelas, funções, triggers e diagramas.

---

## 1. Sumário executivo

A nova versão melhorou bastante em relação à anterior. O pacote está muito mais limpo, não contém mais a árvore `.obsidian/plugins`, não há arquivos `.md.md`, os links internos não vazios resolvem, a pasta de triggers foi padronizada como `TRG_UPDATES`, a tabela `negociacoes` foi renomeada corretamente no diretório, a duplicidade de `descricao` em `outorgas` foi corrigida, e `notificar_grupo()` passou a usar `u.id` em vez de `u.cpf`.

Ainda assim, a documentação não está pronta para ser usada como fonte canônica de schema/migration. O problema mais grave é que `fn_alertar_renovacao_licencas()` continua inconsistente com `notificacoes`: agora a função usa `id_usuario`, mas a lista de colunas do `INSERT` tem 8 colunas e o `SELECT` retorna 9 valores, porque `l.id` foi inserido como valor de `entidade_id` sem que a coluna `entidade_id` tenha sido adicionada ao `INSERT`. Isso quebraria a execução da função.

O segundo ponto crítico é que praticamente todas as tabelas continuam com RLS documentado como habilitado, mas sem policies. Isso é aceitável como diagnóstico temporário, mas perigoso como estado final: em um banco real, usuários comuns tenderiam a não conseguir ler nem alterar linhas.

O terceiro ponto crítico é que os ERDs adicionados são úteis, mas vários estão defasados em relação às próprias tabelas documentadas. Encontrei 49 campos nos diagramas que não existem nas tabelas correspondentes e uma entidade inexistente (`negocioacoes`). Isso indica que os diagramas ainda foram gerados com base em um modelo anterior ou em uma interpretação diferente do schema atual.

Minha avaliação: **a versão nova está mais organizada e muito mais próxima de uma documentação técnica robusta, mas ainda precisa de uma rodada de saneamento antes de virar referência oficial, base de geração de DDL ou documentação auditável**.

---

## 2. Inventário analisado

| Item | Resultado |
|---|---:|
| Arquivos no diretório extraído | 79 |
| Diretórios | 20 |
| Arquivos Markdown | 79 |
| Tabelas documentadas | 42 |
| Funções documentadas | 7 |
| Triggers documentadas | 24 |
| Diagramas Markdown/Mermaid | 5 |
| Links Obsidian encontrados | 98 |
| Links Obsidian não vazios | 97 |
| Links Obsidian quebrados | 0 |
| Links Obsidian vazios | 1 |
| Arquivos vazios | 1 |
| Diretórios vazios | 3 |
| Tabelas com RLS habilitado e sem policy documentada | 41 |
| Tabelas sem confirmação objetiva de RLS habilitado | 1 |
| Entidades Mermaid inexistentes | 1 |
| Campos Mermaid inexistentes nas tabelas | 49 |

O ZIP passou no teste de integridade de compressão; o problema não está no arquivo compactado, mas nas inconsistências internas da documentação.

---

## 3. O que melhorou em relação à versão anterior

### Melhorias claras

1. **Pacote muito mais limpo**: a pasta `.obsidian/plugins` foi removida, reduzindo ruído e risco de distribuir configuração/plugin desnecessário.
2. **Extensões corrigidas**: não encontrei mais arquivos `.md.md`.
3. **Triggers padronizadas**: a pasta atual é `Triggers/TRG_UPDATES/`, sem o erro anterior `TGR_UPDATES`.
4. **Tabela `negociacoes` corrigida no diretório**: o arquivo agora é `Tabelas/MODULO6/negociacoes.md`, corrigindo a grafia do nome da tabela no filesystem.
5. **`notificar_grupo()` corrigida**: a função passou a inserir `u.id` em `notificacoes.id_usuario`, compatível com `UUID REFERENCES usuarios_roles(id)`.
6. **`fn_numerar_requerimento()` melhorada**: a versão atual abandonou `MAX(...) + 1` e passou a usar uma tabela auxiliar com `INSERT ... ON CONFLICT DO UPDATE`, que é uma abordagem muito mais segura contra condição de corrida.
7. **`outorgas` corrigida**: a duplicidade anterior do campo `descricao` não aparece mais.
8. **Links internos melhoraram muito**: encontrei 97 links Obsidian não vazios e nenhum link quebrado.
9. **Diagramas foram adicionados**: agora há material visual/ERD para navegação arquitetural, ainda que precise ser sincronizado com as tabelas reais.

### Melhorias parciais

1. **Funções de notificação**: a troca de `cpf` para `id` foi feita em `notificar_grupo()` e parcialmente em `fn_alertar_renovacao_licencas()`, mas esta última ficou com erro de quantidade de colunas.
2. **`negociacoes`**: a tabela foi corrigida no diretório, mas o ERD principal ainda usa `negocioacoes`.
3. **Arquitetura RAG**: o tema está mais bem descrito em `rag_chunks.md`, mas `Arquitetura_RAG.md` continua vazio.
4. **RLS**: a documentação explicita o impacto de RLS sem policies, o que é bom como diagnóstico, mas ainda não há policies nem matriz de acesso.

---

## 4. Achados críticos

### C1. `fn_alertar_renovacao_licencas()` tem erro de quantidade de colunas no INSERT

Arquivo afetado: `Functions/fn_alertar_renovacao_licencas().md`

A função declara o `INSERT` em 8 colunas:

```sql
INSERT INTO notificacoes (
    id,
    id_usuario,
    titulo,
    mensagem,
    tipo,
    entidade_tipo,
    lido,
    criado_em
)
```

Mas o `SELECT` retorna 9 valores:

```sql
gen_random_uuid(),
u.id,
'Licença próxima do vencimento',
CONCAT(...),
'ALERTA',
'licenca',
l.id,
FALSE,
NOW()
```

O valor `l.id` parece ser destinado à coluna `entidade_id`, mas essa coluna não foi incluída no `INSERT`.

Impacto: **crítico**. A função falharia com erro de quantidade de expressões/colunas em PostgreSQL.

Correção recomendada:

```sql
CREATE OR REPLACE FUNCTION fn_alertar_renovacao_licencas()
RETURNS VOID AS $$
BEGIN
    INSERT INTO notificacoes (
        id,
        id_usuario,
        titulo,
        mensagem,
        tipo,
        entidade_tipo,
        entidade_id,
        lido,
        criado_em
    )
    SELECT
        gen_random_uuid(),
        u.id,
        'Licença próxima do vencimento',
        CONCAT(
            'A licença ', l.numero,
            ' vencerá em ', l.validade
        ),
        'ALERTA',
        'licenca',
        l.id,
        FALSE,
        NOW()
    FROM licencas l
    JOIN usuarios_roles u
        ON u.id_operador = l.id_operador
    WHERE
        l.status = 'ATIVA'
        AND l.validade BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL '30 days'
        AND u.ativo = TRUE
        AND NOT EXISTS (
            SELECT 1
            FROM notificacoes n
            WHERE
                n.id_usuario = u.id
                AND n.tipo = 'ALERTA'
                AND n.entidade_tipo = 'licenca'
                AND n.entidade_id = l.id
        );
END;
$$ LANGUAGE plpgsql;
```

Também recomendo criar índice para a condição de deduplicação:

```sql
CREATE INDEX idx_notificacoes_alerta_entidade_usuario
ON notificacoes (id_usuario, tipo, entidade_tipo, entidade_id);
```

Se a regra for “não duplicar alerta de licença por usuário”, uma alternativa ainda mais forte é um índice único parcial:

```sql
CREATE UNIQUE INDEX uq_notificacoes_alerta_licenca_usuario
ON notificacoes (id_usuario, entidade_id)
WHERE tipo = 'ALERTA'
  AND entidade_tipo = 'licenca';
```

---

### C2. RLS habilitado sem policies em 41 tabelas

Arquivos afetados: quase todos os arquivos em `Tabelas/**`, exceto `Tabelas/LEGADO_MAPTEC/legado.maptec_tecnologias.md`, que apenas pede validação do RLS no schema `legado`.

A documentação apresenta repetidamente:

```sql
ALTER TABLE <tabela> ENABLE ROW LEVEL SECURITY;
```

e a situação:

```text
Nenhuma policy identificada.
```

Impacto: **muito alto**. Com RLS habilitado e sem policies, usuários comuns tendem a receber zero linhas em `SELECT`, e `UPDATE`/`DELETE` não afetam registros. Isso pode bloquear completamente a aplicação se ela usa usuários autenticados sem `BYPASSRLS` ou sem service role.

Recomendação elegante: criar uma matriz formal de policies por categoria de tabela. Exemplo de matriz mínima:

| Categoria | Exemplo de tabelas | Leitura | Escrita | Observação |
|---|---|---|---|---|
| Identidade e vínculos | `usuarios_roles`, `operadores`, `responsaveis_legais` | usuário vê seu operador; AEB vê todos | restrita por papel | base das demais policies |
| Operação regulatória | `licencas`, `atividades_espaciais`, `requerimentos` | operador vê seus registros; AEB vê todos | workflow controlado | regras por status/etapa |
| Documentos | `documentos`, `licenca_sei`, `requerimento_sei` | segregada por operador/processo | restrita | cuidado com anexos sensíveis |
| Notificações | `notificacoes` | usuário vê as próprias | usuário marca como lida | criação por sistema/AEB |
| Histórico/auditoria | `auditoria`, `licenca_historico` | leitura restrita | escrita por sistema | idealmente imutável |
| Legado/MAPTEC | `legado.*` | leitura controlada | escrita/migração restrita | considerar `SECURITY DEFINER` com cuidado |

Exemplo conceitual para notificações próprias, ajustável ao mecanismo real de autenticação:

```sql
CREATE POLICY notificacoes_select_proprias
ON notificacoes
FOR SELECT
USING (
    id_usuario = current_setting('app.usuario_id')::uuid
);

CREATE POLICY notificacoes_update_lida_propria
ON notificacoes
FOR UPDATE
USING (
    id_usuario = current_setting('app.usuario_id')::uuid
)
WITH CHECK (
    id_usuario = current_setting('app.usuario_id')::uuid
);
```

Se o sistema usa Supabase, essa expressão deve ser adaptada para `auth.uid()` ou para uma tabela de mapeamento entre usuário autenticado e `usuarios_roles.id`.

---

### C3. ERDs estão substancialmente dessincronizados do modelo documentado

Arquivos afetados:

- `Diagramas/ERD-Modular-Partes.md`
- `Diagramas/ERD-Atividades-Missoes-Partes.md`
- `Diagramas/ERD-Operadores-Atividades-Missoes-Relacionamentos.md`
- `Diagramas/ERD-Operadores-Relacionamentos.md`
- `Diagramas/ERD-Usuarios-Operadores.md`

Encontrei **49 campos nos diagramas que não existem nas respectivas tabelas documentadas** e **1 entidade inexistente**. Exemplos importantes:

| Diagrama | Entidade/campo no ERD | Problema |
|---|---|---|
| `ERD-Modular-Partes.md` | `documentos.id_processo` | `documentos` usa `id_operador`; não possui `id_processo`. |
| `ERD-Modular-Partes.md` | `notificacoes.id_processo` | `notificacoes` usa `id_usuario`, `entidade_tipo`, `entidade_id`; não possui `id_processo`. |
| `ERD-Modular-Partes.md` | `auditoria.id_processo` | `auditoria` usa `tabela`, `id_registro`, `usuario`; não possui `id_processo`. |
| `ERD-Modular-Partes.md` | `rag_chunks.id_documento` | `rag_chunks` usa `fonte` e `id_fonte`; não possui `id_documento`. |
| `ERD-Modular-Partes.md` | `negocioacoes` | Entidade inexistente; deveria ser `negociacoes`. |
| `ERD-Modular-Partes.md` | `garantias.id_licenca` | `garantias` usa `id_atividade`, não `id_licenca`. |
| `ERD-Modular-Partes.md` | `fiscalizacoes.id_licenca` | `fiscalizacoes` usa `id_atividade`, não `id_licenca`. |
| `ERD-Modular-Partes.md` | `incidentes.id_licenca` | `incidentes` usa `id_atividade`, não `id_licenca`. |
| `ERD-Modular-Partes.md` | `outorgas.id_licenca` | `outorgas` usa `id_operador_origem`, `id_operador_destino`, `id_artefato`; não `id_licenca`. |
| `ERD-Modular-Partes.md` | `requerimentos.id_tipo`, `requerimentos.id_etapa` | `requerimentos` não tem esses campos; o fluxo é modelado por `tipo_licenca` textual e tabelas auxiliares separadas. |
| `ERD-Modular-Partes.md` | `condicionantes.id_requerimento` | `condicionantes` usa `id_licenca`. |
| `ERD-Operadores-Relacionamentos.md` | `processos.criado_por_usuario` | `processos` não documenta esse campo. |
| `ERD-Usuarios-Operadores.md` | `auditoria.id_usuario` | `auditoria` possui `usuario VARCHAR(255)`, não `id_usuario`. |

Impacto: **alto**. O risco é alguém usar o ERD como fonte de implementação ou entendimento do domínio e construir joins, APIs, filtros ou migrations sobre relacionamentos inexistentes.

Recomendação elegante: gerar os ERDs a partir do schema real ou, pelo menos, a partir de uma fonte declarativa única. Se o schema real ainda não existe, gere os diagramas a partir dos arquivos de tabela após um parser validar os campos e FKs.

---

### C4. Tipo SQL inválido `TIMESTAMPZ` persiste em `incidentes`

Arquivo afetado: `Tabelas/MODULO1/incidentes.md`

Campo afetado:

```text
data_ocorrencia TIMESTAMPZ NOT NULL
```

O tipo correto em PostgreSQL é:

```sql
TIMESTAMPTZ
```

ou, de forma expandida:

```sql
TIMESTAMP WITH TIME ZONE
```

Impacto: **crítico se usado para gerar DDL**, pois `TIMESTAMPZ` não é um tipo PostgreSQL válido.

Correção recomendada:

```sql
data_ocorrencia TIMESTAMPTZ NOT NULL
```

---

### C5. `rag_chunks` tem typo estrutural em `chunck_index`

Arquivo afetado: `Tabelas/CORE/rag_chunks.md`

Na estrutura, o campo aparece como:

```text
chunck_index
```

Mas nas regras de negócio o texto usa:

```text
chunk_index
chunks_index
```

Impacto: **alto**. Esse tipo de erro é especialmente perigoso em tabelas RAG porque o nome correto costuma ser esperado por pipelines de chunking, indexação e recuperação semântica. Se código Python/TypeScript usar `chunk_index` e o banco tiver `chunck_index`, haverá falhas ou mapeamentos manuais desnecessários.

Correção recomendada, se ainda não há dados em produção:

```sql
ALTER TABLE rag_chunks
RENAME COLUMN chunck_index TO chunk_index;
```

Depois, revisar as regras de negócio para usar apenas `chunk_index`.

---

## 5. Achados de severidade alta

### A1. `SOFT DELETE` aparece como se fosse restrição SQL

Arquivos afetados:

- `Tabelas/MODULO1/atividades_espaciais.md`
- `Tabelas/MODULO8/missoes.md`
- `Tabelas/MODULO8/tecnologias.md`

Exemplo:

```text
id_operador UUID NOT NULL, REFERENCES operadores(id), SOFT DELETE
```

`SOFT DELETE` não é uma cláusula SQL de FK. Como conceito arquitetural, ele deve aparecer em seção própria e ser implementado com `deletado_em`, views, policies ou filtros de aplicação.

Além disso, há contradição documental: nesses arquivos a estrutura fala em `SOFT DELETE`, mas as observações técnicas dizem que a exclusão de `operadores` remove registros via `ON DELETE CASCADE`. Isso não é a mesma coisa.

Caso especial: `atividades_espaciais` fala em `SOFT DELETE`, mas não possui campo `deletado_em`, ao contrário de `missoes` e `tecnologias`.

Correção recomendada:

- Decidir uma regra única por tabela: `ON DELETE RESTRICT`, `ON DELETE SET NULL`, `ON DELETE CASCADE` ou soft delete.
- Remover `SOFT DELETE` da coluna de restrições.
- Se soft delete for o objetivo, adicionar `deletado_em TIMESTAMPTZ` e evitar cascatas destrutivas em tabelas regulatórias.

---

### A2. Snippets SQL de relacionamento frequentemente divergem da estrutura

Vários blocos SQL em `## Relacionamentos` omitem `NOT NULL`, `ON DELETE CASCADE`, `ON DELETE SET NULL` ou divergem da própria tabela. Em alguns casos isso é só simplificação didática; em outros, contradiz a semântica.

Exemplos importantes:

| Arquivo | Campo | Estrutura | Snippet SQL |
|---|---|---|---|
| `Tabelas/MODULO2/peticionamentos.md` | `id_operador` | `REFERENCES operadores(id), ON DELETE SET NULL` | `id_operador UUID NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO3/artefatos.md` | `id_operador` | `REFERENCES operadores(id), ON DELETE SET NULL` | `id_operador UUID NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO1/licencas.md` | `id_operador` | `NOT NULL, REFERENCES operadores(id), ON DELETE CASCADE` | `id_operador UUID REFERENCES operadores(id)` |
| `Tabelas/MODULO9/condicionantes.md` | `id_licenca` | `NOT NULL, REFERENCES licencas(id), ON DELETE CASCADE` | `id_licenca UUID REFERENCES licencas(id)` |
| `Tabelas/MODULO9/etapa_fluxo.md` | `id_tipo_requerimento` | `REFERENCES tipo_requerimento(id), NOT NULL, ON DELETE CASCADE` | `id_tipo_requerimento UUID REFERENCES tipo_requerimento(id)` |

Impacto: **alto** se esses snippets forem copiados para migrations.

Recomendação: padronizar os snippets para refletirem exatamente a estrutura, ou escrever explicitamente que são exemplos parciais. Melhor ainda: substituir snippets manuais por DDL canônico gerado.

---

### A3. `notificacoes` ainda possui índice/PK documentado com erro

Arquivo afetado: `Tabelas/CORE/notificacoes.md`

A seção de índices mostra:

```text
notificacoesusuarios_roles(id) | PRIMARY KEY
```

O correto seria algo como:

```text
PK notificacoes(id) | PRIMARY KEY
```

Impacto: **médio-alto**. Não quebra o banco por si só, mas indica resíduo de cópia/cola e reduz confiabilidade da documentação em uma tabela central.

---

### A4. `fn_licenca_historico_status()` tem observação operacional incorreta

Arquivo afetado: `Functions/fn_licenca_historico_status().md`

A função é claramente uma trigger function (`RETURNS TRIGGER`, `RETURN NEW`) e é chamada por:

```sql
CREATE TRIGGER trg_licenca_historico_status
AFTER UPDATE ON licencas
FOR EACH ROW
EXECUTE FUNCTION fn_licenca_historico_status();
```

Mas a observação diz que ela normalmente seria executada via `pg_cron`, Supabase Scheduled Functions ou job scheduler externo. Isso parece comentário herdado da função de alerta de renovação, não dessa trigger.

Impacto: **alto para operação/documentação**, porque confunde a forma de execução de uma rotina de histórico.

Correção recomendada: mover essa observação para `fn_alertar_renovacao_licencas()` e documentar `fn_licenca_historico_status()` como trigger de atualização de `licencas.status`.

---

### A5. `requerimento_numeradores` foi introduzida, mas não está documentada como tabela

Arquivo afetado: `Functions/fn_numerar_requerimento().md`

A função agora propõe:

```sql
CREATE TABLE requerimento_numeradores(
    ano INTEGER PRIMARY KEY,
    ultimo INTEGER NOT NULL DEFAULT 0
);
```

A ideia é boa e corrige a fragilidade de `MAX(...) + 1`, mas a tabela auxiliar não aparece em `Tabelas/`, não tem RLS, não tem owner/permissões, não tem comentário de migration order e não aparece nos diagramas.

Impacto: **alto** se alguém aplicar a função sem criar a tabela auxiliar antes.

Recomendação: criar `Tabelas/MODULO9/requerimento_numeradores.md` ou `Tabelas/CORE/requerimento_numeradores.md`, dependendo da decisão arquitetural. Como é tabela técnica de numeração, eu tenderia a colocá-la em um módulo técnico/CORE ou no próprio MODULO9 com indicação de tabela interna.

DDL sugerido:

```sql
CREATE TABLE requerimento_numeradores (
    ano INTEGER PRIMARY KEY,
    ultimo INTEGER NOT NULL DEFAULT 0,
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE requerimento_numeradores
IS 'Tabela técnica para controle transacional de numeração anual de requerimentos.';
```

---

### A6. `legado.maptec_tecnologias` não confirma RLS, mas há função SECURITY DEFINER para leitura

Arquivos afetados:

- `Tabelas/LEGADO_MAPTEC/legado.maptec_tecnologias.md`
- `Functions/fn_listar_maptec(...).md`

A tabela legada diz apenas:

```text
Necessário validar diretamente no schema legado se o RLS está habilitado para a tabela.
```

Ao mesmo tempo, `fn_listar_maptec()` é uma RPC `SECURITY DEFINER` para leitura do legado. Isso pode ser adequado, mas precisa de governança explícita, porque funções `SECURITY DEFINER` podem concentrar privilégios sensíveis.

Recomendações:

- Especificar schema da função, por exemplo `public.fn_listar_maptec`, se o `GRANT` usa `public.fn_listar_maptec(TEXT)`.
- Garantir owner seguro da função, não um superusuário desnecessariamente amplo.
- Manter `REVOKE ALL FROM PUBLIC`.
- Documentar quais perfis podem executar a função.
- Se o legado contém dados confidenciais, criar filtros por confidencialidade/status ou restringir a função a perfis internos.

---

## 6. Achados de severidade média

### M1. Arquivo e diretórios vazios persistem

Arquivos vazios encontrados:

- `Arquitetura_RAG.md`

Diretórios vazios encontrados:

- `Scripts`
- `Seguran#U00e7a`
- `Views`

`Arquitetura_RAG.md` deveria ser especialmente preenchido, porque o modelo já possui `rag_chunks`, campos `embedding` e menções a pgvector.

Sugestão para `Arquitetura_RAG.md`:

- fontes indexáveis;
- estratégia de chunking;
- dimensão e modelo de embedding;
- métrica vetorial;
- política de reindexação;
- política de acesso/RLS;
- como `fonte` + `id_fonte` resolvem entidades;
- jobs responsáveis por extração, limpeza, embedding e atualização.

---

### M2. Pasta `Seguran#U00e7a` continua com encoding/nome quebrado

Diretório afetado: `Seguran#U00e7a/`

A grafia sugere que `Segurança` foi escapado/incorporado incorretamente. Em repositórios técnicos, eu recomendaria uma pasta ASCII estável:

```text
Seguranca/
```

ou, se o time aceitar acentos em path:

```text
Segurança/
```

---

### M3. Há um link Obsidian vazio

Arquivo afetado: `Diagramas/ERD-Atividades-Missoes-Partes.md`

Trecho:

```text
Este documento segue o mesmo modelo do [[]], mas foca em ...
```

Correção sugerida:

```text
Este documento segue o mesmo modelo do [[ERD-Modular-Partes]], mas foca em ...
```

---

### M4. `ON DELETE CASCADE` aparece em entidades regulatórias/históricas sensíveis

Há várias FKs com `ON DELETE CASCADE`, inclusive em documentos, licenças, histórico, fiscalizações, sanções, condicionantes e legado documental. Em alguns casos isso é correto para tabelas puramente associativas; em outros, pode ser perigoso para rastreabilidade regulatória.

Exemplos que merecem revisão de intenção:

- `documentos.id_operador ON DELETE CASCADE`;
- `licencas.id_operador ON DELETE CASCADE`;
- `licenca_historico.id_licenca ON DELETE CASCADE`;
- `licenca_sei.id_licenca ON DELETE CASCADE`;
- `condicionantes.id_licenca ON DELETE CASCADE`;
- `sancoes.id_operador ON DELETE CASCADE`;
- `legado.maptec_homologacoes.id_tecnologia ON DELETE CASCADE`.

Em domínio regulatório, costuma ser mais seguro usar `RESTRICT`, `NO ACTION`, `SET NULL` ou soft delete com trilha de auditoria, para preservar histórico.

---

### M5. Campos dinâmicos reduzem integridade referencial

Exemplos:

- `notificacoes.entidade_tipo` + `notificacoes.entidade_id`;
- `rag_chunks.fonte` + `rag_chunks.id_fonte`;
- `auditoria.tabela` + `auditoria.id_registro`.

Esse padrão é flexível, mas não garante FK real. Para manter elegância e segurança, recomendo uma dessas opções:

1. manter o modelo dinâmico, mas criar validações na aplicação e índices compostos;
2. criar tabelas especializadas para entidades de alto volume;
3. adicionar constraints auxiliares por tipo, quando possível;
4. documentar claramente quais valores de `entidade_tipo`/`fonte` são aceitos e como resolver cada um.

---

### M6. Índices auxiliares de FKs e filtros ainda são majoritariamente recomendações, não definição canônica

A documentação frequentemente diz “não foi identificado índice específico”. Isso é bom como observação, mas o schema final deveria definir os índices necessários.

Índices prioritários sugeridos:

```sql
CREATE INDEX idx_usuarios_roles_id_operador ON usuarios_roles(id_operador);
CREATE INDEX idx_documentos_id_operador ON documentos(id_operador);
CREATE INDEX idx_processos_id_operador ON processos(id_operador);
CREATE INDEX idx_notificacoes_id_usuario_lido ON notificacoes(id_usuario, lido);
CREATE INDEX idx_notificacoes_entidade ON notificacoes(entidade_tipo, entidade_id);
CREATE INDEX idx_licencas_id_operador ON licencas(id_operador);
CREATE INDEX idx_licencas_id_atividade ON licencas(id_atividade);
CREATE INDEX idx_requerimentos_id_operador ON requerimentos(id_operador);
CREATE INDEX idx_requerimentos_status ON requerimentos(status);
CREATE INDEX idx_requerimento_sei_id_requerimento ON requerimento_sei(id_requerimento);
CREATE INDEX idx_licenca_historico_id_licenca_criado ON licenca_historico(id_licenca, criado_em DESC);
```

Para `rag_chunks`, o índice HNSW já aparece documentado; recomendo também índice lógico para origem:

```sql
CREATE INDEX idx_rag_chunks_fonte_id_fonte
ON rag_chunks(fonte, id_fonte);
```

---

### M7. Defaults textuais em documentação podem virar SQL inválido se copiados literalmente

Alguns defaults textuais aparecem sem aspas, por exemplo `PENDENTE` ou `MIGRACAO_MAPTEC_V1`. Como valor documental isso é compreensível, mas em DDL PostgreSQL precisa de aspas:

```sql
status TEXT DEFAULT 'PENDENTE'
migrado_por TEXT DEFAULT 'MIGRACAO_MAPTEC_V1'
```

Recomendação: na tabela Markdown, padronizar defaults textuais com aspas para evitar erro ao copiar.

---

### M8. Pequenos typos e polimento textual

Ocorrências observadas:

- `embbedings OpenAI` deveria ser `embeddings OpenAI`; aparece em pelo menos `documentos.md`, `operadores.md` e `processos.md`.
- `chunck_index` deveria ser `chunk_index`.
- `negocioacoes` ainda aparece no ERD principal.
- `Seguran#U00e7a` deveria ser corrigido.
- `notificacoesusuarios_roles(id)` deveria ser `PK notificacoes(id)`.

Não são todos críticos, mas afetam a percepção de confiabilidade da documentação.

---

## 7. Validação positiva: o que está consistente

Apesar dos problemas acima, há boas notícias importantes:

1. **As FKs declaradas nas estruturas das tabelas resolvem para tabelas/colunas documentadas.** Não encontrei referência estrutural a tabela inexistente.
2. **Os triggers documentados apontam para tabelas e funções existentes.** Validei 24 triggers; as funções referenciadas existem na pasta `Functions/`.
3. **Os triggers de `atualizado_em` apontam para tabelas que possuem `atualizado_em`.** Isso está bem melhor do que a média de documentação manual.
4. **Links Obsidian não vazios estão resolvendo.** Foram 97 links não vazios e 0 quebrados.
5. **A separação modular está coerente.** CORE, LEGADO_MAPTEC e MODULO1...MODULO10 dão uma boa base de navegação.
6. **A documentação das regras de negócio é ampla.** A maioria das tabelas tem objetivo, constraints, relacionamentos, segurança, dependências, observações e regras.

---

## 8. Recomendações elegantes de melhoria

### 8.1. Separar “documentação” de “fonte canônica de schema”

Hoje os Markdown parecem misturar documentação, pseudo-DDL, snippets e diagnóstico. A forma mais elegante seria:

```text
/db
  /migrations
    001_core.sql
    002_modulo1.sql
    ...
  /policies
    core_rls.sql
    modulo1_rls.sql
  /functions
    fn_alertar_renovacao_licencas.sql
    fn_numerar_requerimento.sql
/docs
  /tabelas
  /diagramas
  /arquitetura
/scripts
  validate_docs.py
  generate_erd.py
```

A documentação poderia ser gerada ou validada contra migrations reais. Isso evita que ERD, Markdown e função evoluam separadamente.

---

### 8.2. Criar um linter de documentação de banco

Um script simples já capturaria os principais problemas desta rodada:

- tipos SQL inválidos;
- `INSERT` com colunas e valores em quantidades diferentes;
- links Obsidian vazios/quebrados;
- campos Mermaid inexistentes;
- arquivos vazios;
- diretórios com encoding quebrado;
- `SOFT DELETE` em coluna de restrições;
- snippets de relacionamento divergentes da estrutura;
- RLS habilitado sem policy;
- FKs sem índice auxiliar documentado.

Esse linter poderia rodar em CI a cada alteração do vault.

---

### 8.3. Gerar ERDs automaticamente

Os diagramas são úteis, mas hoje são a parte mais dessincronizada. Recomendo gerar Mermaid a partir de uma fonte única, por exemplo:

- migrations reais;
- introspecção do PostgreSQL;
- YAML declarativo de schema;
- parser dos próprios Markdown, desde que os Markdown sejam tratados como fonte canônica.

Uma regra prática: **se o campo não existe na estrutura da tabela, ele não deve aparecer no ERD**.

---

### 8.4. Tratar RLS como produto, não como rodapé

A documentação de RLS está muito repetitiva e ainda não operacional. Sugiro criar um documento central:

```text
Seguranca/RLS_Matriz_de_Acesso.md
```

Conteúdo recomendado:

- papéis do sistema;
- tabelas por domínio;
- permissões por operação;
- policies por tabela;
- exceções service role;
- regras para operadores externos;
- regras para usuários internos AEB;
- testes de acesso esperados.

Depois, cada tabela pode apenas referenciar a matriz.

---

### 8.5. Revisar estratégia de exclusão e retenção regulatória

Para tabelas regulatórias, eu evitaria cascatas destrutivas como padrão. Uma estratégia elegante seria:

- `operadores`: soft delete ou status institucional;
- `licencas`, `requerimentos`, `processos`, `sancoes`, `fiscalizacoes`: preservar histórico;
- tabelas associativas puras: `ON DELETE CASCADE` aceitável;
- documentos sensíveis: retenção definida por regra;
- auditoria/histórico: idealmente imutáveis.

---

### 8.6. Criar convenção de nomes

Sugestão:

- sempre `snake_case`;
- timestamps: `criado_em`, `atualizado_em`, `deletado_em`;
- booleanos: `st_...` ou verbos/adjetivos, mas com padrão único;
- chaves: `id_<entidade>`;
- tabelas associativas: `<entidade_a>_<entidade_b>` com PK composta;
- nomes de arquivos sem parênteses/reticências, por exemplo `fn_listar_maptec.md` em vez de `fn_listar_maptec(...).md`.

---

## 9. Plano de correção recomendado

### Prioridade 1 — bloquear erros que quebram execução

1. Corrigir `fn_alertar_renovacao_licencas()` adicionando `entidade_id` na lista de colunas.
2. Trocar `TIMESTAMPZ` por `TIMESTAMPTZ` em `incidentes`.
3. Corrigir `chunck_index` para `chunk_index`, ou documentar explicitamente se o erro já existe no banco e precisa de migração controlada.
4. Corrigir `negocioacoes` nos ERDs.

### Prioridade 2 — alinhar modelo visual e documentação

1. Regenerar ERDs a partir das tabelas documentadas.
2. Remover campos inexistentes dos diagramas.
3. Corrigir relações antigas baseadas em `id_processo` quando a tabela atual usa `id_operador` ou campos dinâmicos.

### Prioridade 3 — segurança e operação

1. Criar matriz RLS.
2. Implementar policies mínimas por papel.
3. Definir service role/jobs para rotinas automáticas.
4. Documentar permissões de `SECURITY DEFINER`.

### Prioridade 4 — acabamento técnico

1. Preencher `Arquitetura_RAG.md`.
2. Renomear `Seguran#U00e7a`.
3. Corrigir link `[[]]`.
4. Documentar `requerimento_numeradores`.
5. Padronizar defaults textuais com aspas.
6. Corrigir typos e copiar/colar residual.

---

## 10. Conclusão

A nova versão evoluiu muito e já transmite uma arquitetura mais madura. Os problemas anteriores mais ruidosos foram em grande parte resolvidos. O ponto mais importante agora é transformar essa documentação em **fonte confiável e verificável**.

O caminho mais eficiente é corrigir os poucos erros que quebrariam execução (`fn_alertar_renovacao_licencas`, `TIMESTAMPZ`, `chunck_index`), regenerar os ERDs com base no schema real, e tratar RLS como uma camada central de arquitetura.

Depois dessas correções, o material terá qualidade suficiente para servir como documentação técnica séria e como base para revisão de migrations, APIs e políticas de acesso.

---

# Apêndice A — Campos em ERDs que não existem nas tabelas

| Arquivo | Linha | Entidade/campo | Diagnóstico |
|---|---:|---|---|
| `ERD-Atividades-Missoes-Partes.md` | 22 | `outorgas.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Atividades-Missoes-Partes.md` | 26 | `garantias.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Atividades-Missoes-Partes.md` | 30 | `fiscalizacoes.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Atividades-Missoes-Partes.md` | 34 | `incidentes.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Atividades-Missoes-Partes.md` | 42 | `representacoes.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Atividades-Missoes-Partes.md` | 78 | `atividade_artefatos.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Atividades-Missoes-Partes.md` | 83 | `missao_artefatos.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Atividades-Missoes-Partes.md` | 88 | `missao_tecnologias.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Atividades-Missoes-Partes.md` | 138 | `condicionantes.id_requerimento` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 13 | `documentos.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 17 | `notificacoes.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 21 | `auditoria.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 25 | `rag_chunks.id_documento` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 46 | `garantias.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 50 | `fiscalizacoes.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 54 | `incidentes.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 73 | `peticionamentos.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 89 | `atividade_artefatos.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 106 | `outorgas.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 128 | `consultas_publicas.id_agenda` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 132 | `atos_normativos.id_consulta` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 136 | `relatorios_air_arr.id_ato` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 163 | `demandas_comunicacao.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 182 | `missao_artefatos.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 187 | `missao_tecnologias.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 206 | `requerimentos.id_tipo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 207 | `requerimentos.id_etapa` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 225 | `condicionantes.id_requerimento` | Campo não existe na tabela documentada correspondente. |
| `ERD-Modular-Partes.md` | 245 | `representacoes.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 19 | `processos.criado_por_usuario` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 24 | `atividades_espaciais.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 32 | `fiscalizacoes.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 36 | `incidentes.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 44 | `outorgas.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 48 | `garantias.id_licenca` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 53 | `missoes.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 62 | `atividade_artefatos.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 67 | `missao_artefatos.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Atividades-Missoes-Relacionamentos.md` | 72 | `missao_tecnologias.id` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Relacionamentos.md` | 36 | `processos.criado_por_usuario` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Relacionamentos.md` | 41 | `documentos.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Relacionamentos.md` | 46 | `notificacoes.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Relacionamentos.md` | 51 | `auditoria.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Relacionamentos.md` | 52 | `auditoria.id_usuario` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Relacionamentos.md` | 58 | `peticionamentos.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Operadores-Relacionamentos.md` | 63 | `demandas_comunicacao.id_processo` | Campo não existe na tabela documentada correspondente. |
| `ERD-Usuarios-Operadores.md` | 39 | `processos.criado_por_usuario` | Campo não existe na tabela documentada correspondente. |
| `ERD-Usuarios-Operadores.md` | 44 | `auditoria.id_usuario` | Campo não existe na tabela documentada correspondente. |
| `ERD-Usuarios-Operadores.md` | 45 | `auditoria.id_processo` | Campo não existe na tabela documentada correspondente. |

---

# Apêndice B — Entidades em ERDs sem tabela correspondente

| Arquivo | Linha | Entidade | Diagnóstico |
|---|---:|---|---|
| `ERD-Modular-Partes.md` | 150 | `negocioacoes` | Entidade não documentada como tabela; provavelmente deveria ser `negociacoes`. |

---

# Apêndice C — Divergências entre estrutura e snippets SQL de relacionamento

Esta lista inclui diferenças de `NOT NULL`, `ON DELETE` e `SOFT DELETE` entre a tabela de estrutura e os blocos SQL de relacionamento. Algumas podem ser simplificações didáticas; ainda assim, se o Markdown for usado como fonte de DDL, devem ser corrigidas.

| Arquivo | Campo | Diferença | Detalhe |
|---|---|---|---|
| `Tabelas/CORE/documentos.md` | `id_operador` | `ON DELETE CASCADE` | Estrutura: `NOT NULL, REFERENCES operadores(id), ON DELETE CASCADE` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/CORE/responsaveis_legais.md` | `id_operador` | `ON DELETE CASCADE` | Estrutura: `NOT NULL, REFERENCES operadores(id), ON DELETE CASCADE` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/CORE/usuarios_roles.md` | `id_operador` | `ON DELETE SET NULL` | Estrutura: `REFERENCES operadores(id), ON DELETE SET NULL` / SQL: `REFERENCES operadores(id)` |
| `Tabelas/LEGADO_MAPTEC/legado.maptec_homologacoes.md` | `id_tecnologia` | `ON DELETE CASCADE` | Estrutura: `REFERENCES legado.maptec_tecnologias(id), ON DELETE CASCADE` / SQL: `REFERENCES legado.maptec_tecnologias(id)` |
| `Tabelas/MODULO1/atividades_espaciais.md` | `id_operador` | `SOFT DELETE` | Estrutura: `NOT NULL, REFERENCES operadores(id), SOFT DELETE` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO1/fiscalizacoes.md` | `id_operador` | `ON DELETE CASCADE` | Estrutura: `NOT NULL, REFERENCES operadores(id), ON DELETE CASCADE` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO1/garantias.md` | `id_operador` | `ON DELETE CASCADE` | Estrutura: `REFERENCES operadores(id), ON DELETE CASCADE` / SQL: `REFERENCES operadores(id)` |
| `Tabelas/MODULO1/incidentes.md` | `id_operador` | `ON DELETE CASCADE` | Estrutura: `NOT NULL, REFERENCES operadores(id), ON DELETE CASCADE` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO1/licencas.md` | `id_operador` | `NOT NULL, ON DELETE CASCADE` | Estrutura: `NOT NULL, REFERENCES operadores(id), ON DELETE CASCADE` / SQL: `REFERENCES operadores(id)` |
| `Tabelas/MODULO1/sancoes.md` | `id_operador` | `ON DELETE CASCADE` | Estrutura: `NOT NULL, REFERENCES operadores(id), ON DELETE CASCADE` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO10/licenca_historico.md` | `id_licenca` | `ON DELETE CASCADE` | Estrutura: `REFERENCES licencas(id), ON DELETE CASCADE` / SQL: `REFERENCES licencas(id)` |
| `Tabelas/MODULO10/licenca_sei.md` | `id_licenca` | `ON DELETE CASCADE` | Estrutura: `REFERENCES licencas(id), ON DELETE CASCADE` / SQL: `REFERENCES licencas(id)` |
| `Tabelas/MODULO2/peticionamentos.md` | `id_operador` | `NOT NULL, ON DELETE SET NULL` | Estrutura: `REFERENCES operadores(id), ON DELETE SET NULL` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO3/artefatos.md` | `id_operador` | `NOT NULL, ON DELETE SET NULL` | Estrutura: `REFERENCES operadores(id), ON DELETE SET NULL` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO3/atividade_artefatos.md` | `id_atividade` | `ON DELETE CASCADE` | Estrutura: `PRIMARY KEY, REFERENCES atividades_espaciais(id), ON DELETE CASCADE` / SQL: `REFERENCES atividades_espaciais(id)` |
| `Tabelas/MODULO3/atividade_artefatos.md` | `id_artefato` | `ON DELETE CASCADE` | Estrutura: `PRIMARY KEY, REFERENCES artefatos(id), ON DELETE CASCADE` / SQL: `REFERENCES artefatos(id)` |
| `Tabelas/MODULO4/licenca_fases.md` | `id_licenca` | `ON DELETE CASCADE` | Estrutura: `NOT NULL, REFERENCES licencas(id), ON DELETE CASCADE` / SQL: `NOT NULL REFERENCES licencas(id)` |
| `Tabelas/MODULO6/negociacoes.md` | `acordo_id` | `ON DELETE CASCADE` | Estrutura: `REFERENCES acordos_internacionais(id), ON DELETE CASCADE` / SQL: `REFERENCES acordos_internacionais(id)` |
| `Tabelas/MODULO8/missoes.md` | `id_operador` | `SOFT DELETE` | Estrutura: `NOT NULL, REFERENCES operadores(id), SOFT DELETE` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO8/tecnologias.md` | `id_operador` | `SOFT DELETE` | Estrutura: `NOT NULL, REFERENCES operadores(id), SOFT DELETE` / SQL: `NOT NULL REFERENCES operadores(id)` |
| `Tabelas/MODULO9/condicionantes.md` | `id_licenca` | `NOT NULL, ON DELETE CASCADE` | Estrutura: `NOT NULL, REFERENCES licencas(id), ON DELETE CASCADE` / SQL: `REFERENCES licencas(id)` |
| `Tabelas/MODULO9/etapa_fluxo.md` | `id_tipo_requerimento` | `NOT NULL, ON DELETE CASCADE` | Estrutura: `REFERENCES tipo_requerimento(id), NOT NULL, ON DELETE CASCADE` / SQL: `REFERENCES tipo_requerimento(id)` |
| `Tabelas/MODULO9/perfil_etapa_fluxo.md` | `id_etapa` | `NOT NULL, ON DELETE CASCADE` | Estrutura: `REFERENCES etapa_fluxo(id), NOT NULL, ON DELETE CASCADE` / SQL: `REFERENCES etapa_fluxo(id)` |
| `Tabelas/MODULO9/requerimento_interessados.md` | `id_requerimento` | `NOT NULL, ON DELETE CASCADE` | Estrutura: `REFERENCES requerimentos(id), NOT NULL, ON DELETE CASCADE` / SQL: `REFERENCES requerimentos(id)` |
| `Tabelas/MODULO9/requerimento_sei.md` | `id_requerimento` | `NOT NULL, ON DELETE CASCADE` | Estrutura: `REFERENCES requerimentos(id), NOT NULL, ON DELETE CASCADE` / SQL: `REFERENCES requerimentos(id)` |
| `Tabelas/MODULO9/requerimentos.md` | `id_operador` | `NOT NULL` | Estrutura: `REFERENCES operadores(id), NOT NULL` / SQL: `REFERENCES operadores(id)` |

---

# Apêndice D — Arquivos e diretórios vazios

## Arquivos vazios

- `Arquitetura_RAG.md`

## Diretórios vazios

- `Scripts`
- `Seguran#U00e7a`
- `Views`

