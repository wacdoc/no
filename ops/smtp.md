# Bygg din egen SMTP-postserver

## ingress

SMTP kan kjøpe tjenester direkte fra skyleverandører, for eksempel:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali sky e-post push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Du kan også bygge din egen e-postserver - ubegrenset sending, lave totalkostnader.

Nedenfor viser vi trinn for trinn hvordan du bygger vår egen e-postserver.

## Servervalg

Den selvdrevne SMTP-serveren krever en offentlig IP med porter 25, 456 og 587 åpne.

Vanlige offentlige skyer har blokkert disse portene som standard, og det kan være mulig å åpne dem ved å utstede en arbeidsordre, men det er tross alt veldig plagsomt.

Jeg anbefaler å kjøpe fra en vert som har disse portene åpne og støtter oppsett av omvendte domenenavn.

Her anbefaler jeg [Contabo](https://contabo.com) .

Contabo er en hostingleverandør basert i München, Tyskland, grunnlagt i 2003 med svært konkurransedyktige priser.

Hvis du velger Euro som kjøpsvaluta, vil prisen være billigere (en server med 8 GB minne og 4 CPUer koster ca. 529 yuan per år, og den første installasjonsavgiften er gratis i ett år).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Når du legger inn en bestilling, merk `prefer AMD` , og ​​serveren med AMD CPU vil ha bedre ytelse.

I det følgende vil jeg ta Contabos VPS som eksempel for å demonstrere hvordan du bygger din egen e-postserver.

## Ubuntu systemkonfigurasjon

Operativsystemet her er Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Hvis serveren på ssh viser `Welcome to TinyCore 13!` (som vist i figuren nedenfor), betyr det at systemet ikke er installert ennå. Koble fra ssh og vent noen minutter for å logge på igjen.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Når `Welcome to Ubuntu 22.04.1 LTS` vises, er initialiseringen fullført, og du kan fortsette med følgende trinn.

### [Valgfritt] Initialiser utviklingsmiljøet

Dette trinnet er valgfritt.

For enkelhets skyld legger jeg installasjonen og systemkonfigurasjonen av ubuntu-programvaren i [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Kjør følgende kommando for å installere med ett klikk.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Kinesiske brukere, vennligst bruk følgende kommando i stedet, og språket, tidssonen osv. vil bli satt automatisk.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo muliggjør IPV6

Aktiver IPV6 slik at SMTP også kan sende e-post med IPV6-adresser.

rediger `/etc/sysctl.conf`

Endre eller legg til følgende linjer

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Følg opp med [kontaktveiledningen: Legge til IPv6-tilkobling til serveren din](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Rediger `/etc/netplan/01-netcfg.yaml` , legg til noen linjer som vist i figuren nedenfor (Contabo VPS standard konfigurasjonsfil har allerede disse linjene, bare fjern kommentarer).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Deretter `netplan apply` for å få den endrede konfigurasjonen til å tre i kraft.

Etter at konfigurasjonen er vellykket, kan du bruke `curl 6.ipw.cn` for å se ipv6-adressen til ditt eksterne nettverk.

## Klone konfigurasjonslageret ops

```
git clone https://github.com/wactax/ops.soft.git
```

## Generer et gratis SSL-sertifikat for ditt domenenavn

Sending av e-post krever et SSL-sertifikat for kryptering og signering.

Vi bruker [acme.sh](https://github.com/acmesh-official/acme.sh) for å generere sertifikater.

acme.sh er et automatisert sertifikatsigneringsverktøy med åpen kildekode,

Gå inn i konfigurasjonsvarehuset ops.soft, kjør `./ssl.sh` , og en `conf` mappe vil bli opprettet i **den øvre katalogen** .

Finn din DNS-leverandør fra [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , rediger `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Kjør deretter `./ssl.sh 123.com` for å generere `123.com` og `*.123.com` sertifikater for ditt domenenavn.

Den første kjøringen vil automatisk installere [acme.sh](https://github.com/acmesh-official/acme.sh) og legge til en planlagt oppgave for automatisk fornyelse. Du kan se `crontab -l` , det er en slik linje som følger.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Banen for det genererte sertifikatet er noe sånt som `/mnt/www/.acme.sh/123.com_ecc。`

Sertifikatfornyelse vil kalle `conf/reload/123.com.sh` script, rediger dette skriptet, du kan legge til kommandoer som `nginx -s reload` for å oppdatere sertifikatbufferen til relaterte applikasjoner.

## Bygg SMTP-server med chasquid

[chasquid](https://github.com/albertito/chasquid) er en åpen kildekode SMTP-server skrevet på Go-språket.

Som en erstatning for de eldgamle postserverprogrammene som Postfix og Sendmail, er chasquid enklere og enklere å bruke, og det er også lettere for sekundær utvikling.

Kjør `./chasquid/init.sh 123.com` vil bli installert automatisk med ett klikk (erstatt 123.com med ditt avsende domenenavn).

## Konfigurer e-postsignatur DKIM

DKIM brukes til å sende e-postsignaturer for å forhindre at brev behandles som spam.

Etter at kommandoen er kjørt, vil du bli bedt om å sette DKIM-posten (som vist nedenfor).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Bare legg til en TXT-post til din DNS (som vist nedenfor).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Se tjenestestatus og logger

 `systemctl status chasquid` Vis tjenestestatus.

Tilstanden for normal drift er som vist i figuren nedenfor

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` eller `journalctl -xeu chasquid` kan se feilloggen.

## Omvendt konfigurasjon av domenenavn

Det omvendte domenenavnet er for å tillate at IP-adressen blir løst til det tilsvarende domenenavnet.

Å angi et omvendt domenenavn kan forhindre at e-poster blir identifisert som spam.

Når e-posten er mottatt, vil mottaksserveren utføre omvendt domenenavnanalyse på IP-adressen til avsenderserveren for å bekrefte om avsenderserveren har et gyldig omvendt domenenavn.

Hvis avsenderserveren ikke har et omvendt domenenavn, eller hvis det omvendte domenenavnet ikke samsvarer med IP-adressen til avsenderserveren, kan mottakerserveren gjenkjenne e-posten som spam eller avvise den.

Besøk [https://my.contabo.com/rdns](https://my.contabo.com/rdns) og konfigurer som vist nedenfor

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Etter å ha angitt det omvendte domenenavnet, husk å konfigurere videreoppløsningen av domenenavnet ipv4 og ipv6 til serveren.

## Rediger vertsnavnet til chasquid.conf

Endre `conf/chasquid/chasquid.conf` til verdien av det omvendte domenenavnet.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Kjør deretter `systemctl restart chasquid` for å starte tjenesten på nytt.

## Sikkerhetskopier conf til git repository

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

For eksempel sikkerhetskopierer jeg conf-mappen til min egen github-prosess som følger

Opprett et privat lager først

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Gå inn i conf-katalogen og send til lageret

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Legg til avsender

løpe

```
chasquid-util user-add i@wac.tax
```

Kan legge til avsender

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Kontroller at passordet er riktig angitt

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Etter å ha lagt til brukeren vil `chasquid/domains/wac.tax/users` bli oppdatert, husk å sende det til lageret.

## DNS legg til SPF-post

SPF ( Sender Policy Framework ) er en e-postbekreftelsesteknologi som brukes for å forhindre e-postsvindel.

Den verifiserer identiteten til en e-postavsender ved å sjekke at avsenderens IP-adresse samsvarer med DNS-postene til domenenavnet den hevder å være, og forhindrer svindlere i å sende falske e-poster.

Å legge til SPF-poster kan forhindre at e-poster blir identifisert som spam så mye som mulig.

Hvis din domenenavnserver ikke støtter SPF-type, legg til TXT-typepost.

For eksempel er SPF for `wac.tax` som følger

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF for `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Merk at jeg har `include:_spf.google.com` her, dette er fordi jeg vil konfigurere `i@wac.tax` som avsenderadresse i Google-postboksen senere.

## DNS-konfigurasjon DMARC

DMARC er forkortelsen for (Domain-based Message Authentication, Reporting & Conformance).

Den brukes til å fange opp SPF-sprett (kanskje forårsaket av konfigurasjonsfeil, eller noen andre utgir seg for å være deg for å sende spam).

Legg til TXT-post `_dmarc` ,

Innholdet er som følger

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Betydningen av hver parameter er som følger

### p (Retningslinjer)

Indikerer hvordan du håndterer e-poster som ikke bekrefter SPF (Sender Policy Framework) eller DKIM (DomainKeys Identified Mail). P-parameteren kan settes til en av tre verdier:

* ingen: Ingen handling iverksettes, bare bekreftelsesresultatet blir tilbakeført til avsenderen via e-postrapporteringsmekanismen.
* Karantene: Legg e-posten som ikke har bestått bekreftelsen i spam-mappen, men vil ikke avvise e-posten direkte.
* avvis: Avvis direkte e-poster som mislykkes med bekreftelse.

### fo (feilalternativer)

Angir mengden informasjon som returneres av rapporteringsmekanismen. Den kan settes til en av følgende verdier:

* 0: Rapporter valideringsresultater for alle meldinger
* 1: Rapporter bare meldinger som mislykkes med verifisering
* d: Rapporter bare feil ved bekreftelse av domenenavn
* s: rapporter bare SPF-verifiseringsfeil
* l: Rapporter bare DKIM-verifiseringsfeil

### rua & ruf

* rua (Rapporterings-URI for aggregerte rapporter): E-postadresse for mottak av aggregerte rapporter
* ruf (Reporting URI for Forensic reports): e-postadresse for å motta detaljerte rapporter

## Legg til MX-poster for å videresende e-poster til Google Mail

Fordi jeg ikke kunne finne en gratis bedriftspostkasse som støtter universelle adresser (Catch-All, kan motta alle e-poster sendt til dette domenenavnet, uten begrensninger på prefikser), brukte jeg chasquid til å videresende alle e-poster til Gmail-postboksen min.

**Hvis du har din egen betalte bedriftspostkasse, må du ikke endre MX og hoppe over dette trinnet.**

Rediger `conf/chasquid/domains/wac.tax/aliases` , angi postkasse for videresending

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` indikerer alle e-poster, `i` er e-postadresseprefikset til avsenderbrukeren opprettet ovenfor. For å videresende e-post må hver bruker legge til en linje.

Legg så til MX-posten (jeg peker direkte på adressen til det omvendte domenenavnet her, som vist i første linje i figuren nedenfor).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Etter at konfigurasjonen er fullført, kan du bruke andre e-postadresser til å sende e-post til `i@wac.tax` og `any123@wac.tax` for å se om du kan motta e-poster i Gmail.

Hvis ikke, sjekk chasquid-loggen ( `grep chasquid /var/log/syslog` ).

## Send en e-post til i@wac.tax med Google Mail

Etter at Google Mail mottok e-posten, håpet jeg naturligvis å svare med `i@wac.tax` i stedet for i.wac.tax@gmail.com.

Gå til [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) og klikk "Legg til en annen e-postadresse".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Deretter skriver du inn bekreftelseskoden mottatt av e-posten som ble videresendt til.

Til slutt kan den settes som standard avsenderadresse (sammen med muligheten til å svare med samme adresse).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

På denne måten har vi fullført etableringen av SMTP-postserveren og bruker samtidig Google Mail til å sende og motta e-post.

## Send en test-e-post for å sjekke om konfigurasjonen er vellykket

Skriv inn `ops/chasquid`

Kjør `direnv allow` å installere avhengigheter (direnv har blitt installert i forrige én-tasts initialiseringsprosess og en krok er lagt til skallet)

så løp

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Betydningen av parametrene er som følger

* bruker: SMTP-brukernavn
* pass: SMTP-passord
* til: mottaker

Du kan sende en test-e-post.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Det anbefales å bruke Gmail til å motta test-e-poster for å sjekke om konfigurasjonene er vellykkede.

### TLS standard kryptering

Som vist i figuren nedenfor er det denne lille låsen, som betyr at SSL-sertifikatet har blitt aktivert.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Klikk deretter "Vis original e-post"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Som vist i figuren nedenfor, viser Gmails opprinnelige e-postside DKIM, noe som betyr at DKIM-konfigurasjonen er vellykket.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Sjekk Mottatt i overskriften på den originale e-posten, og du kan se at avsenderadressen er IPV6, noe som betyr at IPV6 også er konfigurert.
