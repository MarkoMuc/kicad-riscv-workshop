# ESP32 board workshop

- Uradni repozitorij: https://github.com/bulicp/FRI-ESP32-PCB-workshop/tree/main

- Postavimo shemo. Kjer grupiramo komponente glede na funkcionalnost.
- Preko USB + USB C konektor zelimo da napajamo vse komponente na sistemi (procesor, ledice, itd.)

## Dan 1: USB-C in vse potrebne povezave.

Izhodisce je `00-template` iz git repozitorija delavnice:
- `.PCB`: printed circuit board.
- `.SCH`: schemas. Dve shemi: power and digital + schema, ki ju zdruzi.

Zaceli bomo z power:
- USB C in konektorji.
- Lahko se uporablja za napajanje, nekateri procesorji tudi uporablja za flashanje, v ta namem rabimo USB del povezave, ne le napajanje.

Komponente vpeljemo iz knjiznice komponent, ostale dodamo: Simboli in footprint.
- Simbol je samo simbol (za sheme).
- Footprint je odtis komponent na tiskanem vezju (kje se lota, prostor, tloris komponente).

Pritisni `a`, da odpres izbiro/dodajanje simbolov.
Nasa knjiznica je `fri`.

Zacnemo z USB konektorjem `USC_C_Receptacle_USB2.0_16P` iz `fri`.
Rdece blazinice so kjer se zalota.
Vidimo 11 povezav, a jih je v resnici 16.
Predstavljene so s zicko, ki se konca s krogom.
To je zato, kjer `VBUS` ima v resnici vec povezav?

Pini na USB-C konektorju:
- `VBUS` signal je zica, ki bo dostavila 5V, za napajanje.
    - Za vsak vir naptosti nas zanimata dve stvari: tok in moc.
    - USB C standard doloca 3 napajalne moci:
        - Vsaj 500mA toka in 5V napteost = 2,5 W.
        - Ter se drugi dve.
    - Za nas bo 500mA dovolj, a velikokrat potrebujemo vec. Sistemo mora vedeti koliko mu lahko oddaja napajlni vir.
        Npr. ce ni dovolj napajanja, se sistem naj ne bi brizgal. Kako pa to lahko ve?
        - Imamo dva pina `CC1` in `CC2`. Povesta ali napajlnik dostavlja 500 mA ali vec.
- `CC1` in `CC2` sta analogna, ne digitalna.
    - Odjamalec more vedeti s cim ga napaja. Obratno napajalnik mora vedeti porabo (ponor?).
        Npr. da ni samo `VBUS-GRND` povezava?
    - Potrebujemo 2 upora za delovanje napajanje (zaradi standarda).
        Dodamo upor z imenom simbola `R`.
        Prvi je `5.1K` povezan na `CC1`, isto naredimo za `CC2`.
        Spodnja dela uporov povezemo na `GND`.
- `SHIELD` in `GND` povezem na zemljo. To je lahko isti konektor za `GND`, locimo zato, da se zice ne krizajo.
- `SBUS1` in `SBUS2` ne rabimo, a je to potrebno povedati KiCad, tako da oznacimo, da prikljucek ni povezan.
    Pritisnemo `q` in povezemo na prikljucek.
- `D` prikljucki so diferencialni (komplementarni) prikljucki. Ko prenasam `0` je na `D_+: 1` in na `D_-: 0` ter obratno.
    To opravimo zaradi vpliva zunanjih dejavnikov (in posledicnega induciranega toka).
    Ce se napetost poveca zaradi šuma, se podobno zgodi na drugih zicah v blizini => signali po diferencialnih parih.
    Tako lahko locimo med `0` in `1` kljub sumu.
    Na primer vedno opravimo: `D_+ - D_-`.
    Po odstevanju se šum odstani.
    Tu imamo 4 pine, pri USB-C lahko zato vstavimo kabel ne glede na orientacijo.
- `VBUS` ima pri nas 5V, povezemo ga preko zice, ki nadaljuje na drugem listu (drugi shemi).
    To naredimo tako, da zico oznacimo (label).

