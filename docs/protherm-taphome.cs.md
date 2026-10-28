# Protherm GeniaAir Split v TapHome přes eBUS

Ověřeno 28. 10. 2026: tepelné čerpadlo **Protherm GeniaAir Split HA 10-5 OS + HA 12-5 WSB**, regulátor MiPro Sense, eBUS Adapter Stick C6 přes Wi-Fi a Debian 13 amd64. Oficiální **ebusd 26.1 s MQTT** běží jako systemd služba; TapHome přijímá hodnoty a posílá čtecí/zapisovací požadavky přes MQTT broker. Home Assistant ani Docker nejsou potřeba.

## Prerekvizity

- [Protherm GeniaAir Split – výrobce](https://www.protherm.cz/pro-nase-zakazniky/produkty/tepelne-cerpadlo-geniaair-split-14592.html) a [prospekt s hydraulickým modulem WSB](https://www.protherm.cz/files/downloads/prospekty/prospekt-geniaair-split-26052025-3020929.pdf).
- [eBUS Adapter Stick C6 – oficiální hardware a zapojení](https://adapter.ebusd.eu/v5-c6/stick). Zapojení na eBUS proveď podle dokumentace adaptéru a čerpadla. Ve webovém rozhraní C6 nastav enhanced protokol `ens`, TCP port `9999` a stabilní LAN adresu.
- Linuxový server; zde HP t655 s Debianem 13. [ebusd – oficiální repozitář](https://github.com/john30/ebusd), [vydání 26.1](https://github.com/john30/ebusd/releases/tag/26.1).
- MQTT broker v LAN. Může běžet [přímo v TapHome](https://taphome.com/en/docs/integration/packet-parser/mqtt-broker/) nebo externě; níže se připojují ebusd i TapHome ke **stejnému brokeru**.

### Packet Parser šablony v TapHome

V této sestavě se používají dvě oddělené šablony:

1. **eBUS Adapter Shield C6 Stick** — [oficiální šablona a popis](https://taphome.com/en/compatibility/ebus-adapter-shield-c6-stick/), [oficiální repozitář šablon](https://github.com/taphome-official/packetparser_templates). Sleduje stav adaptéru; samotné hodnoty čerpadla řeší další modul. Její MQTT prefix ověř podle skutečného nastavení adaptéru. Tento návod ponechává C6 v režimu `ens` s externím ebusd; kvůli šabloně není potřeba přepínat na micro-ebusd.
2. **Protherm MiPro Sense Remote** — [MQTT XML šablona z této instalace](../examples/protherm-taphome/Protherm-MiPro-Sense-Remote.xml). Obsahuje čtení tlaku, vlhkosti, venkovní teploty, měsíčních energií, stav čerpadla a tři termostatové prvky. Pokojový termostat zóny 2 má také zápis `Z2DayTemp/set`; zbývající dva termostatové prvky mají zapisovací skript prázdný.

Oficiální šablony jsou v aplikaci pod **Hardware → Packet Parser → Přidat ze šablony**. Vlastní přiložené XML importuj jako Packet Parser šablonu; poté nastav broker místo `MQTT_BROKER_IP`, port `1883` a root topic `GeniaAir-split/#`. Přihlašovací údaje nastav podle svého brokeru. XML neobsahuje heslo ani identifikátor domácnosti.

Šablona zachovává původní skripty, převody, intervaly čtení i mapování `Te`/`Se`; nový import do TapHome nebyl automatizovaně testován. Prvek „Hlavní kohout topení“ čte `Hc1PumpStatus` — jde o indikaci stavu čerpadla, nikoli povel ventilu. Při importu do jiné instalace ověř význam prvků a zónu 2 před použitím zápisu. C6 monitorovací šablona je doplněk; pro přenos hodnot čerpadla je podstatný MQTT modul Protherm a níže popsaný ebusd.

## 1. Vyber správné definice

Označení čerpadla nestačí. Tato instalace má následující skutečné identifikace:

| eBUS adresa | ID | SW | HW | CSV |
| --- | --- | --- | --- | --- |
| `08` | HMU00 | 0902 | 5103 | `08.hmu.HW5103.csv` |
| `15` | CTLS3 | 0709 | 3804 | `15.ctls3.csv` |
| `76` | VWZIO | 0902 | 5103 | `76.vwzio.csv` |

Identifikaci lze přečíst pomocí `ebusctl scan 08`, `ebusctl scan 15`, `ebusctl scan 76`, až je daemon připojený. Jsou to aktivní identifikační dotazy, nikoli změny nastavení. U jiných ID/SW/HW nejprve vyber odpovídající konfiguraci; tento profil není univerzální pro všechny Prothermy.

Zdroje: [upstream ebusd-configuration](https://github.com/john30/ebusd-configuration), [použitá CTLS3 větev](https://github.com/Damecek/ebusd-configuration/tree/ctls3-cooling-setpoints), [PR #611](https://github.com/john30/ebusd-configuration/pull/611). CTLS3 má vlastní definici; nezaměňuj ji za VRC700.

[Adresář příkladů](../examples/protherm-taphome/) obsahuje ověřený runtime profil, jednotku a šablony prostředí. CSV pocházejí z commitu [`7796675`](https://github.com/Damecek/ebusd-configuration/commit/779667594669fde6e9f7bf2839b96ace19f61fb0). Jeho lockfile připíná `asset-emitter 0.79.1` kompatibilní s compilerem 1.12; sestavení zdrojů vyžaduje Node 22, `npm ci` a `npm run compile`. Pro běh přiloženého profilu Node není potřeba.

**Odlišnost profilu:** u 14 energetických zpráv VWZIO je poslední nulový bajt osmibajtového ID převeden na ignorované pole požadavku `m,IGN:1`. Bajty na sběrnici i názvy zpráv zůstávají stejné; ebusd 26.1 tak projde limitem sedmi bajtů ID. Není to úprava binárky ebusd. Celý nefiltrovaný upstream CSV strom tímto testem neprošel; nekopíruj jej místo ověřeného profilu. [Seznam a kontrolní součty](../examples/protherm-taphome/provenance.json).

## 2. Nainstaluj oficiální daemon a profil

Následující příklad je pro **novou instalaci Debian 13 amd64**. Na existující instalaci nejprve zazálohuj konfiguraci a zastav starý ebusd; proti jednomu C6 nepouštěj dvě instance.

```sh
sudo apt-get install --no-install-recommends ca-certificates curl git
curl -fLO https://github.com/john30/ebusd/releases/download/26.1/ebusd-26.1_amd64-trixie_mqtt1.deb
sha256sum -c <<'SHA256'
b1f595970b156b0fdb77132527abb9a8e0b71855bca7ed2bac7fe93a6430283f  ebusd-26.1_amd64-trixie_mqtt1.deb
SHA256
sudo systemctl mask ebusd.service
sudo apt-get install ./ebusd-26.1_amd64-trixie_mqtt1.deb

git clone --branch docs/protherm-taphome https://github.com/Damecek/ebusd-configuration.git
cd ebusd-configuration
id -u ebusd >/dev/null 2>&1 || sudo useradd --system --user-group --home-dir /nonexistent --shell /usr/sbin/nologin ebusd
sudo install -d -o root -g ebusd -m 0750 /etc/ebusd/telemetry-csv /etc/ebusd/telemetry-csv/vaillant
sudo install -o root -g ebusd -m 0640 examples/protherm-taphome/csv/broadcast.csv /etc/ebusd/telemetry-csv/
sudo install -o root -g ebusd -m 0640 examples/protherm-taphome/csv/vaillant/*.csv /etc/ebusd/telemetry-csv/vaillant/
sudo install -m 0600 examples/protherm-taphome/telemetry.env.example /etc/ebusd/telemetry.env
sudo install -m 0600 examples/protherm-taphome/mqtt.env.example /etc/ebusd/mqtt.env
sudoedit /etc/ebusd/telemetry.env /etc/ebusd/mqtt.env
```

V šablonách nahraď `C6_IP`, `MQTT_BROKER_IP`, `MQTT_USER` a `REPLACE_WITH_PASSWORD` vlastními údaji. Heslo patří do chráněného souboru, nikoli do Git repozitáře nebo argumentů příkazové řádky. Broker ani C6 nevystavuj do internetu.

```sh
sudo -u ebusd ebusd --checkconfig --configpath=/etc/ebusd/telemetry-csv
sudo systemctl unmask ebusd.service
sudo install -m 0644 examples/protherm-taphome/ebusd.service /etc/systemd/system/ebusd.service
sudo systemctl daemon-reload
sudo systemd-analyze verify /etc/systemd/system/ebusd.service
sudo systemctl enable --now ebusd.service
```

Kontrola profilu má najít 567 zpráv. Jednotka používá `--mqttjson`, prefix `GeniaAir-split`, lokální CSV a normální polling 5 s. Běží bez root práv, startuje po bootu a po pádu se restartuje za 15 s. HTTP je vypnuté výchozí hodnotou; příkazový port 8888 poslouchá jen na localhostu. U 26.1 nezadávej explicitní `--httpport=0` ani `--mqttport=0`: parser je odmítá.

`--readonly` je vhodné pro čistě pasivní diagnostiku, ale zablokuje i odeslání čtecích dotazů z TapHome. Běžná integrace jej proto nepoužívá a ponechává čtení i zápis podle existujících definic. [Oficiální parametry ebusd](https://github.com/john30/ebusd/wiki/2.-Run).

## 3. Nastav TapHome

Použij MQTT XML šablonu z prerekvizit. V **Hardware → Packet Parser → MQTT** nastav stejný broker, port `1883` a přihlašovací údaje. Root topic je `GeniaAir-split/#`. Pokud chceš modul sestavit ručně, následující skripty ukazují stejný princip.

Příklad pro **WaterPressure**, analogový vstup. Čtecí skript:

```text
MQTTPUBLISH("GeniaAir-split/ctls3/WaterPressure/get", "");
```

Skript posluchače:

```text
IF RECEIVEDMSG.TOPIC = "GeniaAir-split/ctls3/WaterPressure"
    An := PARSEJSON(TOSTRING(RECEIVEDMSG.PAYLOAD), "value.value");
END
```

Odpověď ebusd má například `{"value":{"value":1.4}}` — hodnota je 1,4 bar. U analogového vstupu nastav převod tak, aby `An = 1.4` zobrazil **1,4 bar**; ve zdejší konfiguraci odpovídá 140 % hodnotě 1,4 bar. `--mqttjson=short` tento parser rozbije, protože nemá vnořené `value.value`.

Stejný vzor fungoval pro `Hc1FlowTemp`, `HwcStorageTemp`, `HwcTempDesired` a `RoomHumidity`. Pro `Hc1PumpStatus` se čte číselný stav 0/1; použij odpovídající typ zařízení v TapHome. Každá zpráva může mít jinak pojmenovaná pole — parser přizpůsob skutečnému JSON.

Zápis používá topic se suffixem `/set`, pouze u zapisovatelných zpráv a s odpovídající hodnotou. Je to skutečný zásah do topení, ne test spojení. Při ověřování instalace není potřeba nic zapisovat. [Oficiální MQTT rozhraní ebusd](https://github.com/john30/ebusd/wiki/3.3.-MQTT-client), [integrace MQTT v TapHome](https://taphome.com/en/compatibility/mqtt-guide/) a [jazyk skriptů](https://taphome.com/en/docs/script/).

## 4. Ověř provoz

```sh
systemctl is-enabled ebusd.service
systemctl status ebusd.service
ebusctl info
journalctl -u ebusd.service --since=-5min --no-pager
```

Očekávej `enabled`, `active (running)`, `signal: acquired` a odpovědi na požadavky TapHome. Na této sestavě byly ověřeny tlak, teploty, vlhkost i automatické zotavení procesu. Fyzické odpojení adaptéru a reboot nebyly součástí testu. Některé zprávy HMU `Errorhistory` stále vykazují chyby dekódování; tento návod jejich opravu nepokrývá.

Pro odstavení integrace použij `sudo systemctl stop ebusd.service`. Konfigurace topení tím není přenastavena.
