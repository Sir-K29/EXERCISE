# EXERCISE
# Tutorial: Hardening Windows 11 Pro Standalone

> **Note generali:**
> * Si lavora con l'account `AdminIT` dal **punto 3** in poi.
> * Lo screenshot **`[F]`** va effettuato dopo ogni passaggio contrassegnato.
> * **Tempi indicativi totali:** circa 2 ore.

---

## 0. Preparazione (5 min)

1. Accendi la VM da **Hyper-V Manager** e accedi con l’utente iniziale.
2. Premi `Win + R` > digita `winver` > **OK**. Annota edizione e build **`[F]`**.
3. Se ci sono aggiornamenti automatici attivi, mettili in pausa: **Impostazioni > Windows Update > Sospendi aggiornamenti**.
4. In **Hyper-V Manager**: tasto destro sulla VM > **Checkpoint**, assegna il nome `00-iniziale`.
   * *Attenzione:* Se la VM ha un file ISO montato nel DVD, smontalo ora (**Impostazioni VM > Controller SCSI > Unità DVD > Nessuno**), altrimenti BitLocker darà errore più tardi.

---

## 1. Connettività (10 min)

1. Apri il Prompt dei comandi (`cmd`) ed esegui i seguenti comandi, facendo uno screenshot **`[F]`** per ciascuno:
   ```cmd
   ipconfig /all
   ping 8.8.8.8
   nslookup google.com
   ```
2. **Se il ping all’IP fallisce:** vai su **Hyper-V Manager > Impostazioni VM > Scheda di rete** e verifica che sia collegata a uno switch virtuale (*Default Switch* o esterno). Poi nella VM esegui `ncpa.cpl` > **Proprietà scheda > IPv4**: imposta DHCP automatico, oppure IP, gateway e DNS statici.
3. **Se il ping all’IP funziona ma `nslookup` no:** problema DNS. Imposta come DNS `8.8.8.8` nelle proprietà IPv4.
4. Vai su **Impostazioni > Rete e Internet > Proxy**: assicurati che sia tutto disattivato.
5. Apri Microsoft Edge e naviga su un sito web **`[F]`**.

---

## 2. Baseline (15 min – solo osservazione)

1. `lusrmgr.msc` > **Gruppi > Administrators**: verifica chi è membro **`[F]`**.
2. `lusrmgr.msc` > **Utenti**: verifica che `Guest` e `Administrator` abbiano la freccia verso il basso (indica che sono disattivati) **`[F]`**.
3. **Sicurezza di Windows** (dal menu Start): controlla *Protezione da virus*, *Firewall*, *Controllo app e browser* **`[F]`**.
4. `optionalfeatures.exe`: cerca *"Supporto per la condivisione di file SMB 1.0/CIFS"* e verifica che **non** sia spuntato (non confonderlo con *"SMB Direct"*) **`[F]`**.
5. **Impostazioni > Sistema > Desktop remoto**: verifica lo stato **`[F]`**.
6. **Pannello di controllo > Crittografia unità BitLocker**: verifica lo stato **`[F]`**.
7. `UserAccountControlSettings.exe`: annota il livello attuale e chiudi senza modificare **`[F]`**.
8. `secpol.msc` > **Criteri account > Criteri password** e **Criteri di blocco**: annota i valori **`[F]`**.
9. Crea il checkpoint `01-baseline` in Hyper-V Manager.

> **Importante:** Da questa fase ricavi la tabella delle criticità. Scrivi solo ciò che hai davvero trovato.

---

## 3. Account e privilegi (10 min)

> **Ordine obbligatorio**, per non restare senza amministratore.

1. `lusrmgr.msc` > **Utenti** > tasto destro nello spazio vuoto > **Nuovo utente**.
2. Nome: `AdminIT`, imposta una password, togli la spunta da *"L’utente deve cambiare la password al prossimo accesso"* > **Crea** **`[F]`**.
3. **Gruppi > Administrators** > doppio clic > **Aggiungi** > scrivi `AdminIT` > **Controlla nomi** > **OK** **`[F]`**.
4. **Start** > icona utente > **Disconnetti**, poi accedi con `AdminIT`.
   * *Suggerimento:* Se nella schermata di login compare solo l’ultimo utente, usa *"Altro utente"* e scrivi `.\AdminIT`.