Za vsako komponento moramo poiskati `footprint` in povedat kaksen je footprint, da lahko tiskamo.
Za upornike izberemo `R_01005_0402Metric`:
    - `0402` pomeni velikost (sirina) fizicnega rezistorja. Manjsi ko je, manj inducirane napetosti?
    - `_SMD` pomeni: surface mount device.

Vse komponente na vezju se prelije s lakom, ki ni prevoden.
USB-C konektor ni prelit, in je iz kovine.
Vezje je potrebno zasciti pred elektrostaticnem naboju, ki bi poskodoval komponente.
Zagotavlja se s komponentami, ki preosmerijo ta naboj v `GND`.
To komponento postavimo na zico iz `VBUS`, ki se sprozi ko je napetost vec kot 5V.

Uprabimo diodo, ki v neki tocki, ne glede na velikost toka, napetost ne spreminja.
Povezemo diodo `PESD5V0S1UL` in jo obrnemo z `r`, povezemo naprej na `GND` (spodnji del) ter na `VBUS` zico.

"Induciran kondenzator" je posledica vec plastne strukture cipov in komponent (plasti silicija).
Iscemo komponento z zelo nizko kapacitivnost, saj kodenzator deformira PWL signale (kot so podatkovni).
Pri nas uporabimo za zascito USB komponente, `USBLC6-2SC6`, na sliko lahko vidimo interno diodo, ki sciti.
Prikljucka `1-6` ter `3-4` sta ista. `5` gre na `VBUS`, `2` gre na `GND`.
Povezemo na `D_-` in `D_+` (ter iste povezave skupaj), dodamo globalne labele (pazi na ime naj se konca enako, mislim da KICAD to potrebuje).

Neko osnovno analizo vezja opravimo z `inspect - ERC`, ki nas opozori, da ni elektricnega vira.
Da ga eksplicitno ustvarimo, dodamo `PWR_FLAG`. Podobno naredimo tudi za `GND` (t.j. minus).

Nasledjni korak je sprememba 5V toka v 3.3V, ki ga sprejme ESP32.
Uporabi se DC/DC pretvornike, regulatorje, transformerje?
Za pretvorbo naptosti uporabimo regulator `XC6220B331MR`:
- Regulator ima pin za kontrolo `CE-Chip Enable`, ki ga prizge.
    To se opravi npr. ko je premajhna napetost (prazna baterija, da jo zascitimo).
- Mi ne rabimo CE, zato povezemo na VBUS z upornikom, ki naj bi bil po standardu 10K ohm.
Datasheet pravi, da potrebujemo se 2 kondenzatorja, eden za vhod, eden za izhod.
Pri kondenzatorjih je tezko dosesti pripisano kapacitivnost.
Zelimo cim manjsi footprint, tezje je kontrolikarti kapacitivnost (zaradi segrevanja itd.).
Manjsi footprint pomeni manjsa induktivnost.
Izberemo footprint `0805SMD`.
Ustvarili smo LDO 3.3V regulator. Ker se mocno segrevajo, je potrebno pravilno postaviti, da se pasivno ohlaja.

Uprabimo lahko tudi DC-DC konvertorji, ki so brezizgubni (oddajo jo v kondenzator, ko jo rabijo dvigniti, jo vzamejo iz kondenzatorja).
Sestavljajo jih tuljave.
Vsi cipi, vkljucno z nasim esp32, imajo DC-DC konverter (realiziran na cipu).
Dovod elektricne moci lahko tudi sestavlja notranja baterija, ter sistem, ki izbira med baterijo in direktno povezavo.

Sedaj imamo vse kar rabimo, da povezemo CPE, premaknemo se na `digital` shemo.
V nasem primeru imamo `esp32-C3-mini`, ki je RISC-V procesor.
Sicer je to SoC, kar pomeni, da ni samo procesor, temvec je modul.
Vsebuje anteno (za WIFI ter BLE), sistem za flashanje, RAM, itd.
Pricljucki na `ESP32-C3-MINI-1`:
- `3V3` in `GND` povezemo na +3.3V in zemljo.
- `EN` povezemo na label `EN`.
- `GPIO18/19` povezemo na `USBD+/-` labele.
- `GPIO21/20` povezemo na `TXD/RXD` labele.
- `GPIO3-5` povezemo na isto imenske labele.
- `GPIO6/7` sta vezana na LED diode.
- `GPIO2 + GPIO8 + GPIO9` so pri ESP32 kontrolni signali, ki povejo, da rabi CPE poiskati novo vrednost is flasha (ki je bil spremenjen?) za zagon.
    Imenujemo `GPIOX_BOOT`.
