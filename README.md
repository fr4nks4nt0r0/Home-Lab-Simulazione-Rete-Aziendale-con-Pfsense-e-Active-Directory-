# Home-Lab-Simulazione-Rete-Aziendale-con-Pfsense-e-Active-Directory-
Ambiente laboratorio virtualizzato (VMware) che replica infrastruttura di una piccola azienda, un firewall/Router perimetrale (pfsense), un Domain Controller Windows Server con AD e DNS, un client Windows unito al dominio.

OBIETTIVO PROGETTO:
Dopo aver ottenuto la cert CompTIA Security+, volevo consolidare le basi pratiche di sistemistica e networking che stanno dietro la teoria, in particolare capendo come un'azienda strutturi la propria rete interna: un firewall che fa da confine e gateway, un dominio Active Directory per la gestione di utenti e macchine, e un client che si autentica su quel dominio.

ARCHITETTURA:
<img width="960" height="540" alt="strutturahomelab" src="https://github.com/user-attachments/assets/9ed1adbf-f542-4af6-ad31-f0fc01100f6a" />

Struttura Active Directory:
lab.local
- Domain controllers (default, non modificato)
- Server
- Client => workstation utente (ho applicato GPO di hardening qui)
- Utenti => account persona es. Mario Rossi


Stack Tecnologico:

Hypervisor VMware///Workstation Player (reti host-only)
Firewall-router///pfSense 2.90 CE
Domain Controller//Windows Server 2022 Evaluation
Servizi///Active Directory Domain Service, DNS integrato
Client///Windows 11 Enterprise Evaluation
Rete///10.10.0.0/24(client), 10.10.20.0/24 (Server) NAT per WAN

Setup --- Panoramica
1) Rete virtuale: Create 2 reti host only dedicate (VMnet1 per il segmento client, VMnet2 per server) su VMware, isolate dalla rete fisica del laptop.
2) pfSense: installato da ISO, assegnate le interfacce WAN (NAT, verso internet) e LAN (segmento Client, 10.10.10.0/24), e OPT1(segmento server, 10.10.20.0/24) ciascuna con DHCP dedicato.
3) DC: installato Windows server 2022, promosso a DC creando una nuova foresta, lab.local, con il ruolo DNS installato in automatico e integrato con AD.
4) STRUTTURA AD: create le Organizational Unit Server, Client, e Utenti per separare logicamente risorse, workstation e account, in modo da poter applicare Group Policy mirate a ciascun gruppo.
5) Client: Installato windows 11 sul segmento client, configurato DNS per puntare al Domain Controller, e unito al dominio lab.local col relativo oggetto computer.
6) Segmentazione firewall: configurate le regole su pfsense per controllare esplicitamente il traffico tra i due segmenti, invece di lasciare tutto aperto.
7) Hardening: applicate group policy sulla OU client per un banner di sicurezza al login e la disattivazione di protocolli legacy vulnerabili.


SEGMENTAZIONE DI RETE COME VLAN SIMULATA
Non disponendo di switch fisico ho cercato di implementare il principio con 2 reti host only separate vmnet1 e vmnet2, ciascuna collegata ad una diversa interfaccia di pfsense.
Ogni interfaccia di pfSense blocca di default tutto il traffico in entrata tranne quello permesso da una regola esplicita, la LAN (client) ha una regola "Allow LAN to Any" quindi il client può sempre raggiungere il server.
L'interfaccia OPT1 (server) nasce invece senza regola col traffico completamente bloccato, finchè non si aggiunge una regola che permette al server di raggiungere il segmento Client (Allow opt1 to LAN subnets). 

Verifica pratica:
- Con la regola attiva, ping dal DC (10.10.20.10) al client (10.10.10.20) => risposta regolare, 0% loss
- Con la regola disattiva, stesso ping =: 100% loss

HARDENING IMPLEMENTATO

