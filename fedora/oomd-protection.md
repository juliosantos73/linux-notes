# Configuração de Proteção contra Travamentos por Alta Carga de I/O e RAM (Fedora 44)

Este guia documenta a resolução de um travamento severo do sistema (*thrashing*) gerado por conflito de escrita em disco e alocação de memória entre o **Docker Engine** e o **JetBrains IntelliJ IDEA**, que resultou no congelamento do GNOME Shell e falha do serviço `sssd-kcm`.

A solução consistiu em reativar e tunar o **Systemd-OOMD** (Out-Of-Memory Daemon) utilizando as boas práticas de sobreposição (*drop-ins*) do Fedora.

---

## 🛠️ Diagnóstico do Problema

O travamento foi identificado analisando os logs do boot anterior através do comando:
```bash
journalctl -b -1 | grep -iE 'oom|kill|alloc' | tail -n 20
```

### Sintomas observados:
1. LED de atividade do disco piscando freneticamente (Gargalo de I/O).
2. Interface gráfica congelada, respondendo muito lentamente ao CapsLock.
3. Mensagens `Can't update stage views actor... because it needs an allocation` no GNOME Shell.
4. Serviço `systemd-oomd` inativo/desativado de fábrica, impedindo o Linux de matar os processos vilões automaticamente.

---

## 🚀 Passos para Configuração e Correção

### 1. Reativar o Serviço Systemd-OOMD
O serviço estava desligado no sistema. Forçou-se a inicialização automática no boot:

```bash
sudo systemctl unmask systemd-oomd
sudo systemctl enable --now systemd-oomd
```

### 2. Criar Estrutura de Configuração Personalizada (*Drop-in*)
Seguindo as diretrizes do Fedora (mantendo os arquivos de `/usr/lib/` intactos), criou-se um diretório e um arquivo de configuração local em `/etc/`:

```bash
# Criar o diretório de drop-in para o oomd
sudo mkdir -p /etc/systemd/oomd.conf.d

# Criar o arquivo de regras personalizadas
sudo nano /etc/systemd/oomd.conf.d/99-custom.conf
```

### 3. Aplicar os Limites de Proteção Otimizados (32GB RAM)
Dentro do arquivo `99-custom.conf`, injetou-se a configuração abaixo. Os limites foram reduzidos (de 90% para 80% na Swap, e de 60% para 40% na pressão de memória) para tornar o sistema mais agressivo contra travamentos de IDEs e Contêineres:

```ini
[OOM]
SwapUsedLimit=80%
DefaultMemoryPressureLimit=40%
```

### 4. Aplicar e Validar as Alterações

Reinicie o serviço para que o Kernel adote os novos limites imediatamente:
```bash
sudo systemctl restart systemd-oomd
```

#### Validar se o Systemd concatenou as configurações corretamente:
```bash
systemd-analyze cat-config systemd/oomd.conf
```
*A saída deve mostrar o arquivo padrão de fábrica seguido pelo bloco customizado `/etc/systemd/oomd.conf.d/99-custom.conf` no final.*

#### Validar o status em tempo real do monitor de memória:
```bash
oomctl
```
*Certifique-se de que os valores `Swap Used Limit` e `Default Memory Pressure Limit` refletem **80.00%** e **40.00%**.*

---

## 🧹 Manutenção Preventiva do Docker (Opcional)
Se houver suspeita de contêineres fantasmas ou travados consumindo recursos em segundo plano após um reboot forçado, limpe o cache do Docker com segurança (não remove contêineres ativos ou volumes de dados):

```bash
docker system prune -f
```
