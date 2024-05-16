## Arquitetura
![architecture_diagram](apolo.drawio.svg)

## Modelagem de Dados
![db_diagram](diagram.png)
<center>[db_diagram_url](https://dbdiagram.io/d/Apolo-65df568dcd45b569fb2165f3)</center>

### Como levar os dados do JIRA ao Superset?
1. Cadastrar no JIRA um Webhook de eventos de task *(Marcar todos os eventos: Create, Update, Delete)*. **Obs: Nunca devemos deletar uma task! Tasks excluidas também serão contabilizadas nas métricas, caso o jira informe o delete de uma task, mapeie como excluido em um booleano por exemplo, mas mantenha o registro!**
2. Dentro do Backend do Apolo, desenvolver um endpoint para receber esse callback. O comportamento desse endpoint deve ser um upsert na tabela de *ap_jira_task*, sendo a única PK da tabela e a referência do Upsert a TAG do Jira. Ex: Upsert no registro utilizando como integridade relacional o número da Task. **MAG-21513 -> upsert em jira_task_id = 21513**.
3. Com os dados na tabela *ap_jira_task*, realizaremos o envio para o S3 através da rede da AWS. Uma vez ao dia, em momento de baixa demanda (preferêncialmente da 00h - 03h) executar um script na EC2 em que o banco se encontra, enviando o csv (sempre com o mesmo nome, importante para o mapeamento no data lake) para o [esse bucket no s3.](https://sa-east-1.console.aws.amazon.com/s3/upload/datainteligence-apolo?region=sa-east-1&bucketType=general)

*Observar que a EC2 do Banco de Dados se encontra na conta Magazord e o bucket de destino na conta Data Inteligence, sendo necessário realizar uma configuração prévia de permissões para que o envio ao s3 seja possível*

----------

#### Configuração de Acesso entre Contas AWS para Transferência de Arquivos entre EC2 e S3
1. Crie uma Função IAM na Conta da Instância EC2 PostgreSQL:

    Acesse o console IAM na AWS Management Console.
    Crie uma nova função IAM com permissões para acessar o S3. Essa função será assumida pela instância EC2.
    Anexe uma política à função que permita o acesso ao bucket S3 específico na outra conta AWS. Você pode criar uma política personalizada para esse fim.

2. Modifique a Função IAM Anexada à Instância EC2:

    Anexe a função IAM criada no passo 1 à instância EC2 que executa o PostgreSQL. Você pode fazer isso parando a instância, modificando sua configuração de função IAM e depois reiniciando-a.

3. Atualize o Script com o ARN da Função IAM Correta:

    No script, em vez de usar chaves de acesso ou credenciais de usuário IAM, você usará um ARN de função IAM que a instância EC2 assumirá. Essa função deve ter permissões para acessar o S3.

4. Configure a Política do Bucket na Conta do Bucket S3:

    Na conta AWS onde o bucket S3 está localizado, você precisa configurar uma política do bucket que permita o acesso a partir da função IAM na conta EC2 PostgreSQL. Esta política deve conceder as permissões necessárias para fazer upload de arquivos no bucket.

5. Atualize o Script com a URL Correta do Bucket S3:

    Atualize o script com a URL correta do bucket S3, considerando o ARN do bucket e a política de bucket configurada.

----------

#### backup.sh - Script para gerar o dump da tabela e realizar o envio para o S3
``` bash
#!/bin/bash
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Conexão com o Banco
DB_CONNECTION="postgresql://apolo:EgKhLAec@magazord-postgresql.vpc-amazon.magazord.com.br:5432/apolo"

# Nome da tabela a ser exportada
TABLE_NAME="ap_jira_tasks"

# Defina o nome do arquivo atual e do arquivo de histórico
CURRENT_FILE="ap_jira_tasks_dump.csv"
OLD_FILE="ap_jira_tasks_dump_old.csv"

# Renomear o arquivo atual para o arquivo de histórico, se existir
if [ -f "$CURRENT_FILE" ]; then
    mv "$CURRENT_FILE" "$OLD_FILE"
    echo "Arquivo atual renomeado para $OLD_FILE."
fi

# Gerar o novo CSV
psql "$DB_CONNECTION" -c "COPY (SELECT * FROM $TABLE_NAME) TO STDOUT WITH CSV HEADER" > "$CURRENT_FILE"

# Verifique se o arquivo CSV foi gerado com sucesso
if [ -f "$CURRENT_FILE" ]; then
    # Faça upload do novo arquivo CSV para o S3
    aws s3 cp "$CURRENT_FILE" "s3://$S3_BUCKET/"

    # Verifique se o upload foi bem-sucedido
    if [ $? -eq 0 ]; then
        echo "Arquivo CSV enviado com sucesso para o S3."
        
        # Exclui o arquivo CSV local
        rm "$CURRENT_FILE"
    fi
fi

```

----------

#### Crie um crontab para executar o comando de forma recorrente

``` bash

# Dê permissão de execução
$ chmod +x /caminho/ate/o/script/backup.sh

$ crontab -e 
$ 0 0 * * * /caminho/ate/o/script/backup.sh


```