Segmentazione tramite Active Directory(OU+GPO)
Creata una GPO ("GPO-Client-Hardening) collegata specificamente alla OU Client, così le impostazioni si applicano solo alle workstation non DC:
- Banner di sicurezza: messaggio e titolo di avviso configurati tramite "interactive logon: message tex/title for users attempting to log on", per simulare una pratica standard in ambito enterprise (avviso legale di monitoraggio prima dell'accesso)

Disattivazione protocolli legacy vulnerabili:
NTLMv1 disabilitato: policy Network security: LAN Manager authentication level messa su "Send NTLMv2 response only. Refuse LM & NTLM", per prevenire potenziali attacchi downgrade o relay basati su NTLMv1
SMBv1 disabilitato: rimosso via Group Policy Preferences (chiave di registro SMB1 a 0, sotto LanmanServer/Parameters) protocollo vulnerabile ad esempio a Wanna cry
LLMNR disabilitato: policy "turn off multicast name resolution" attivata, per chiudere un potenziale vettore di attacco man-in-the-middle su reti interne.

PROBLEMI INCONTRATI E RISOLUZIONE<----
Questa è la parte che considero più importante del progetto, visto che la configurazione "pulita" segue una guida, ma la maggior parte dell'apprendimento effettivo è arrivata dai problemi.

1) Subnet mismatch fra rete virtuale e pfsense
   Dopo l'installazione, la reete Host-Only di VMware(VMnet1) era su una subnet diversa, 192.168.183.0/24, rispetto a quella assegnata di default ala lan di pfsense, 192.168.1.0/24.
   Le due reti non potevano comunicare, risolto allineando manualmente la subnet dellarete virtuale a quella di pfsense.

2) CONFLITTO CON RETE WIFI DI CASA
Il primo tentativo di allineamento è finito su 192.168.1.0/24, stessa subnet del router di casa, questo ha causato ambiguità dal routing windows, risolto spostando intero laboratorio su 10.10.10.0/24.

3) Conflitto IP fra host e gateway
Dopo cambio di subnet, sia l'adattatore virtuale del pc host sia l'interfaccia LAN di pfsense hanno finito per avere lo stesso indirizzo, 10.10.10.1, un duplicato che impediva ogni comunicazione.
Risolto assegnando manualmente un IP diverso, 10.10.10.2 all'adattatore del pc host, lasciando 10.10.10.1 a pfsense come gateway.

4) interfaccia LAN condigurata come cliente dhcp invece che statica
in un passaggio della conf da console, l'interfaccia LAN di pfsense ha finito per richiedere un IP via DHCP invece di usarne uno statico, un errore concettuale dato che è pfsense a dover distribuire gli ip, non riceverli.
Risolto riconfigurando interfaccia con ip statico e attivando separatemente DHCP per i client.

5) Accesso alla dasboard, bloccato da un browser specifico
Brave impediva collegamento alla pagina di login di pfsense per via degli shield, risolto individuando si trattasse di un problema lato browser non di rete.

6) Spostando il Domain Controller su un nuovo segmento di rete, il client non riusciva più ad autenticare l'account di dominio necessario per aggiornare le proprie impostazioni DNS, (serve il dominio raggiungibile per elevare i permessi, ma serve il DNS aggiornato per raggiungere il dominio). Risolto cambiando l'ordine delle operazioni: aggiornare prima il DNS del client mentre il dominio era ancora pienamente raggiungibile, e solo dopo spostare il Domain Controller sul nuovo segmento.

7) Una regola pensata per permettere il traffico Server→Client usava come destinazione l'indirizzo del singolo router (LAN address) invece dell'intera subnet (LAN subnets), escludendo di fatto ogni host reale della rete client. 

8) Regola disattivata per errore durante iterazioni di test
Durante ripetuti test di verifica (attiva/disattiva la regola per dimostrare il blocco), la regola è rimasta disattivata da un passaggio precedente, nascondendo il motivo del successivo errore di ping. Risolto riguardando con attenzione lo stato (enable/disable)




VERIFICA FINALE----------
- Get-ADDomain sul DC conferma il dominio di lab.local attivo
- nslookup lab.local risolve correttamente all'IP del DC.
- Login riuscito sul client con un account di dominio LAB\mrrossi
- Get-AdCOmputer -Filter * mostra sia DC sia CLIENT correttamente registrati, ciascuno nella propria OU.
- Ping server=>client verificato in entrambi gli stati della regola firewall (attiva = passa, disattivata = blocca)
- Banner di sicurezza visibile solo sul client non sul dc, a conferma della corretta applicazione alla OU client.
- SMBv1 Verificato disabilitato via reg query sul client.

Screenshot disponibili nella cartella /screenshots.

COMPETENZE DIMOSTRATE
- Progettazione e configurazione di una rete segmentata con NAT, DHCP e routing tra subnet.
- Installazione di un firewall perimetrale (pfsense) con regole di controllo del traffico fra segmenti, comprendendo la direzionalità delle regole in un firewall stateful
- Deployment di AD Domain Services e integrazione con DNS
- Progettazione della struttura organizzativa in AD con OU per applicazione mirata di group policy
- Hardening di base contro protocolli legacy obsoleti
- Troubleshooting sistematico di problemi di rete attraverso approccio metodico, individuare variabile e verificare un livello alla volta.

SVILUPPI FUTURI:
Raccolta centralizzata dei log (windows event forwarding o SIEM open source
Simulazione di attacchi noti
