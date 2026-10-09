# GitHub Secrets na pipeline

Atividade prática de GitHub Secrets (FAP DevOps, Turma 5).
Fork de carlhenriquex/github_secrets_demo. Aluno: Luiz Filipe.

## 1. Problema encontrado

O workflow tinha a informação sensível escrita direto no arquivo `.github/workflows/main.yml`:

    env:
      MENSAGEM_SECRETA: "Minha mensagem secreta"

Observação: o enunciado chama o arquivo de `pipeline.yml`, mas no repositório ele se chama `main.yml`. Mantive o nome original.

**Onde o segredo está armazenado?**
No próprio arquivo do workflow, em texto puro, dentro do repositório e do histórico do Git.

**Quem poderia visualizar essa informação?**
Qualquer pessoa com acesso de leitura ao repositório. Como ele é público, qualquer pessoa na internet, além de quem tiver um clone ou fork.

**Por que essa prática não é recomendada?**
Porque o arquivo é versionado e compartilhado. O segredo vai junto para todo clone e fork, fica no histórico mesmo depois de apagado e não dá para controlar quem leu.

**O que poderia acontecer se fosse uma senha ou um token real?**
Quem lesse o repositório poderia usar a credencial para acessar o banco, a API ou o serviço de nuvem em nome do projeto. Existem robôs que varrem repositórios públicos atrás de credenciais, então o uso indevido costuma ser rápido. A única saída seria trocar a credencial.

## 2. Solução

Secret utilizado: `MENSAGEM_SECRETA`.

Criei o secret no meu fork com `gh secret set MENSAGEM_SECRETA`, que faz o mesmo que o caminho Settings > Secrets and variables > Actions > New repository secret.

No workflow, troquei o valor fixo pela referência ao secret:

    env:
      MENSAGEM_SECRETA: ${{ secrets.MENSAGEM_SECRETA }}

Dois detalhes do fork: os secrets do repositório original não são copiados, então precisei criar o meu, e o GitHub Actions vem desligado, então precisei habilitar na aba Actions.

## 3. Como o secret chega até a aplicação

1. O valor fica guardado criptografado nas configurações do repositório no GitHub.
2. Quando a pipeline roda, o GitHub Actions resolve `${{ secrets.MENSAGEM_SECRETA }}`.
3. O bloco `env` do step entrega esse valor como a variável de ambiente `MENSAGEM_SECRETA`, só para aquele step.
4. O `server.js` lê a variável com `process.env.MENSAGEM_SECRETA`.

Resumo: Secret, GitHub, GitHub Actions, variável de ambiente, aplicação. O código só conhece o nome da variável, nunca o valor.

## 4. Evidência de execução

Execução: https://github.com/Fililpe/github_secrets_demo/actions/runs/37995050626

Trecho do log (completo em `evidencias/pipeline-execucao.txt`):

    MENSAGEM_SECRETA: ***
    Secret recebido pela aplicação com sucesso.
    Quantidade de caracteres: 12

O que isso confirma:

- A aplicação continua recebendo a mensagem (12 caracteres, o tamanho da frase que cadastrei).
- O `main.yml` não tem mais o valor real.
- O valor não aparece no log. O GitHub troca por `***` e a aplicação só mostra o tamanho.

Um ponto de atenção: o valor antigo ("Minha mensagem secreta") continua no histórico do Git, no commit anterior. Aqui não tem problema porque era só um exemplo, mas com uma senha real seria preciso considerar vazada e trocar.

## 5. Questões

**1. Qual a diferença entre armazenar o valor no arquivo da pipeline e no GitHub Secrets?**
No arquivo, o valor é versionado e fica visível para todo mundo que lê o repositório. No GitHub Secrets, ele fica criptografado fora do código, só é entregue para a pipeline na hora da execução e é mascarado nos logs. Depois de salvo, nem eu consigo ler de volta, só substituir.

**2. Onde o valor do Secret fica configurado?**
Nas configurações do repositório no GitHub, em Settings > Secrets and variables > Actions, na lista Repository secrets.

**3. O valor do Secret deve ser colocado no código da aplicação?**
Não. O código só referencia o nome da variável de ambiente.

**4. Qual a função de `process.env.MENSAGEM_SECRETA`?**
Ler a variável de ambiente `MENSAGEM_SECRETA` no processo do Node. É por ela que a aplicação recebe o valor sem que ele esteja escrito no código. O mesmo código funciona na pipeline, na minha máquina ou em um servidor, cada um com o seu valor.

**5. Por que usar Secrets é importante em uma pipeline de CI/CD?**
Porque a pipeline precisa de credenciais para fazer deploy, publicar imagens e acessar bancos e APIs, e o arquivo dela fica no repositório. Com Secrets dá para usar essas credenciais sem expor, trocar o valor sem mexer no código e limitar quem tem acesso.

**6. Por que seria inadequado colocar uma senha real de banco no arquivo `.yml`?**
Porque qualquer pessoa com acesso ao repositório teria acesso ao banco. A senha ficaria no histórico para sempre, seria copiada em todo clone e fork, e a única correção seria trocar a senha.

Limite que vale conhecer: o mascaramento do log só esconde o valor exato. Se o código transformar o segredo antes de imprimir, ele pode vazar. Por isso a regra é nunca imprimir segredo, e não confiar só na máscara.
