## Automatizar Backup e Restore do PostgreSQL com Docker + EasyPanel
Tutorial para fazer um backups automáticos da sua base de dados Postgres utilizando o EasyPanel básico

### 📋 Pré-requisitos
- Docker instalado e rodando
- EasyPanel configurado
- Acesso ao terminal do servidor

### 🔄 Backup Automatizado

#### 1. Identificar o Container PostgreSQL
Caminho para o console no EasyPanel:
HOME->Configurações->Geral->Servidor->Console

Execute o comando para encontrar o nome do container:
```bash
docker ps | grep 'postgres'
```
###### Saída esperada: {{NOME_DO_CONTAINER}} (ex: teste.1.sgpeeddsde0snib6v7m9p79m)

#### 2. Criar Script de Backup
Crie o arquivo de script:

```
mkdir -p /var/scripts/
nano /var/scripts/backup_{{NOME_DO_BANCO}}.sh
```
Cole o seguinte conteúdo, ajustando as variáveis:

```
#!/bin/bash

# --- Configuracoes ---
# Preencha com as informações do seu ambiente
CONTAINER_NAME="{{NOME_DO_CONTAINER}}"
DB_NAME="{{NOME_DO_BANCO}}"
PG_USER="{{USUARIO_POSTGRES}}"
BACKUP_DIR="{{DIRETORIO_BACKUP}}"

# --- Configuracoes Automáticas ---
TIMESTAMP=$(date +%Y-%m-%d_%H%M)
BACKUP_FILE="$BACKUP_DIR/db_backup_$TIMESTAMP.sql.gz"

# --- Execucao ---
mkdir -p $BACKUP_DIR

echo "Iniciando backup para $BACKUP_FILE..."

docker exec $CONTAINER_NAME pg_dump -U $PG_USER --clean $DB_NAME | gzip > $BACKUP_FILE

if [ $? -eq 0 ]; then
    echo "✅ Backup concluido com sucesso: $BACKUP_FILE"
else
    echo "❌ Erro ao executar o backup."
fi

# Limpeza de backups antigos (últimos 7 dias)
find $BACKUP_DIR -type f -prune -mtime +7 -exec rm {} \;
echo "Limpeza de backups com mais de 7 dias concluida."
```

### Variáveis a serem preenchidas:

* {{NOME_DO_CONTAINER}}: Nome do container PostgreSQL (obtido no passo 1)

* {{NOME_DO_BANCO}}: Nome do banco de dados

* {{USUARIO_POSTGRES}}: Usuário do PostgreSQL (geralmente 'postgres')

* {{DIRETORIO_BACKUP}}: Diretório onde salvar backups (ex: /etc/easypanel/backups/{{NOME_DO_BANCO}}/dados/)

#### 3. Permissões e Agendamento
Dê permissão de execução:

```
chmod +x /var/scripts/backup_{{NOME_DO_BANCO}}.sh
```
Agende no cron:

```
crontab -e
```
\
Adicione a linha (ajuste o horário conforme necessidade):

ex.: backup diário às 23:00
```
0 23 * * * /var/scripts/backup_{{NOME_DO_BANCO}}.sh >> /var/scripts/backup_bd.log 2>&1
```
### 🔙 Preparar o Restore do Backup
1. Criar Script de Restore
```
nano /var/scripts/restore_{{NOME_DO_BANCO}}.sh
```
Cole o script abaixo, ajustando as variáveis:

