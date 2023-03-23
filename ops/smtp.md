# Izgradite vlastiti SMTP poslužitelj za slanje pošte

## preambula

SMTP može izravno kupiti usluge od dobavljača oblaka, kao što su:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali push email u oblaku](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Također možete izgraditi vlastiti poslužitelj e-pošte - neograničeno slanje, niska ukupna cijena.

U nastavku korak po korak demonstriramo kako izgraditi vlastiti poslužitelj pošte.

## Izbor poslužitelja

Samostalni SMTP poslužitelj zahtijeva javni IP s otvorenim portovima 25, 456 i 587.

Često korišteni javni oblaci blokirali su ove priključke prema zadanim postavkama i možda ih je moguće otvoriti izdavanjem radnog naloga, ali to je ipak vrlo problematično.

Preporučam kupnju od domaćina koji ima otvorene te portove i podržava postavljanje obrnutih naziva domena.

Evo, ja preporučam [Contabo](https://contabo.com) .

Contabo je pružatelj usluga hostinga sa sjedištem u Münchenu, Njemačka, osnovan 2003. godine s vrlo konkurentnim cijenama.

Ako odaberete euro kao valutu kupnje, cijena će biti jeftinija (poslužitelj s 8 GB memorije i 4 CPU-a košta oko 529 juana godišnje, a početna naknada za instalaciju je besplatna godinu dana).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Prilikom naručivanja napomenite da `prefer AMD` , a poslužitelj s AMD CPU-om imat će bolje performanse.

U nastavku ću uzeti Contabo VPS kao primjer kako bih pokazao kako izgraditi vlastiti poslužitelj pošte.

## Konfiguracija Ubuntu sustava

Operativni sustav ovdje je Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Ako poslužitelj na ssh-u prikazuje `Welcome to TinyCore 13!` (kao što je prikazano na slici ispod), to znači da sustav još nije instaliran. Odspojite ssh i pričekajte nekoliko minuta da se ponovno prijavite.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Kada se pojavi `Welcome to Ubuntu 22.04.1 LTS` , inicijalizacija je dovršena i možete nastaviti sa sljedećim koracima.

### [Izborno] Inicijalizirajte razvojno okruženje

Ovaj korak nije obavezan.

Radi praktičnosti, instalaciju i konfiguraciju sustava ubuntu softvera stavio sam na [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Pokrenite sljedeću naredbu za instalaciju jednim klikom.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Kineski korisnici, umjesto toga upotrijebite sljedeću naredbu i jezik, vremenska zona itd. bit će automatski postavljeni.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo omogućuje IPV6

Omogućite IPV6 tako da SMTP također može slati e-poštu s IPV6 adresama.

uredi `/etc/sysctl.conf`

Izmijenite ili dodajte sljedeće retke

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Nastavite s [contabo vodičem: Dodavanje IPv6 povezivosti vašem poslužitelju](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Uredite `/etc/netplan/01-netcfg.yaml` , dodajte nekoliko redaka kao što je prikazano na donjoj slici (Contabo VPS zadana konfiguracijska datoteka već ima ove retke, samo ih uklonite iz komentara).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Zatim `netplan apply` kako bi izmijenjena konfiguracija stupila na snagu.

Nakon što je konfiguracija uspješna, možete koristiti `curl 6.ipw.cn` za pregled ipv6 adrese vaše vanjske mreže.

## Klonirajte ops spremišta konfiguracije

```
git clone https://github.com/wactax/ops.soft.git
```

## Generirajte besplatni SSL certifikat za naziv svoje domene

Za slanje pošte potreban je SSL certifikat za šifriranje i potpisivanje.

Koristimo [acme.sh](https://github.com/acmesh-official/acme.sh) za generiranje certifikata.

acme.sh je automatizirani alat za potpisivanje certifikata otvorenog koda,

Unesite konfiguracijsko skladište ops.soft, pokrenite `./ssl.sh` i mapa `conf` bit će stvorena u **gornjem direktoriju** .

Pronađite svog DNS pružatelja iz [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , uredite `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Zatim pokrenite `./ssl.sh 123.com` za generiranje `123.com` i `*.123.com` certifikata za naziv vaše domene.

Prvo pokretanje automatski će instalirati [acme.sh](https://github.com/acmesh-official/acme.sh) i dodati planirani zadatak za automatsku obnovu. Možete vidjeti `crontab -l` , postoji takav redak kako slijedi.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Put za generirani certifikat je nešto poput `/mnt/www/.acme.sh/123.com_ecc。`

Obnavljanje certifikata će pozvati skriptu `conf/reload/123.com.sh` , uredite ovu skriptu, možete dodati naredbe kao što je `nginx -s reload` za osvježavanje predmemorije certifikata povezanih aplikacija.

## Izgradite SMTP poslužitelj s chasquidom

[chasquid](https://github.com/albertito/chasquid) je SMTP poslužitelj otvorenog koda napisan na Go jeziku.

Kao zamjena za prastare programe poslužitelja pošte kao što su Postfix i Sendmail, chasquid je jednostavniji i lakši za korištenje, a lakši je i za sekundarni razvoj.

Pokrenite `./chasquid/init.sh 123.com` će se automatski instalirati jednim klikom (zamijenite 123.com imenom svoje domene pošiljatelja).

## Konfigurirajte DKIM za potpis e-pošte

DKIM se koristi za slanje potpisa e-pošte kako bi se spriječilo da se pisma tretiraju kao neželjena pošta.

Nakon što se naredba uspješno izvede, od vas će se tražiti da postavite DKIM zapis (kao što je prikazano u nastavku).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Samo dodajte TXT zapis u svoj DNS (kao što je prikazano u nastavku).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Pogledajte status usluge i zapisnike

 `systemctl status chasquid` Pregled statusa usluge.

Stanje normalnog rada prikazano je na donjoj slici

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` ili `journalctl -xeu chasquid` može vidjeti zapisnik grešaka.

## Obrnuta konfiguracija naziva domene

Obrnuti naziv domene omogućuje razlučivanje IP adrese u odgovarajući naziv domene.

Postavljanje obrnutog naziva domene može spriječiti prepoznavanje e-pošte kao neželjene pošte.

Kada je pošta primljena, primateljski poslužitelj izvršit će analizu obrnutog naziva domene na IP adresi poslužitelja koji šalje kako bi potvrdio ima li poslužitelj koji šalje ispravan naziv obrnute domene.

Ako poslužitelj koji šalje nema naziv obrnute domene ili ako naziv obrnute domene ne odgovara IP adresi poslužitelja koji šalje, poslužitelj koji prima može prepoznati e-poštu kao spam ili je odbiti.

Posjetite [https://my.contabo.com/rdns](https://my.contabo.com/rdns) i konfigurirajte kako je prikazano u nastavku

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Nakon postavljanja obrnutog naziva domene, ne zaboravite konfigurirati prosljeđivanje razlučivosti naziva domene ipv4 i ipv6 na poslužitelj.

## Uredite naziv hosta chasquid.conf

Izmijenite `conf/chasquid/chasquid.conf` na vrijednost obrnutog naziva domene.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Zatim pokrenite `systemctl restart chasquid` za ponovno pokretanje usluge.

## Sigurnosno kopirajte conf u git repozitorij

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Na primjer, sigurnosno kopiram mapu conf u svoj github proces na sljedeći način

Prvo napravite privatno skladište

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Uđite u conf imenik i pošaljite u skladište

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Dodajte pošiljatelja

trčanje

```
chasquid-util user-add i@wac.tax
```

Može dodati pošiljatelja

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Provjerite je li lozinka ispravno postavljena

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Nakon dodavanja korisnika, `chasquid/domains/wac.tax/users` bit će ažuriran, ne zaboravite ga poslati u skladište.

## DNS dodaj SPF zapis

SPF (Sender Policy Framework) je tehnologija za provjeru e-pošte koja se koristi za sprječavanje prijevare putem e-pošte.

Provjerava identitet pošiljatelja e-pošte provjerom podudara li se IP adresa pošiljatelja s DNS zapisima naziva domene za koju se predstavlja, sprječavajući prevarante da šalju lažne e-poruke.

Dodavanje SPF zapisa može spriječiti identificiranje e-pošte kao neželjene pošte koliko god je to moguće.

Ako vaš poslužitelj naziva domene ne podržava SPF tip, samo dodajte zapis tipa TXT.

Na primjer, SPF za `wac.tax` je sljedeći

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF za `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Imajte na umu da ovdje imam `include:_spf.google.com` , to je zato što ću kasnije konfigurirati `i@wac.tax` kao adresu za slanje u Google poštanskom sandučiću.

## DNS konfiguracija DMARC

DMARC je skraćenica od (Domain-based Message Authentication, Reporting & Conformance).

Koristi se za bilježenje odbijanja SPF-a (možda uzrokovano pogreškama u konfiguraciji ili se netko drugi pretvara da ste vi da šalje neželjenu poštu).

Dodajte TXT zapis `_dmarc` ,

Sadržaj je sljedeći

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Značenje svakog parametra je sljedeće

### p (Politika)

Označava kako postupati s e-porukama koje ne prođu SPF (Sender Policy Framework) ili DKIM (DomainKeys Identified Mail) provjeru. Parametar p može se postaviti na jednu od tri vrijednosti:

* ništa: Ne poduzima se nikakva radnja, samo se rezultat provjere vraća pošiljatelju putem mehanizma za izvješćivanje putem e-pošte.
* Karantena: stavite poštu koja nije prošla provjeru u mapu neželjene pošte, ali je nećete izravno odbiti.
* odbaci: Izravno odbaci e-poštu koja ne prođe provjeru.

### fo (Opcije neuspjeha)

Određuje količinu informacija koju vraća mehanizam izvješćivanja. Može se postaviti na jednu od sljedećih vrijednosti:

* 0: Prijavi rezultate provjere za sve poruke
* 1: Prijavite samo poruke koje ne prođu provjeru
* d: Prijavi samo neuspješne provjere naziva domene
* s: prijavi samo neuspjele provjere SPF-a
* l: Prijavi samo neuspješne provjere DKIM-a

### rua & ruf

* rua (URI izvješća za skupna izvješća): adresa e-pošte za primanje skupnih izvješća
* ruf (URI izvješća za forenzička izvješća): adresa e-pošte za primanje detaljnih izvješća

## Dodajte MX zapise za prosljeđivanje e-pošte na Google Mail

Budući da nisam mogao pronaći besplatni korporativni poštanski sandučić koji podržava univerzalne adrese (Catch-All, može primati sve e-poruke poslane na ovaj naziv domene, bez ograničenja prefiksa), upotrijebio sam chasquid za prosljeđivanje svih e-poruka u svoj Gmail poštanski sandučić.

**Ako imate vlastiti plaćeni poslovni poštanski sandučić, nemojte mijenjati MX i preskočite ovaj korak.**

Uredi `conf/chasquid/domains/wac.tax/aliases` , postavi poštanski sandučić za prosljeđivanje

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` označava sve poruke e-pošte, `i` je prefiks adrese e-pošte korisnika koji šalje poruku kreiranog iznad. Za prosljeđivanje pošte svaki korisnik mora dodati redak.

Zatim dodajte MX zapis (ovdje pokazujem izravno na adresu obrnutog naziva domene, kao što je prikazano u prvom retku na donjoj slici).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Nakon dovršetka konfiguracije, možete koristiti druge adrese e-pošte za slanje e-pošte na `i@wac.tax` i `any123@wac.tax` da vidite možete li primati e-poštu na Gmailu.

Ako nije, provjerite dnevnik chasquid ( `grep chasquid /var/log/syslog` ).

## Pošaljite e-poruku na i@wac.tax putem Google Mail-a

Nakon što je Google Mail primio poštu, prirodno sam se nadao da ću odgovoriti s `i@wac.tax` umjesto i.wac.tax@gmail.com.

Posjetite [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) i kliknite "Dodaj drugu adresu e-pošte".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Zatim unesite kod za provjeru primljen putem e-pošte koja je proslijeđena na.

Konačno, može se postaviti kao zadana adresa pošiljatelja (zajedno s mogućnošću odgovora s istom adresom).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Na ovaj način smo završili uspostavu SMTP mail servera te ujedno koristimo Google Mail za slanje i primanje e-pošte.

## Pošaljite testnu e-poštu da provjerite je li konfiguracija uspješna

Unesite `ops/chasquid`

Pokreni `direnv allow` instalaciju ovisnosti (direnv je instaliran u prethodnom procesu inicijalizacije s jednim ključem i ljusci je dodana kuka)

zatim trči

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Značenje parametara je sljedeće

* korisnik: SMTP korisničko ime
* pass: SMTP lozinka
* za: primatelja

Možete poslati testnu e-poštu.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Preporuča se korištenje Gmaila za primanje testnih e-poruka kako bi se provjerilo jesu li konfiguracije uspješne.

### TLS standardna enkripcija

Kao što je prikazano na donjoj slici, postoji ova mala brava, što znači da je SSL certifikat uspješno omogućen.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Zatim kliknite "Prikaži izvornu e-poštu"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Kao što je prikazano na donjoj slici, izvorna stranica Gmailove pošte prikazuje DKIM, što znači da je konfiguracija DKIM-a uspješna.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Provjerite Primljeno u zaglavlju originalne e-pošte i vidjet ćete da je adresa pošiljatelja IPV6, što znači da je IPV6 također uspješno konfiguriran.