5. Con `AdminIT`: `lusrmgr.msc` > **Gruppi > Administrators** > seleziona l’utente operativo iniziale > **Rimuovi** **`[F]`**.
6. **Gruppi > Users**: verifica che l’utente operativo sia presente **`[F]`**.
7. `Guest` e `Administrator`: se erano attivi, fai doppio clic su ciascuno e spunta *"Account disattivato"*.

---

## 4. UAC (2 min)

1. Esegui `UserAccountControlSettings.exe` > sposta il cursore al massimo su **Notifica sempre** > **OK** **`[F]`**.

---

## 5. Password policy e blocco (10 min)

1. Apri `secpol.msc` > **Criteri account > Criteri password** e imposta:
   * **Lunghezza minima:** `12`
   * **Complessità:** `Abilitato`
   * **Validità massima:** `90 giorni`
   * **Cronologia:** `5` **`[F]`**
2. Vai su **Criteri di blocco account** e imposta:
   * **Soglia di blocco:** `5 tentativi`
   * **Durata del blocco:** `15 minuti`
   * **Reimposta contatore dopo:** `15 minuti` **`[F]`**
3. Vai su **Criteri locali > Opzioni di sicurezza** > imposta *"Accesso interattivo: limite di inattività del computer"* = `600 secondi` **`[F]`**.

---

## 6. Windows Installer (5 min)

1. Apri `gpedit.msc` > **Configurazione computer > Modelli amministrativi > Componenti di Windows > Windows Installer**.
2. *"Disattiva Windows Installer"* > **Attivato**, opzione **Sempre** > **OK**.
3. *"Installa sempre con privilegi elevati"* > **Disattivato** **`[F]`**.
4. Verifica dal Prompt dei comandi come amministratore:
   ```cmd
   reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer" /v DisableMSI
   ```
   *(Deve restituire `0x2`)*.

---

## 7. AppLocker: blocco degli exe (15 min)

> **Nota:** Questo è il cuore della traccia. Su Windows 11 Pro l’applicazione non è ufficialmente supportata, ma nella prova effettiva blocca correttamente gli eseguibili.

1. Apri il Prompt dei comandi come amministratore (il titolo della finestra deve riportare *"Administrator"*) ed esegui:
   ```cmd
   sc config appidsvc start= auto
   sc start appidsvc
   sc query appidsvc
   ```
   * *Attenzione:* Lo spazio dopo `start=` è obbligatorio. Deve risultare `STATE: RUNNING` **`[F]`**. Usa il prompt perché `services.msc` può negare l’accesso.
2. Apri `secpol.msc` > **Criteri di controllo delle applicazioni > AppLocker > Regole eseguibili**.
3. Tasto destro nello spazio vuoto > **Crea regole predefinite** **`[F]`**. Compaiono tre regole: tutti in `C:\Windows`, tutti in `C:\Program Files`, amministratori ovunque.
4. Tasto destro su **AppLocker > Proprietà > scheda Imposizione**: spunta *"Configurato"* per le regole eseguibili e scegli **Applica regole** > **OK** **`[F]`**.
5. Esegui `gpupdate /force`, poi disconnettiti e riaccedi con l’utente standard (se serve, usa `Win + R` > `tsdiscon` per tornare alla schermata di login).

> **Nota per l’esame:** Se vuoi coprire anche installer e script, ripeti il punto 3 su *"Regole Windows Installer"* e *"Regole script"*, selezionando **Applica regole**.

---

## 8. winget e Store (5 min)

1. Apri `gpedit.msc` > **Componenti di Windows > Programma di installazione app**. Doppio clic su *"Abilita programma di installazione app"* > **Disattivato** **`[F]`**.
   * *(Se la cartella non è presente su questa build, passa oltre: il blocco effettivo lo fa AppLocker).*
2. Vai su **Componenti di Windows > Microsoft Store** > *"Disattiva l’applicazione Store"* > **Attivato** **`[F]`**.
   * *(Su edizione Pro l’effetto è solo parziale, quindi menzionalo nei limiti del report).*

---

## 9. Superficie d’attacco, Defender, firewall (15 min)

