# Linux Hardening — Ubuntu Server

Checklist de hardening que aplico nos servidores Ubuntu onde rodam meus sistemas (Node.js + Nginx + PM2).
Cada item tem: **o que é**, **por que importa**, **como aplicar** e **como verificar**.

Estou montando esse checklist para ajudar outras pessoas que mantêm servidores online e sofrem com ataques, ou que querem proteger melhor seus servidores. Como já passei por dificuldades, acredito que vai ajudar bastante gente.

---

## ⚠️ Antes de começar

- Faça um **snapshot/backup** da VPS no painel do provedor.
- Deixe **uma segunda sessão SSH aberta** enquanto mexe no SSH e no firewall. Se algo der errado, você não fica trancado para fora.
- Aplique um item por vez e teste.

---

## 1. Atualizações automáticas de segurança

**Por que:** a maioria das invasões explora falhas que já têm correção publicada.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

**Verificar:**
```bash
systemctl status unattended-upgrades --no-pager
systemctl list-timers apt-daily* --no-pager
cat /etc/apt/apt.conf.d/20auto-upgrades
sudo tail -n 20 /var/log/unattended-upgrades/unattended-upgrades.log
ls /var/run/reboot-required 2>/dev/null && cat /var/run/reboot-required.pkgs
```

**Resultado no meu servidor (02/11/2026):**

- Ubuntu 24.04 (Noble). O `unattended-upgrades` já vinha ativo por padrão: `20auto-upgrades` com as duas opções em `"1"`.
- Os timers `apt-daily` e `apt-daily-upgrade` rodam todo dia. O log mostrava "No packages found", ou seja, o servidor estava em dia.
- Ao rodar o `apt upgrade` manual, o `apparmor` foi atualizado e o sistema passou a pedir reinicialização (`/var/run/reboot-required`).
- Antes de reiniciar, confirmei que a aplicação volta sozinha: serviço `pm2-root` habilitado e `pm2 save` executado.

**Aprendizado:** atualização instalada não é atualização aplicada. Componentes como o apparmor e o kernel só passam a valer depois de reiniciar. Por isso vale checar o `/var/run/reboot-required` com frequência.

---

## 2. Usuário sem root para o dia a dia

**Por que:** usar root direto dá poder total a qualquer erro ou invasão.

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
```

✍️ Resultado no meu servidor:

---

## 3. SSH: só chave, sem senha e sem root

**Por que:** robôs tentam senha no SSH o tempo todo.

✍️ Antes de aplicar, rode e anote quantas tentativas de login falharam:
```bash
sudo grep "Failed password" /var/log/auth.log | wc -l
```

1. Confirme que **consegue entrar por chave SSH** com o usuário novo.
2. Edite `/etc/ssh/sshd_config`:
```
PermitRootLogin no
PasswordAuthentication no
```
3. Teste a configuração e reinicie:
```bash
sudo sshd -t && sudo systemctl restart ssh
```

**Verificar** (em outra janela, sem fechar a sessão atual):
```bash
ssh root@SEU_SERVIDOR   # deve ser recusado
```

✍️ Resultado no meu servidor:

---

## 4. Firewall (UFW)

**Por que:** só as portas que precisam ficar abertas devem ficar abertas.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

**Verificar:**
```bash
sudo ufw status verbose
sudo ss -tulpn          # o que está escutando em cada porta
```

✍️ Quais portas estavam abertas antes e quais ficaram:

---

## 5. Fail2ban

**Por que:** bloqueia automaticamente IPs que erram login várias vezes.

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
```

**Verificar:**
```bash
sudo fail2ban-client status sshd
```

✍️ Quantos IPs foram bloqueados nas primeiras 24h:

---

## 6. Nginx: esconder versão

**Por que:** mostrar a versão ajuda o atacante a procurar falhas conhecidas.

Em `/etc/nginx/nginx.conf`, dentro de `http { }`:
```
server_tokens off;
```
```bash
sudo nginx -t && sudo systemctl reload nginx
```

**Verificar:**
```bash
curl -I https://SEU_DOMINIO   # o cabeçalho Server não deve mostrar a versão
```

✍️ Resultado no meu servidor:

---

## 7. Aplicação Node.js sem root

**Por que:** se a aplicação for invadida, o atacante fica limitado ao usuário dela.

**Verificar:**
```bash
ps -o user= -p $(pgrep -f "node" | head -1)   # não deve ser root
```

✍️ Resultado no meu servidor:

---

## Próximos passos

- [ ] Auditoria com [Lynis](https://cisofy.com/lynis/) (`sudo lynis audit system`) e comparação de score antes/depois
- [ ] Script `check.sh` que verifica todos os itens automaticamente
- [ ] Envio de logs para um SIEM (Wazuh) — ver repositório `blue-team-lab`

---

## Autor

**Francislan Gonçalves** — [LinkedIn](https://www.linkedin.com/in/francislan-gon%C3%A7alves-6a1791129/) · [FN Solutions](https://fnsolutions.cloud)
