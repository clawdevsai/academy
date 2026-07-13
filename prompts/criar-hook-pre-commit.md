Atualize a configuração dos Hooks do Claude Code para adicionar um gate obrigatório antes de qualquer `git push` para o repositório remoto.

## Hook: Pre-Push (Obrigatório)

Sempre que um `git push` for executado, o Claude Code deve interromper temporariamente o processo de envio e executar, obrigatoriamente e nesta ordem, os seguintes subagentes:

1. `lw-code-reviewer`
2. `lw-qa-engineer`
3. `lw-cybersecurity-engineer`

## Responsabilidade de cada agente

### lw-code-reviewer
- Revisar todas as alterações que serão enviadas.
- Validar qualidade do código.
- Verificar arquitetura, boas práticas, manutenibilidade e possíveis bugs.
- Emitir um veredito final.

### lw-qa-engineer
- Validar os testes existentes.
- Executar a suíte de testes apropriada.
- Verificar cobertura das regras de negócio afetadas.
- Garantir que não existam testes falhando, flaky ou ignorados sem justificativa.

### lw-cybersecurity-engineer
- Executar a revisão de segurança.
- Verificar vulnerabilidades conhecidas.
- Executar scanners de segurança configurados no projeto (Bandit, Semgrep, pip-audit, osv-scanner ou equivalentes).
- Identificar segredos expostos, configurações inseguras e riscos de segurança.

---

## Política de Aprovação

O `git push` somente poderá continuar quando TODOS os agentes emitirem um resultado de aprovação.

Resultados aceitos:

- ✅ APROVADO
- ✅ APROVADO COM RESSALVAS (desde que não existam achados Críticos ou Altos)

O push deve ser bloqueado imediatamente caso qualquer agente retorne:

- ❌ REQUER MUDANÇAS
- ❌ IMPLEMENTAÇÃO INCOMPLETA
- ❌ SUÍTE INSUFICIENTE
- ❌ REQUER CORREÇÕES DE SEGURANÇA
- qualquer vulnerabilidade **Crítica** ou **Alta**
- qualquer teste obrigatório falhando
- erro de lint, type-check ou build configurados como obrigatórios pelo projeto

---

## Relatório Consolidado

Antes de permitir o push, apresentar um resumo contendo:

- Resultado do Code Review
- Resultado do QA
- Resultado da Segurança
- Quantidade de problemas encontrados por severidade
- Arquivos afetados
- Recomendações de correção

---

## Fluxo quando houver bloqueio

Caso o push seja bloqueado:

1. Informar claramente o motivo do bloqueio.
2. Listar todos os problemas encontrados.
3. Perguntar ao usuário:

> Foram encontrados problemas que impedem o envio para o repositório remoto. Deseja que eu corrija automaticamente todos os problemas que podem ser resolvidos com segurança?

Se o usuário responder **Sim**:

1. Executar automaticamente as correções permitidas.
2. Reexecutar os três agentes (`lw-code-reviewer`, `lw-qa-engineer` e `lw-cybersecurity-engineer`).
3. Gerar um novo relatório consolidado.
4. Somente liberar o `git push` se todos os gates forem aprovados.

Se o usuário responder **Não**:

- Manter o push bloqueado.
- Exibir o relatório completo para correção manual.

---

## Regras Importantes

- Este gate é obrigatório para todo `git push` remoto.
- Nenhum agente pode ser ignorado ou desabilitado durante o processo.
- O push nunca deve ser executado caso existam problemas Críticos ou Altos.
- Todas as validações devem utilizar as configurações do próprio projeto (linters, testes, scanners, type-checkers e convenções internas).
- O processo deve ser totalmente transparente, exibindo todas as verificações realizadas e seus respectivos resultados.