- `GPIO10` povezemo na labelo `USER_BTN`.


Zaceli bomo z ledicami:
- Ledice so tudi diode, sveti ko je V1 > V2 (kjer je V1 na strani (stranice) trikotnika).
- `V2` stran vezemo na GPIO pin, ker je to stran, ki je manjsa. To pomeni, da iz pina ne vlece toka.
    Tako je ta pin ponor, energija gre v ohisje.
    Drugace bi od CPE zahtevali veliko napetosti?
    Ce vezemo drugo stran (V1) na 3.3V, je problem v tem, da ni nobene upornosti na povezavi. Tok je zato lahko velik in naredi skodo.
    Ta tok je treba omeijti (z uporom). Upor dolocimo glede na datasheet ledice (razpon uporov/tokov).
    Mi izberemo 330 ohmov, manj ko je upora, bolj je svetla ledica.
- Ponovimo isto za `LED1`.

Nadaljujemo z gumbom:
- Uporabimo `SW_Push`.
- Mi zelimo, da ob pritisku na gumb, zaznamo niclo: `USER_BTN`.
    To je zaradi tega, ker je lazje iti iz 1 v 0?
    Drugi del vezemo na 3.3V, ter na nek upornik (drugace je kratki stik).
    Tako je brez pritiska 3.3V, ko pa pritisnemo gumb, gremo iz 3.3 proti ground.
    Ta tok se le porabi (direktno v zemljo), zelimo cim manjsi upor?
    To je pull-up upornik.
    Manjsi ko je upor, vec je vpliva suma, a je manjsi tok?
    Vecji upor, manj vpliva suma, visji tok??
    Izberemo 10k ohm.
- Isto za `EN` gumb na cipu (reset gumb).

Pini GPIO2,8,9:
- Za zagon preko flasha, za prenos v flash, itd.
- GPIO2 vedno 1.
- GPIO8 vedno 1.
- Ce je po reset: GPIO9 high, preberi program v flash.
- Ce je po reset: GPIO9 low, potem se pripravi na branje iz USB.
- To pomeni, da je pametno GPIO9 vezati na nek gumb.
- GPIO2 in GPIO8 lahko vezemo na 3.3V, uporabimo tudi upornik kot v primeru gumba.

Ostale GPIO prikljucke zelimo vezati na GPIO pin, ker so splosni za uporabo:
- Uporabimo `Conn_01x08` za 8 pinov, mi imamo 7 pinov prostih.
- Vrstni red ni pomemben, le lahko naredi koncno vezje kompleksno. Zato razporedimo GPIO cim lepse.
- Viska PIN vezemo na `GND`, ker je to vedno uporabno. Podobno tudi zelimo 3.3V za uporabo.
- Naredimo se en prikljucek, ki ima tri 3.3V pine, 4 x GND pine in 1 x VBUS (5.1V) pin.

Manjko nam se kondenzatorji na vseh napajalnih pinih.
Pri nas je to le eden.
Kondenzator ne sme biti velik (drugace vecji induciran tok?) ter mora biti cim blizje prikljucku.
Drugo pravilo je to, da vendo rabimo 2 kondenzatorja, drugi mora biti vecji (za dogodke, ki so daljsi kot 1ns).
Eden je `100nF`, drug je `10mF`.
Drugi kondenzator mora tudi imeti vecje ohisje (ker je vecji), tako izberemo drug footprint: `0402_1005` in `0805`.

Manjsi konenzator dodamo se drugod:
- Kjerkoli je gumb.


Footprints:
- `100nF` -> `0402_1005Metric`.
- `10uF` -> `0805_2012Metric`.

## Dan 2

`Tools - Bulk edit symbol fields` se uporablja za spreminjanje polj vseh simbolov v projektu.
Sprva bomo popravili vrednosti LED diod.