1. Esegui `sysdm.cpl` > scheda **Connessione remota**: nessuna spunta su *Assistenza remota*. Vai su **Impostazioni > Sistema > Desktop remoto**: **Off** **`[F]`**.
2. Apri `gpedit.msc` > **Componenti di Windows > Criteri AutoPlay** > *"Disattiva AutoPlay"* > **Attivato**, seleziona **Tutte le unità** **`[F]`**.
3. **Sicurezza di Windows > Protezione da virus e minacce > Gestisci impostazioni**: *Protezione in tempo reale* **On**, *Protezione da manomissione* **On** **`[F]`**.
4. **Controllo app e browser > Protezione basata sulla reputazione**: *SmartScreen* e *blocco delle app potenzialmente indesiderate* **On** **`[F]`**.
5. Apri `wf.msc` > **Proprietà Windows Defender Firewall**. Per ciascuna delle schede **Dominio**, **Privato**, **Pubblico**:
   * **Stato firewall:** `Attivato`
   * **Connessioni in entrata:** `Blocca (predefinito)`
   * **Sezione Registrazione > Personalizza > Registra pacchetti scartati:** `Sì` **`[F]`**

---

## 10. Audit (5 min)

1. Apri `secpol.msc` > **Criteri locali > Criteri di controllo**: abilita **Riuscito** e **Non riuscito** per:
   * *"Eventi di accesso account"*
   * *"Gestione account"*
   * *"Accesso agli oggetti"* **`[F]`**
2. Apri `eventvwr.msc` > **Registri di Windows > Sicurezza** > tasto destro > **Proprietà** > imposta dimensione massima a `131072 KB` **`[F]`**.

---

## 11. BitLocker (10 min – per ultimo)

1. Verifica che nel DVD virtuale non ci sia un file ISO montato e che in **Impostazioni VM > Sicurezza** sia abilitato il **TPM**.
2. Apri **Pannello di controllo > Crittografia unità BitLocker** > **Attiva BitLocker** su `C:` **`[F]`**.
3. Salva la chiave di ripristino **fuori dalla VM** e **non incollarla nel report**.
4. Riavvia se richiesto, poi crea il checkpoint `02-hardening`.
   * *Nota:* Creare lo snapshot dopo BitLocker evita problemi di ripristino con il TPM virtuale.

---

## 12. Test finali (20 min)

### Con l’utente standard (accesso fresco dopo il passo 7)

* **T1:** Apri Edge e naviga **`[F]`**.
* **T2:** Esegui `whoami /groups`, verifica l'assenza del gruppo `Administrators` **`[F]`**.
* **T3:** Scarica un eseguibile portatile (es. `putty.exe`) in `Downloads` e avvialo. Atteso: *"This program is blocked by group policy"* **`[F]`**.
* **T4:** Copia lo stesso eseguibile in `%LocalAppData%` o `%Temp%` e avvialo: deve risultare bloccato **`[F]`**.
* **T5:** Scarica l’MSI di 7-Zip da `7-zip.org` e fai doppio clic: deve risultare bloccato **`[F]`**.
* **T6:** Apri il prompt dei comandi ed esegui `winget --version`: deve risultare bloccato **`[F]`**.
* **T7:** Prova a modificare un’impostazione di sistema (es. apri `wf.msc`): richiesta credenziali admin. **Annulla**, non inserire la password **`[F]`**.
* **T8:** Apri `secpol.msc` o `gpedit.msc`: richiesta credenziali o accesso negato **`[F]`**.

### Con AdminIT (controllo di contrasto)

* **T9:** Lancia lo stesso eseguibile `.exe` e l’installer `.msi`: devono funzionare regolarmente **`[F]`**.

### Evidenza dal log

* Apri **Visualizzatore eventi > Registri applicazioni e servizi > Microsoft > Windows > AppLocker > EXE and DLL**: individua l'evento di blocco (**ID 8004**) **`[F]`**.

> **Nota:** Compila la colonna *Esito* del report solo con ciò che vedi realmente.

---

## Troubleshooting: Se qualcosa non blocca

* **Exe che gira da Downloads:** controlla *Imposizione* e le *regole predefinite* in AppLocker, verifica che il servizio `appidsvc` sia `RUNNING`, e assicurati di aver effettuato un nuovo accesso con l’utente standard dopo `gpupdate /force`.
* **MSI che gira:** verifica che il valore di registro `DisableMSI` sia `0x2`.
* **UAC che chiede credenziali:** premi **Annulla**, non inserire le credenziali di `AdminIT` dalla sessione standard, altrimenti il processo partirà con privilegi amministrativi e le regole non si applicheranno.
