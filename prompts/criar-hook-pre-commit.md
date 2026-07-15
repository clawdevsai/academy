# Prompt para criação do gate de pré-push (hook + skill)

> ⚠️ **Ação esperada agora:** criar os dois artefatos abaixo (o hook e a skill) exatamente como especificado. Não execute a revisão agora — apenas crie os arquivos.

> **Nota técnica importante (por isso este prompt tem duas partes, não uma só):** hooks do Claude Code são scripts determinísticos — recebem dados via stdin e decidem bloquear ou não através do código de saída (`exit 2` bloqueia, `exit 0`/`exit 1` deixam passar). Um hook **não consegue** invocar subagentes com raciocínio, avaliar um veredito qualitativo ("⚠️ APROVADO COM RESSALVAS") ou conduzir uma conversa de Sim/Não com o usuário — isso é trabalho de um agente com LLM, não de um script de shell. Por isso, o gate completo precisa de duas peças que trabalham juntas:
>
> 1. **Um hook real** (`PreToolUse` em `Bash`) que bloqueia qualquer `git push` executado diretamente, direcionando o usuário para o comando correto.
> 2. **Uma skill** (`/pre-push-review`), que é quem de fato orquestra os três subagentes de checagem (`revisor`, `qa`, `cyber-sec`), interpreta os vereditos, conversa com o usuário quando algo bloqueia, aciona o `bug-fix` para corrigir os achados quando autorizado, e — só quando tudo estiver aprovado — executa o `git push` de verdade.

Este prompt assume que os agentes `revisor`, `qa`, `cyber-sec` e `bug-fix` já foram criados (arquivos correspondentes em `prompts/criar-agente-*.md`).

---

## Parte 1 — Hook: bloquear `git push` direto

Adicione (ou crie, se não existir) a entrada abaixo em `.claude/settings.json`, na seção `hooks` → `PreToolUse`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git push*)",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/scripts/bloquear-push-direto.sh"
          }
        ]
      }
    ]
  }
}
```

Crie o script `.claude/scripts/bloquear-push-direto.sh`:

```bash
#!/bin/bash
# .claude/scripts/bloquear-push-direto.sh
# Bloqueia git push executado diretamente pelo Bash.
# O push só deve acontecer através da skill /pre-push-review,
# que roda o gate de qualidade/QA/segurança antes de liberar.

echo "Bloqueado: use o comando /pre-push-review para rodar o gate de qualidade, QA e segurança antes do push." >&2
exit 2
```

Torne o script executável (`chmod +x .claude/scripts/bloquear-push-direto.sh`).

**Atenção ao código de saída:** use exatamente `exit 2`. `exit 1` não bloqueia a ação — apenas registra um aviso e o Claude Code segue em frente. Esse é o erro mais comum ao configurar hooks de segurança.

---

## Parte 2 — Skill: `/pre-push-review`

Gere esta skill usando o plugin oficial **skill-creator** (https://claude.com/plugins/skill-creator), em vez de escrever `SKILL.md` à mão — ele garante frontmatter válido e estrutura de progressive disclosure corretas.

Se o plugin não estiver instalado, instale primeiro:

```text
claude plugin install skill-creator@claude-plugins-official
```

Use a skill `skill-creator` para criar a skill `pre-push-review`, fornecendo a ela exatamente o nome, a `description` e o corpo abaixo como especificação — não deixe o skill-creator improvisar conteúdo além do que está definido aqui. Se o plugin não estiver disponível no ambiente (offline, sem permissão), caia no modo manual: crie o arquivo `.claude/skills/pre-push-review/SKILL.md` diretamente com o frontmatter e o conteúdo abaixo.

Frontmatter e corpo a fornecer ao skill-creator (ou a escrever manualmente em `.claude/skills/pre-push-review/SKILL.md`):

```yaml
---
name: pre-push-review
description: >
  Roda o gate obrigatório de qualidade antes de um git push: aciona revisor,
  qa e cyber-sec em sequência, consolida os vereditos, aciona o bug-fix para
  corrigir achados quando autorizado, e só libera o push quando todos
  aprovarem. Invocado manualmente pelo usuário com /pre-push-review.