Vsak kondenzator ima induktivnost, manjsi imajo manjso, vecji visjo (zakaj je to pommebno, kva to pomen?)
Konektorji za pine normalno proizvalajec vezja ne doda, temvec samo opravi routing, nato sam zalotas konektor (oz pine).
Za `J2/J3` izberemo `Exclude from Bill Of Material` ter `Exclude from board`, to da proizvalajcu vedeti, da bomo opravili sami.
Na vsaki komponenti izberemo se `Do not populate`.
V polje `LCSC` vpisemo pripadajoco kodo izdelka na strani LCSC.
Dodamo lahko tudi `MPN` stevilko, to je identifikacijska stevila izdelka proizvalajca.
Previdno izberemo vse footprinte ter identifikacijske stevilke komponent.

Nasledjni korak je ustvariti pravo vezje.
Razvoj se nadalje opravlja v oknu `Switch PCB editor`.

Tiskano vezje je sestavljeno iz vec plasti:
- `.Cu` so bakrene plasti.
- Tiskanje (crke, znaki) je ena plast.
- Plast za masko.
- Plast za silk screen.
- Plast edge cuts (zunanji obris).
- Itd.

Preden lahko risemo izberemo plast.
Zacnemo z `Edge.cuts` in narisemo zeljene dimenzije vezja, na templatu je to ze narejeno.
Sprva prenesemo vse footprinte iz shem: `Update PCB from Schematic`.
Kako postavimo komponente ni strogo doloceno, moramo pa razmisliti o vlogi neke komponente in
postaviti logicno glede na to (npr. USB na rob itd.).
Na zectku pri premikanju izberemo samo `footprint`, da premaknemo vse.
Komponente vedno postavljamo na `F.Cu`: `Front Cu`.
Zakaj imamo 4 bakrene plasti?:
- Na vsaki bakreni plasti risemo linije.
- Vec kot je plasti, tezke je risanje linij, vezja so vecja, jumperji (resitev za krizajne zick), itd.
- `via`- "luknje", ki omogocajo prehajanje med plastmi.
- Danes imajo vsa vezja vsaj 2 plasti:sprednja in zadnja.
    - Pri dvo plastnih je tezava s pozicijo `GND` lokacij (dolge povratne poti),
        to resijo tako, da vsaki plasti dodamo se eno `GND` plast, oznaceni kot `In1.Cu`.
    - Velika prevodna plast, z majhno induktivnostjo?
    - Velikokrat je se dodatna `GND` plast, ki loci `power` in `data` plasti.
- `F.Cu` so signali, `In1.Cu` je `GND plane`, `In2.Cu` je `power plane`, `B.Cu` je signal plane.
- Danes je vecinoma minimalno 4 plasti.
- Pri RAMu nujno rabimo istocasne signale is procesorja, vsi signali morajo imeti isto inpendanco?, zato imajo svojo plast?

Pri nas:
- `F.Cu`: vse komponente.
- `In1.Cu`: `GND`.
- `In2.Cu`: power.
- `B.Cu`: probamo izogniti.

Na zacetku postavljamo funkcionalne bloke.
Ko postavljamo `GND` blazinice si zelimo najkrajse povezave (uporabimo `GND` plast).
Dodamo tudi fiduciale, to so krogci, ki pomagajo stroju, da se orientira.
Nato na `In1.Cu` dodamo `Filed zone`.
V `Board setup` nastavimo dimenzije vij, ter zick.
Nato narisemo poligon za `VBUS` na `In2.Cu`.
Povezave se vendo rise na `F.Cu`.

`USBD-` in `USBD+` sta diferencialna signala in morata priditi do CPE istocasno!
To pomeni, da morata biti obe zici enaki (po dolzini, in obliki, kolikor se seveda da to narediti).
Uporabi `Route - Route differential pair`.
Ko imamo dve zoni na isti plasti, on sprva zrise zone z visjo prioriteto.

Pri CPE se moramo zavedati, da je `Antenna Area` svoja cona, pod njo (ter L in D od nje), ne sme biti kovine.
Definiramo `Keep out` cone.

Zice po katerih gredo (okoli 1 Mb na sekundo) ne prekrivamo z viami.
Dodatno pravilo je, da cim vec zapolni z `GND` polju.

## Dan 3

Ne zelimo `GND` otokov.
Napovnemo ostali prostor z viami.
