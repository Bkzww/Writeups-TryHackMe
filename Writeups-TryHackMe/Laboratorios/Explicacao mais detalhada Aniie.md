**Passo 1: Levar o Ator para o seu camarim ( `cp ... /tmp/python3` )**

Você pegou o Ator (Python) e o levou para a pasta `/tmp` (um camarim público). Agora ele está sob o seu controle.

**Passo 2: O Roubo do Carimbo ( `/sbin/setcap ...` )**

Você descobriu que o Segurança da Bilheteria ( `setcap` ) estava dormindo (vulnerável com SUID). Você pegou o carimbo dele e foi até o seu camarim.

Lá no `/tmp`, você usou esse carimbo do `setcap` e **carimbou o crachá do seu Ator (Python)** com a marca **`cap_setuid`** (que significa _"Este ator tem permissão para fingir que é o Diretor"_).

> **Atenção:** Você não colocou o código do segurança dentro do ator. Você apenas usou o carimbo do segurança para marcar o crachá que o ator está vestindo no peito. O arquivo do Python continua idêntico, ele só ganhou um "atributo" novo no sistema de arquivos do Linux.

**Passo 3: A Atuação ( `/tmp/python3 -c 'os.setuid(0)...'` )**

Agora, você manda o seu Ator (Python) entrar no palco.

O Python executa a linha `os.setuid(0)` (que é ele interpretando o papel e dizendo: _"Eu sou o Diretor/Root!"_).

Os seguranças do teatro (o kernel do Linux) correm para pará-lo, mas olham para o peito do Python e vêem o carimbo **`cap_setuid`** que você colocou lá no Passo 2.

O kernel diz: _"Ah, desculpe, o crachá dele está carimbado pelo `setcap`. Ele realmente tem permissão para agir como Diretor"_. O Linux aceita, o Python vira root de verdade e abre o terminal ( `/bin/bash` ) para você.