---
```

Corpo da skill (system prompt executado quando `/pre-push-review` é chamado):

```markdown
Você vai orquestrar o gate de pré-push. Siga esta sequência obrigatoriamente:

## 1. Executar os três agentes em sequência

Acione, nesta ordem, via delegação de subagente:

1. `revisor` — revisa as alterações que serão enviadas (qualidade, arquitetura, bugs).
2. `qa` — valida e executa a suíte de testes relevante às alterações.
3. `cyber-sec` — roda a revisão de segurança (vulnerabilidades, segredos expostos, scanners configurados no projeto).

Cada agente deve retornar um veredito explícito.

## 2. Política de aprovação

O push só pode prosseguir quando TODOS os agentes retornarem:

- ✅ aprovação plena, ou
- ⚠️ aprovação com ressalvas — desde que não haja nenhum achado Crítico ou Alto.

Bloquear imediatamente se qualquer agente retornar:

- ❌ reprovação (qualquer variação: "requer mudanças", "implementação incompleta", "suíte insuficiente", "requer correções de segurança");
- qualquer vulnerabilidade Crítica ou Alta;
- qualquer teste obrigatório falhando;
- erro de lint, type-check ou build definidos como obrigatórios pelo projeto.

## 3. Relatório consolidado

Antes de decidir, apresente ao usuário um resumo com:

- resultado do Code Review;
- resultado do QA;
- resultado da Segurança;
- quantidade de problemas por severidade;
- arquivos afetados;
- recomendações de correção.

## 4. Se aprovado

Execute o `git push` real via Bash e confirme ao usuário.

## 5. Se bloqueado

1. Explique claramente o motivo do bloqueio e liste os problemas.
2. Pergunte ao usuário: "Foram encontrados problemas que impedem o push. Deseja que eu acione o bug-fix para corrigir automaticamente os que podem ser resolvidos com segurança?"
3. Se o usuário responder **sim**: acione o subagente `bug-fix`, passando a ele o relatório consolidado (achados, arquivos, testes falhando) como evidência real — nunca uma descrição vaga. O `bug-fix` diagnostica a causa raiz e aplica a menor correção possível, seguindo seu próprio fluxo de testes.
4. Depois que o `bug-fix` concluir, repita os passos 1–3 do zero (revisor, qa, cyber-sec de novo), e só faça o push se o novo ciclo aprovar integralmente.
5. Se o `bug-fix` não conseguir resolver algo, ou se o usuário responder **não**: mantenha o push bloqueado e encerre exibindo o relatório completo (incluindo o que o `bug-fix` já tentou, se for o caso) para correção manual.

## Regras inegociáveis

- Nenhum dos três agentes de checagem (revisor, qa, cyber-sec) pode ser pulado ou desabilitado.
- A correção de achados nunca é feita pela skill diretamente — sempre delegada ao subagente `bug-fix`, que segue seu próprio processo de causa raiz e validação por testes.
- Nunca faça o push se houver achado Crítico ou Alto pendente.
- Use sempre as ferramentas de lint, teste, scanner e type-check já configuradas no projeto (não invente novas).
- Seja transparente: mostre todas as verificações e seus resultados, mesmo quando aprovado.
```

---

## Após criar os arquivos (ação da tarefa atual)

Confirme:
1. O caminho e conteúdo de `.claude/settings.json` (seção hooks) e do script `.claude/scripts/bloquear-push-direto.sh`.
2. O caminho do arquivo `.claude/skills/pre-push-review/SKILL.md`, e se foi gerado via `skill-creator` ou manualmente (fallback).
3. Lembre o usuário de rodar `chmod +x .claude/scripts/bloquear-push-direto.sh` caso o ambiente não tenha aplicado a permissão automaticamente.
