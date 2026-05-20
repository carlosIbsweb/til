# Expandir Volume EBS na AWS (Linux)

Guia para expandir partição e filesystem após aumentar um volume EBS em instâncias Linux na Amazon Web Services.

---

## Pré-requisito

O volume EBS já deve ter sido expandido pelo console ou CLI da AWS **antes** de rodar os comandos abaixo. Os comandos aqui apenas fazem o sistema operacional reconhecer o novo tamanho.

---

## 1. Identificar o disco

```bash
lsblk
```

### Saída esperada — tipo `xvda` (instâncias mais antigas):

```
xvda
└─xvda1
```

### Saída esperada — tipo `nvme` (instâncias modernas):

```
nvme0n1
├─nvme0n1p1   ← partição root
├─nvme0n1p14
├─nvme0n1p15
└─nvme0n1p16
```

> **Atenção:** instâncias mais novas (Nitro) usam `nvme0n1` em vez de `xvda`. Confirme antes de prosseguir.

---

## 2. Instalar o `growpart`

```bash
sudo apt update
sudo apt install -y cloud-guest-utils
```

> Se o disco estiver **sem espaço** e a instalação falhar, veja a seção [Disco sem espaço](#disco-sem-espaço) antes de continuar.

---

## 3. Expandir a partição

### Para discos `xvda`:

```bash
sudo growpart /dev/xvda 1
```

### Para discos `nvme`:

```bash
sudo growpart /dev/nvme0n1 1
```

> O número `1` refere-se ao índice da partição root (a primeira partição do disco).

---

## 4. Expandir o filesystem

### Para discos `xvda`:

```bash
sudo resize2fs /dev/xvda1
```

### Para discos `nvme`:

```bash
sudo resize2fs /dev/nvme0n1p1
```

---

## 5. Confirmar o novo espaço

```bash
df -h
```

O mount `/` deve mostrar o novo tamanho total do volume.

---

## Disco sem espaço

Se ao rodar `growpart` aparecer o erro abaixo:

```
mkdir: cannot create directory '/tmp/growpart.XXXXX': No space left on device
FAILED: failed to make temp dir
```

Libere espaço antes de continuar:

```bash
sudo rm -rf /tmp/*
sudo apt clean
sudo rm -rf /var/cache/apt/*
sudo journalctl --vacuum-time=1d
```

Verifique o espaço liberado:

```bash
df -h
```

Depois de ter alguns MB livres, repita os passos 2, 3 e 4 normalmente.

---

## Resumo rápido (nvme)

```bash
# 1. Ver o disco
lsblk

# 2. Instalar growpart
sudo apt update && sudo apt install -y cloud-guest-utils

# 3. Expandir partição
sudo growpart /dev/nvme0n1 1

# 4. Expandir filesystem
sudo resize2fs /dev/nvme0n1p1

# 5. Confirmar
df -h
```

---

## Referências

- [AWS Docs — Extending a Linux file system after resizing a volume](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html)
- Pacote `cloud-guest-utils`: fornece o comando `growpart`
- Comando `resize2fs`: faz parte do pacote `e2fsprogs` (pré-instalado no Ubuntu)