```
#!/bin/bash

# --- CONFIGURACOES FIXAS ---
CONTAINER_NAME="{{NOME_DO_CONTAINER}}"
PG_USER="{{USUARIO_POSTGRES}}"
PG_DB="{{NOME_DO_BANCO}}"
BACKUP_DIR="{{DIRETORIO_BACKUP}}"

# --- CONFIGURACOES AUTOMATICAS ---
HOST_TMP_DIR="/tmp/pg_restore"
CONTAINER_TMP_FILE="/tmp/backup_restaurar.sql"

# --- FUNCAO PRINCIPAL ---
restaurar_banco() {
    echo "--- RESTAURADOR POSTGRESQL ---"
    echo "Procurando arquivos de backup em: $BACKUP_DIR"
    
    # Lista arquivos disponíveis
    ls -1 "$BACKUP_DIR"/*.sql.gz | xargs -n 1 basename
    echo "--------------------------------"
    read -p "Digite o NOME COMPLETO do arquivo .sql.gz a restaurar: " BACKUP_FILENAME

    FULL_BACKUP_PATH="$BACKUP_DIR/$BACKUP_FILENAME"

    if [[ ! -f "$FULL_BACKUP_PATH" ]]; then
        echo "❌ Erro: Arquivo '$FULL_BACKUP_PATH' não encontrado."
        exit 1
    fi

    echo ""
    echo "✅ Backup encontrado: $FULL_BACKUP_PATH"
    echo "--- Iniciando o processo de Restore ---"

    # Preparar ambiente
    mkdir -p $HOST_TMP_DIR
    HOST_SQL_PATH="$HOST_TMP_DIR/restore.sql"
    
    # Descompactar
    echo "-> 1/3. Descompactando $BACKUP_FILENAME..."
    gunzip -c "$FULL_BACKUP_PATH" > "$HOST_SQL_PATH"
    if [ $? -ne 0 ]; then
        echo "❌ Erro na descompactação."
        rm -rf $HOST_TMP_DIR
        exit 1
    fi

    # Copiar para container
    echo "-> 2/3. Copiando para o container $CONTAINER_NAME..."
    docker cp "$HOST_SQL_PATH" "$CONTAINER_NAME:$CONTAINER_TMP_FILE"
    if [ $? -ne 0 ]; then
        echo "❌ Erro na cópia para o Docker."
        rm -rf $HOST_TMP_DIR
        exit 1
    fi

    # Recriar banco e restaurar
    echo "-> 3.1/3. Recriando o banco de dados $PG_DB..."
    docker exec -i $CONTAINER_NAME dropdb -U $PG_USER $PG_DB 2>/dev/null
    docker exec -i $CONTAINER_NAME createdb -U $PG_USER $PG_DB

    echo "-> 3.2/3. Executando RESTORE no banco $PG_DB..."
    docker exec -i $CONTAINER_NAME psql -U $PG_USER -d $PG_DB -f $CONTAINER_TMP_FILE

    if [ $? -eq 0 ]; then
        echo "🎉 RESTORE CONCLUÍDO COM SUCESSO! O banco $PG_DB foi restaurado."
    else
        echo "🛑 ERRO: Ocorreu um erro durante o restore."
    fi

    # Limpeza
    echo "-> Limpando arquivos temporários..."
    docker exec $CONTAINER_NAME rm -f $CONTAINER_TMP_FILE
    rm -rf $HOST_TMP_DIR
    echo "Limpeza concluída."
}

restaurar_banco
```
\
2. Permissões de Execução
```
chmod +x /var/scripts/restore_{{NOME_DO_BANCO}}.sh
```

### Como Usar o Restore
1. Listar Backups Disponíveis
```
cd {{DIRETORIO_BACKUP}} ls
```
2. Executar Restore
```
./var/scripts/restore_{{NOME_DO_BANCO}}.sh
```

O script irá:
* Listar todos os backups disponíveis
* Pedir o nome completo do arquivo para restore
* Executar todo o processo automaticamente

### 📝 Notas Importantes
* Backups antigos são automaticamente removidos após 7 dias, opcional.
* Teste o restore em ambiente de desenvolvimento antes de usar em produção
* Verifique os logs em /var/scripts/backup_bd.log para troubleshooting
* Mantenha as credenciais seguras e ajuste as permissões dos arquivos

#### 🆘 Troubleshooting
* Container não encontrado: Verifique se o Docker está rodando e o nome do container
* Erro de permissão: Certifique-se que o usuário tem acesso ao diretório de backups
* Backup vazio: Verifique se o banco de dados existe e tem dados
