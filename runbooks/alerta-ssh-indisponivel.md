# Runbook — Alerta: Serviço SSH indisponível

| | |
|---|---|
| **Severidade** | High |
| **Item** | `net.tcp.service[ssh]` |
| **Gatilho** | `last(/Zabbix server/net.tcp.service[ssh])=0` |

## O que o alerta significa

O Zabbix tentou abrir uma conexão TCP no serviço SSH do host e não
obteve resposta. Indica serviço parado, porta tomada por outro processo,
ou bloqueio de rede. **Não** significa necessariamente que o host está
fora do ar.

## Como confirmar

```bash
systemctl status ssh              # está active? está enabled?
ss -tulpn | grep :22              # a porta está em escuta? quem é o dono?
journalctl -u ssh -n 50 --no-pager
```

As três respostas juntas já apontam a causa na maioria dos casos:

- **Não está em escuta** → serviço parado ou não habilitado
- **Está em escuta, mas o dono não é o `sshd`** → socket activation
- **Está em escuta com o dono certo e o alerta persiste** → rede

## Causas prováveis

| Causa | Como identificar | Correção |
|---|---|---|
| Serviço parado | `status` mostra `inactive (dead)` | `sudo systemctl start ssh` |
| Não habilitado no boot | `status` mostra `disabled` | `sudo systemctl enable ssh` |
| Socket activation segurando a porta | `ss` mostra dono `systemd` com `pid=1`; `status` mostra `TriggeredBy: ● ssh.socket` | `sudo systemctl disable --now ssh.socket` e depois `sudo systemctl enable --now ssh` |
| Falha ao iniciar | `status` mostra `failed` | Ler o motivo no `journalctl -u ssh` antes de tentar subir de novo |
| Bloqueio de rede ou firewall | Porta em escuta, serviço `active`, alerta persiste | Verificar regra de firewall e alcance de rede |
| Agente do Zabbix fora do ar | Outros itens do mesmo host também sem dado | Verificar `zabbix-agent` — o problema é na coleta, não no SSH |

## Atenção

`systemctl stop ssh` **não** libera a porta 22 quando o `ssh.socket`
está ativo: o systemd continua segurando a porta como PID 1 e sobe o
serviço sob demanda. Confirmar sempre com `ss -tulpn` em vez de confiar
só no `systemctl status`.

## Quando escalar

Se a porta está em escuta, o serviço está `active` e o alerta persiste,
o problema é de rede ou de filtro, e não do host — escalar para a equipe
responsável por rede, com o resultado dos três comandos de confirmação
anexado.

## Após resolver

1. Confirmar no Zabbix que o evento mudou para **RESOLVED**
2. Registrar o tempo de resolução
3. Se a causa foi socket activation ou serviço desabilitado no boot,
   corrigir a configuração — senão o alerta volta no próximo reinício
