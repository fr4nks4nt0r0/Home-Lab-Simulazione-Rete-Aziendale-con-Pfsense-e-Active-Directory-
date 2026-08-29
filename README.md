# Home-Lab-Simulazione-Rete-Aziendale-con-Pfsense-e-Active-Directory-
Ambiente laboratorio virtualizzato (VMware) che replica infrastruttura di una piccola azienda, un firewall/Router perimetrale (pfsense), un Domain Controller Windows Server con AD e DNS, un client Windows unito al dominio.

OBIETTIVO PROGETTO:
Dopo aver ottenuto la cert CompTIA Security+, volevo consolidare le basi pratiche di sistemistica e networking che stanno dietro la teoria, in particolare capendo come un'azienda strutturi la propria rete interna: un firewall che fa da confine e gateway, un dominio Active Directory per la gestione di utenti e macchine, e un client che si autentica su quel dominio.

ARCHITETTURA:
<img width="960" height="540" alt="strutturahomelab" src="https://github.com/user-attachments/assets/9ed1adbf-f542-4af6-ad31-f0fc01100f6a" />

Stack Tecnologico:

Hypervisor VMware///Workstation Player (reti host-only)
Firewall/router///pfSense 2.90 CE
Domain Controller//Windows Server 2022 Evaluation
Servizi///Active Directory Domain Service, DNS integrato
Client///Windows 11 Enterprise Evaluation
Rete///10.10.10.0/24(LAN), NAT per WAN

Setup --- Panoramica
1) Rete virtuale: Creata una rete host only dedicata (VMnet1) su VMware, isolata dalla rete fisica del laptop, per far da "cavo" fra vm.
2) pfSense: installato da ISO, assegnate le interfacce WAN (NAT, verso internet) e LAN (verso le altre VM), configurato come DHCP server sulla LAN.
3) DC: installato Windows server 2022, promosso a DC creando una nuova foresta, lab.local, con il ruolo DNS installato in automatico e integrato con AD.
4) Utenti di dominio: creato il primo utente tramite Active Directory Users and Computers
5) Client: Installato windows 11, configurato DNS per puntare al Domain Controller, e unito al dominio lab.local
6) Verifica: login riuscito con l'utente di dominio dal client, e conferma da Powershell (Get-ADComputer) che la macchina risultasse correttamente registrata in Active Directory.




PROBLEMI INCONTRATI E RISOLUZIONE<----
Questa è la parte che considero più importante del progetto, la configurazione "pulita" segue una guida, ma la maggior parte dell'apprendimento effettivo è arrivata dai problemi.

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





VERIFICA FINALE----------
- Get-ADDomain sul DC conferma il dominio di lab.local attivo
- nslookup lab.local risolve correttamente all'IP del DC.
- Login riuscito sul client con un account di dominio LAB\mrrossi
- Get-AdCOmputer -Filter * mostra sia DC sia CLIENT correttamente registrati.

Screenshot disponibili nella cartella /screenshots.

COMPETENZE DIMOSTRATE
- Progettazione e configurazione di una rete isolata, con NAT, DHCP e routing base.
- Installazione di un firewall perimetrale (pfsense)
- Deployment di AD Domain Services e integrazione con DNS
- Gestione utenti e macchinine in un dominio Windows
- Troubleshooting sistematico di problemi di rete attraverso approccio metodico, individuare variabile e verificare un livello alla volta.
