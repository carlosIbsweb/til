
# Guia de Migração entre Servidores Linux

## Visão Geral

```
Servidor Origem  →  rsync  →  Servidor Destino
(tem chave privada)           (tem chave pública)
```

---

## 1. Configurar Chaves SSH entre Servidores

### Como funciona:

| Servidor | O que precisa |
|----------|--------------|
| Origem (quem envia) | Chave **privada** (`~/.ssh/id_rsa`) |
| Destino (quem recebe) | Chave **pública** em `~/.ssh/authorized_keys` |

### Verificar se já tem chave no servidor origem:

```bash
ls ~/.ssh/
# Deve aparecer: id_rsa  id_rsa.pub
```

### Se não tiver, gerar uma nova chave:

```bash
ssh-keygen -t ed25519
# Aperta Enter em tudo (sem senha)
```

### Ver a chave pública para copiar:

```bash
cat ~/.ssh/id_rsa.pub
# ou
cat ~/.ssh/id_ed25519.pub
```

### Adicionar a chave pública no servidor destino:

**Opção 1 — Automático (se tiver acesso com senha):**
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub usuario@IP_DESTINO

# Com porta diferente:
ssh-copy-id -i ~/.ssh/id_rsa.pub -p 2222 usuario@IP_DESTINO
```

**Opção 2 — Manual (já logado no destino):**
```bash
echo "COLE_AQUI_A_CHAVE_PUBLICA" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Testar a conexão:

```bash
ssh usuario@IP_DESTINO
# Deve entrar sem pedir senha
```

---

## 2. Transferir Arquivos com rsync

### Comando básico:

```bash
rsync -avz --progress /caminho/origem/ usuario@IP_DESTINO:/caminho/destino/
```

### Com chave SSH específica:

```bash
rsync -avz --progress -e "ssh -i ~/.ssh/id_rsa" /caminho/origem/ usuario@IP_DESTINO:/caminho/destino/
```

### Com porta SSH diferente:

```bash
rsync -avz --progress -e "ssh -i ~/.ssh/id_rsa -p 2222" /caminho/origem/ usuario@IP_DESTINO:/caminho/destino/
```

### Atenção na barra `/` no final:

```bash
# COM barra — manda o CONTEÚDO da pasta
rsync ... /storage/www/app/  usuario@IP:/var/www/app/

# SEM barra — manda a PASTA inteira (cria a pasta dentro do destino)
rsync ... /storage/www/app   usuario@IP:/var/www/
```

### Flags explicadas:

| Flag | Descrição |
|------|-----------|
| `-a` | Preserva permissões, datas e links simbólicos |
| `-v` | Verbose (mostra o que está sendo transferido) |
| `-z` | Comprime durante a transferência |
| `--progress` | Mostra progresso em tempo real |

---

## 3. Rodar rsync sem Risco de Queda (Recomendado para arquivos grandes)

Use `screen` para manter o processo rodando mesmo se a conexão cair:

```bash
# Iniciar sessão screen
screen -S migracao

# Rodar o rsync normalmente
rsync -avz --progress /storage/www/app/ usuario@IP_DESTINO:/var/www/app/

# Se a conexão cair, reconectar com:
screen -r migracao
```

### Se a conexão cair sem screen:

Sem problema! O rsync retoma de onde parou. Só rode o mesmo comando novamente:

```bash
rsync -avz --progress /storage/www/app/ usuario@IP_DESTINO:/var/www/app/
# Ele não reenvia o que já foi transferido
```

---

## 4. Fluxo de Migração Completo (com Docker)

### Estratégia recomendada:

```
Homolog (código atualizado) → Novo servidor → Prod (só files/banco)
```

### Passo a passo:

**1. Criar pasta no destino:**
```bash
sudo mkdir -p /var/www/bussola
```

**2. Enviar código do homolog:**
```bash
rsync -avz --progress -e "ssh -i ~/.ssh/id_rsa" \
  /storage/www/bussola/ \
  ubuntu@IP_DESTINO:/var/www/bussola/
```

**3. Instalar Docker no destino:**
```bash
curl -fsSL https://get.docker.com | sh
sudo apt install docker-compose-plugin -y
```

**4. Subir a aplicação:**
```bash
cd /var/www/bussola
docker compose up -d
```

**5. Copiar arquivos/uploads do prod:**
```bash
rsync -avz --progress -e "ssh -i ~/.ssh/id_rsa" \
  /storage/www/bussola/storage/app/public/ \
  ubuntu@IP_DESTINO:/var/www/bussola/storage/app/public/
```

**6. Exportar banco de dados do prod:**
```bash
# MySQL
mysqldump -u root -p banco_prod > backup.sql

# Enviar pro destino
rsync -avz --progress backup.sql ubuntu@IP_DESTINO:/var/www/bussola/

# Importar no destino
mysql -u root -p banco_novo < backup.sql
```

---

## 5. AWS — Particularidades

### Conectar com chave .pem:

```bash
ssh -i ~/.ssh/minha-chave.pem ubuntu@IP_AWS
```

### rsync com chave .pem:

```bash
rsync -avz --progress -e "ssh -i ~/.ssh/minha-chave.pem" \
  /storage/www/app/ \
  ubuntu@IP_AWS:/var/www/app/
```

### Usuários padrão por SO:

| SO | Usuário |
|----|---------|
| Ubuntu | `ubuntu` |
| Amazon Linux | `ec2-user` |
| CentOS | `centos` |
| Debian | `admin` |

### Liberar portas no Security Group (console AWS):

```
EC2 → Instances → sua instância
→ Security → Security Groups
→ Edit Inbound Rules → Add Rule

Tipo: Custom TCP
Porta: 80, 443, 22 (ou porta customizada)
Source: 0.0.0.0/0
```

---

## 6. Liberar Porta SSH no Servidor (se porta 22 bloqueada)

### Editar config do SSH:

```bash
sudo nano /etc/ssh/sshd_config
```

Adicionar nova porta:
```
Port 22
Port 2222
```

### Liberar no firewall CSF (WHM):

```bash
sudo nano /etc/csf/csf.conf
# Adicionar 2222 na linha TCP_IN
sudo csf -r
```

### Liberar no firewalld:

```bash
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload
```

### Reiniciar SSH:

```bash
sudo systemctl restart sshd
```

### Configurar no VS Code (SSH config):

```
Host meu-servidor
    HostName SEU_IP
    User ubuntu
    Port 2222
    IdentityFile ~/.ssh/id_rsa
```

---

## Dicas Gerais

- **Sempre teste** a conexão SSH antes de rodar o rsync
- **Use screen** para transferências grandes (acima de 5GB)
- **rsync é seguro** — se cair, retoma de onde parou
- **authorized_keys** pode ter várias chaves, uma por linha
- **Nunca delete** a chave atual do authorized_keys antes de testar a nova
