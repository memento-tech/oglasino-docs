# Politika privatnosti — Oglasino

**Poslednje ažuriranje:** 10.06.2026.

---

## 1. Ko smo mi

Oglasino je onlajn platforma za male oglase za kupovinu, prodaju i razmenu novih i polovnih proizvoda. Platforma posluje preko dva državna portala — Srbija (rs) i Crna Gora (me) — i dostupna je i posetiocima iz drugih zemalja, uključujući Evropsku uniju. Oglasino ne obrađuje niti posreduje u plaćanjima između korisnika; sve transakcije se dogovaraju i sprovode direktno između kupaca i prodavaca.

Oglasino možete koristiti putem našeg veb-sajta i putem naših izvornih mobilnih aplikacija za iOS i Android. Ova Politika privatnosti odnosi se na sve njih. Mobilne aplikacije nude iste funkcije kao veb-sajt i obrađuju vaše podatke na isti način, osim tamo gde uređaj pravi razliku — dozvole aplikacije (kao što je kamera) i čuvanje podataka na uređaju — što izričito opisujemo u odeljcima 2.16 i 7.

Rukovalac podacima o ličnosti za Oglasino je:

**Igor Stojanović**
Bulevar 12. februar 32, 18000 Niš, Srbija
privacy@oglasino.com

Oglasino trenutno posluje kao preduzetnik pod gore navedenim imenom. Vlasnik namerava da u budućnosti osnuje privredno društvo kada platforma pokaže održiv rast. Kada se to dogodi, ova Politika privatnosti biće ažurirana kako bi odražavala novo pravno lice, a vi ćete o tome biti obavešteni putem platforme.

[LAWYER REVIEW: The platform is offered to EU visitors while the controller is established outside the EU/EEA. Assess whether an EU representative under GDPR Art. 27 must be appointed; if so, appoint one and add the representative's identity and contact details here.]

Ova Politika privatnosti objavljuje se na srpskom i engleskom jeziku. U slučaju bilo kakve neusaglašenosti između jezičkih verzija, merodavna je verzija na srpskom jeziku. [LAWYER REVIEW: confirm that the authoritative-language approach for the Privacy Policy should match the Terms of Use (§17).]

---

## 2. Podaci koje prikupljamo i zašto

Ovaj odeljak opisuje svaku kategoriju podataka o ličnosti koje Oglasino prikuplja, zašto ih prikupljamo i pravni osnov na koji se oslanjamo u skladu sa Opštom uredbom o zaštiti podataka (GDPR), srpskim Zakonom o zaštiti podataka o ličnosti i crnogorskim propisima o zaštiti podataka o ličnosti.

### 2.1 Podaci o nalogu

Kada kreirate nalog na Oglasinu, prikupljamo i čuvamo:

- **Imejl adresa** — neophodna za kreiranje naloga, prijavljivanje i obaveštenja u vezi sa nalogom.
- **Identifikator za autentifikaciju (Firebase UID)** — interni identifikator koji izdaje naš provajder autentifikacije, Firebase Authentication. Ovaj identifikator povezuje vaš nalog u našoj bazi podataka sa vašim zapisom za autentifikaciju.
- **Prikazano ime** — ime prikazano na vašem javnom profilu i oglasima. Birate ga prilikom registracije imejl adresom; za naloge kreirane putem Google prijavljivanja preuzima se iz vašeg Google profila. (Ako prikazano ime ikada nije dostupno prilikom registracije, kao rezerva se može upotrebiti deo vaše imejl adrese ispred znaka „@”.) Prikazano ime možete promeniti u svako doba u podešavanjima profila.
- **Provajder prijavljivanja** — da li ste se registrovali imejlom ili Google-om.
- **Status verifikacije imejla** — da li je vaš imejl verifikovan.
- **Lozinka** — vaša lozinka se nikada ne šalje Oglasinu niti je on čuva. Njome rukuje isključivo Firebase Authentication, naš provajder autentifikacije, direktno sa vašeg uređaja ili iz pregledača. Oglasino nema pristup njoj.
- **Vremenske oznake kreiranja naloga i poslednjeg ažuriranja.**

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b) GDPR-a) — ne možemo da pružimo uslugu zasnovanu na nalogu bez ovih podataka.

### 2.2 Podaci o profilu

Pored podataka o nalogu, vaš profil može sadržati:

- **Broj telefona** — opciono. Možete ga uneti u podešavanjima profila.
- **Profilna slika (avatar)** — ako se prijavite putem Google-a i vaš Google nalog ima profilnu sliku, pravimo kopiju te slike i čuvamo je na našoj infrastrukturi (Cloudflare R2). Ako nemate Google sliku, platforma prikazuje automatski generisan avatar na osnovu inicijala (prvo slovo vašeg prikazanog imena na obojenoj pozadini) — ovaj generisani avatar nije sačuvana slika i ne sadrži nikakve zasebne podatke o ličnosti. Profilnu sliku možete otpremiti, zameniti ili ukloniti u svako doba u podešavanjima.
- **Lični opis (biografija)** — opcioni slobodni tekst koji možete dodati o sebi. Ako svoju biografiju napišete na jednom jeziku, automatski je prevodimo na ostale podržane jezike platforme kako bi drugi korisnici mogli da je čitaju na svom jeziku (vidi odeljak 2.8 o automatskom prevođenju).
- **Ocena naloga** — numerička ocena izvedena iz recenzija koje ste dobili od drugih korisnika.
- **Status verifikacije** — interna oznaka koja pokazuje da li je platforma verifikovala vaš nalog.

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b)) — podaci o profilu deo su usluge zasnovane na nalogu na koju ste se prijavili.

### 2.3 Podaci o lokaciji

Da biste kreirali ili pregledali oglase, platforma mora da zna koji državni portal koristite i vaše lokalno područje. Prikupljamo:

- **Državni portal (Srbija ili Crna Gora)** — izabran kada kreirate svoj prvi oglas.
- **Region** — izabran kada kreirate svoj prvi oglas.
- **Grad** — izabran kada kreirate svoj prvi oglas.

Vaša lokacija se postavlja na nivou vašeg naloga, a ne po pojedinačnom oglasu. Svi vaši oglasi povezani su sa istim regionom i gradom kao i vaš nalog; platforma ne podržava oglašavanje robe koja se nalazi u drugom regionu ili državi od prodavca. Ne koristimo GPS vašeg uređaja niti bilo koju drugu automatsku geolokaciju — uključujući i u našim mobilnim aplikacijama; vaše područje proizilazi isključivo iz portala, regiona i grada koje izaberete. (Fotografije koje otpremate mogu same po sebi sadržati metapodatke o lokaciji koje dodaje vaš fotoaparat — vidi odeljak 2.4.)

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b)) — Oglasino je geografski ograničena platforma za lokalne oglase; funkcija oglasa ne može da radi bez vaše lokacije.

### 2.4 Podaci o oglasu

Kada kreirate oglas, prikupljamo i čuvamo:

- **Naslov i opis** — koje pišete vi, na bilo kom od podržanih jezika platforme. Automatski ih prevodimo na ostale podržane jezike platforme (vidi odeljak 2.8).
- **Cena i valuta** — srpski dinar (RSD) i evro (EUR) podržani su na srpskom portalu; evro (EUR) je podržan na crnogorskom portalu.
- **Oznaka „Besplatno”** — da li je oglas za artikal koji poklanjate bez naknade (koristi se u Džabe zoni platforme).
- **Kategorija i potkategorija** — izabrane iz fiksne taksonomije.
- **Atributi specifični za kategoriju** — strukturirana polja koja se razlikuju po kategoriji (na primer, veličina, boja, godina za odgovarajuće kategorije).
- **Fotografije** — slike koje otpremate, sačuvane na Cloudflare R2 (vidi odeljak 4 o trećim stranama). Fotografije mogu sadržati skrivene metapodatke koje dodaje vaš fotoaparat — na primer, vreme snimanja i, ako je uključeno na vašem uređaju, lokaciju. Trenutno ne uklanjamo te metapodatke iz otpremljenih slika, pa proverite podešavanja kamere na svom uređaju ako ne želite da budu uključeni u fotografije koje objavljujete.
- **Status oglasa** — da li je oglas aktivan, zabranjen itd.
- **Brojači** — broj pregleda i broj dodavanja u omiljeno za svaki oglas. Sačuvani brojači su anonimne ukupne vrednosti. Da bismo svaki pregled računali samo jednom, čuvamo kratkotrajan tehnički zapis koji povezuje oglas sa identifikatorom vašeg uređaja ili IP adresom najduže 12 sati, nakon čega se automatski briše; ne pravimo istoriju oglasa koje ste pregledali. Odvojeno od toga, ako ste pristali na analitiku, vaša aktivnost pregledanja deo je analitike opisane u odeljku 2.14.
- **Broj izmena** — koliko ste puta izmenili oglas.

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b)) — oglasi su osnovna funkcionalnost platforme.

### 2.5 Poruke i prilozi u ćaskanju

Platforma uključuje funkciju privatnog razmenjivanja poruka između korisnika. Čuvamo:

- **Poruke** — pošiljalac, primalac, sadržaj poruke, vremenska oznaka, status pročitanosti. Poruke se čuvaju na Google Firestore (vidi odeljak 4).
- **Prilozi sa slikama** — porukama možete priložiti slike. One se čuvaju na Cloudflare R2 i dostupne su samo putem potpisanih, vremenski ograničenih veza.
- **Opciona referenca na proizvod** — kada započnete razgovor sa drugim korisnikom klikom na „Pošalji poruku” na stranici oglasa, prva poruka koju pošaljete u tom razgovoru sadrži referencu na taj oglas.
- **Obaveštenja** — obaveštavamo vas o relevantnoj aktivnosti, a sadržaj obaveštenja zavisi od događaja:
  - **Neko doda vaš oglas u omiljeno** — čuvamo obaveštenje u aplikaciji na Firestore-u i šaljemo push obaveštenje. Ono navodi oglas, ali **ne** otkriva identitet korisnika koji ga je dodao u omiljeno.
  - **Neko vas zaprati** — čuvamo obaveštenje u aplikaciji i šaljemo push obaveštenje koje sadrži prikazano ime pratioca i vezu ka njegovom profilu.
  - **Administrator preduzme radnju na vašem nalogu ili sadržaju** — čuvamo obaveštenje u aplikaciji i šaljemo push obaveštenje koje opisuje radnju; ono ne sadrži podatke o ličnosti drugih korisnika.
  - **Primite poruku u ćaskanju** — šaljemo **samo push obaveštenje**; ne čuva se zapis obaveštenja u aplikaciji. Da biste poruku mogli da pročitate sa ekrana obaveštenja svog uređaja, što je standard za razmenu poruka, ovo obaveštenje prikazuje prikazano ime pošiljaoca i tekst poruke.

**Pristup porukama.** Administratori platforme mogu pristupiti sadržaju poruka prilikom istrage prijava o nedoličnom ponašanju ili kršenju naših Uslova korišćenja — na primer, prijave za uznemiravanje protiv drugog korisnika. Taj pristup odvija se kroz naše administratorske alate, a ne kroz bilo kakvu opštu vidljivost: izvan poslova moderacije, vaše poruke su vidljive samo vama i drugom učesniku u razgovoru. Za to šta se dešava sa porukama kada se nalog izbriše, vidi odeljak 8.3.

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b)) za čuvanje i isporuku poruka; legitimni interes (član 6. stav 1. tačka f)) za pristup administratora porukama radi moderacije i bezbednosti.

### 2.6 Recenzije

Platforma omogućava kupcima i prodavcima da nakon interakcije ostave recenzije jedni o drugima. Za svaku recenziju čuvamo:

- **Autor recenzije** — korisnik koji je napisao recenziju.
- **Korisnik na koga se odnosi** — korisnik o kome je recenzija.
- **Referenca na proizvod** — ako se recenzija odnosi na određeni oglas.
- **Ocena** — broj od 1 do 5.
- **Komentar** — slobodni tekst. Komentare recenzija automatski prevodimo na ostale podržane jezike platforme (vidi odeljak 2.8).
- **Slike** — opcione fotografije priložene uz recenziju, sačuvane na Cloudflare R2.
- **Status verifikacije** — da li je autor recenzije imao legitimnu interakciju na platformi sa korisnikom na koga se recenzija odnosi, povezanu sa proizvodom.
- **Status odobrenja** — da li je administrator platforme odobrio recenziju za javno prikazivanje. Sve recenzije zahtevaju odobrenje administratora pre nego što postanu vidljive drugim korisnicima.

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b)) — recenzije su deo sistema poverenja platforme.

### 2.7 Omiljeno i praćenje

Kada sačuvate oglas u omiljeno, čuvamo zapis koji povezuje vaš nalog sa tim oglasom. Vaša lista omiljenih oglasa vidljiva je samo vama. Vlasnik oglasa dobija obaveštenje da je oglas dodat u omiljeno, ali to obaveštenje ne otkriva vaš identitet (vidi odeljak 2.5).

**Praćenje.** Možete pratiti druge korisnike. Kada nekoga zapratite, čuvamo zapis koji povezuje vaš nalog sa njegovim, a praćeni korisnik dobija obaveštenje koje sadrži vaše prikazano ime (vidi odeljak 2.5). Lista naloga koje pratite vidljiva je samo vama; drugim korisnicima ne prikazujemo vaše pratioce niti koga pratite. Zapisi o praćenju brišu se kada se bilo koji od dva naloga trajno izbriše.

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b)).

### 2.8 Automatsko prevođenje vašeg sadržaja

Platforma podržava četiri jezika: srpski, engleski, ruski i crnogorski. Kada napišete naslov oglasa, opis oglasa, biografiju profila ili komentar recenzije na jednom jeziku, vaš tekst šaljemo kompaniji **OpenAI** (eksterna usluga sa sedištem u Sjedinjenim Američkim Državama) radi automatskog prevođenja na ostale podržane jezike platforme. Prevedene verzije čuvaju se uz vaš originalni tekst kako bi korisnici na platformi mogli da vide vaš sadržaj na željenom jeziku.

Ovi zahtevi sadrže samo tekst koji se prevodi, uokviren našim uputstvima za prevođenje — ne uključuju vaše ime, imejl adresu, identifikator naloga niti bilo koje druge metapodatke o vama, a od OpenAI-ja tražimo da ne čuva sadržaj zahteva. Poruke iz ćaskanja se nikada ne šalju OpenAI-ju.

**AI predlog opisa.** Prilikom kreiranja oglasa možete opciono zatražiti da platforma predloži opis. Ako koristite ovu funkciju, naziv oglasa koji ste ukucali šalje se OpenAI-ju radi izrade nacrta, pod istim uslovima opisanim iznad.

Uslovi OpenAI API-ja obavezuju da se podaci poslati putem njihovog API-ja ne koriste za obučavanje njihovih modela. Sam Oglasino ne obučava nijedan AI model na vašem sadržaju (vidi odeljak 6 za naše šire obaveze u vezi sa korišćenjem podataka).

Ova obrada podrazumeva prenos vašeg teksta van Evropskog ekonomskog prostora u Sjedinjene Američke Države. Vidi odeljak 5 o međunarodnim prenosima.

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b)) — višejezični sadržaj je osnovna funkcija platforme.

### 2.9 Prijave

Svaki korisnik može prijaviti drugog korisnika, oglas ili recenziju zbog kršenja naših Uslova korišćenja. Kada se podnese prijava, čuvamo:

- **Ko je podneo prijavu** — nalog prijavioca.
- **Ko ili šta se prijavljuje** — ciljni korisnik, oglas ili recenzija.
- **Kategorija prijave** — jedna od: prevara, loša usluga, kršenje pravila, nasilje ili uznemiravanje, neprimeren sadržaj, obmanjujući ili lažan oglas, kršenje pravila portala, tehnički problem, drugo.
- **Opis** — slobodni tekst od prijavioca.
- **Status rešavanja** — da li je administrator pregledao prijavu i eventualne napomene o rešenju.

Trenutno prijave ručno pregleda administrator. U budućnosti planiramo da uvedemo automatsku obradu određenih kategorija prijava; ova Politika privatnosti biće ažurirana kada se ta promena sprovede. Za to koliko dugo se prijave čuvaju, vidi odeljak 8.6.

**Pravni osnov:** legitimni interes (član 6. stav 1. tačka f)) — bezbednost platforme, sprečavanje zloupotreba i sprovođenje pravila.

### 2.10 Podaci o moderaciji i bezbednosti platforme

Radi održavanja bezbednosti platforme, čuvamo sledeće podatke o svakom korisniku:

- **Status moderacije** vaših oglasa — da li je svaki od vaših oglasa odobren ili zabranjen.
- **Broj kazni** izrečenih protiv vašeg naloga.
- **Status onemogućenosti** — da li je administrator onemogućio vaš nalog.
- **Interni status verifikacije** — da li je vaš nalog interno verifikovan.

**Pravni osnov:** legitimni interes (član 6. stav 1. tačka f)) — bezbednost platforme i sprovođenje pravila.

### 2.11 Preferencije komunikacije

Vaš nalog čuva vaše preferencije komunikacije, koje možete promeniti u svako doba u podešavanjima:

- Da li dozvoljavate transakcione imejlove o vašem nalogu.
- Da li dozvoljavate promotivne ili marketinške imejlove.
- Da li dozvoljavate drugim korisnicima da pozovu vaš broj telefona.

Izbori u vezi sa kolačićima i analitikom ne čuvaju se kao podešavanja naloga: na veb-sajtu ih pravite putem banera za pristanak na kolačiće (sačuvanog u vašem pregledaču), a u mobilnoj aplikaciji putem podešavanja analitike u aplikaciji (sačuvanog na vašem uređaju). Vidi odeljak 7. Obaveštenjima upravljaju dozvole za obaveštenja vašeg uređaja ili pregledača, a ne podešavanje naloga (vidi odeljke 2.5 i 2.15): isključivanje obaveštenja u vašem operativnom sistemu ili pregledaču ih zaustavlja, a odjavljivanje odvaja vaš uređaj od push obaveštenja.

Imajte u vidu da, čak i ako isključite „transakcione imejlove”, i dalje vam možemo slati imejlove koji su neophodni za vaše korišćenje platforme (na primer, potvrdu da je vaš nalog zakazan za brisanje). Zaista opcioni i promotivni imejlovi uslovljeni su prekidačem za „promotivne imejlove”.

**Pravni osnov:** pristanak (član 6. stav 1. tačka a)) za svaku preferenciju. Pristanak možete opozvati u svako doba promenom prekidača u podešavanjima; opoziv ne utiče na zakonitost obrade pre opoziva.

[LAWYER REVIEW: Emails that are essential to the service (for example, the deletion confirmation) are sent regardless of the "transactional emails" toggle. Consider whether contract (Art. 6(1)(b)) or legitimate interest is the accurate basis for essential service emails, with consent governing only the optional categories.]

### 2.12 Tehnički podaci

Da bismo održavali platformu funkcionalnom, bezbednom i otpornom na zloupotrebe, obrađujemo:

- **IP adresa** — u dva kratkotrajna tehnička zapisa: zapisu za ograničavanje učestalosti zahteva u našem Redis kešu, koji se čuva približno 30 minuta, i zapisu za sprečavanje dvostrukog brojanja pregleda opisanom u odeljku 2.4, koji se čuva najduže 12 sati. Zapisi o IP adresi vezuju se za vaš nalog kada ste prijavljeni.
- **Aplikacioni dnevnici (logovi)** — operativni dnevnici aktivnosti na platformi, koji se koriste za otklanjanje grešaka, bezbednosni nadzor i reagovanje na incidente. Svaki zabeleženi zahtev uključuje tehničke metapodatke kao što su vaša IP adresa i, kada ste prijavljeni, identifikator vašeg naloga. Za čuvanje vidi odeljak 8.2.
- **Kolačići** — vidi odeljak 7.

**Pravni osnov:** legitimni interes (član 6. stav 1. tačka f)) — bezbednost, sprečavanje zloupotreba i pouzdanost platforme.

### 2.13 Zapisi o brisanju naloga

Kada izbrišete svoj nalog, nakon brisanja zadržavamo ograničen revizorski zapis. Za sve detalje vidi odeljak 8 (Koliko dugo čuvamo vaše podatke).

### 2.14 Podaci iz analitike

Da bismo razumeli kako se Oglasino koristi i da bismo ga unapredili, prikupljamo analitičke podatke pomoću usluge Google Analytics 4 (GA4) i na veb-sajtu i u mobilnoj aplikaciji — ali **samo ako pristanete**. Kada je uključena, GA4 prikuplja pseudonimne informacije o korišćenju, kao što su pregledane stranice ili ekrani, korišćene pretrage i filteri, pregledani oglasi i slični događaji interakcije, zajedno sa pseudonimnim analitičkim identifikatorom (na vebu, identifikator iz analitičkog kolačića; u mobilnoj aplikaciji, Firebase identifikator instance aplikacije). Ako ste prijavljeni, ove događaje povezujemo sa korisničkim identifikatorom vašeg naloga kako bi se korišćenje moglo razumeti kroz vaše sesije. Naša analitika je isključivo prvostrana: ne koristimo je za oglašavanje, ciljanje reklama niti praćenje preko više sajtova ili aplikacija, a Google-ovi signali za oglašavanje su onemogućeni. Način na koji dajete ili uskraćujete pristanak razlikuje se po platformi: na veb-sajtu putem banera za kolačiće; u mobilnoj aplikaciji putem izbora na nivou uređaja koji pravite pri prvom otvaranju aplikacije i koji možete promeniti u svako doba u podešavanjima aplikacije (za oba vidi odeljak 7). Analitika je podrazumevano isključena; ne uključujemo prikupljanje analitike osim ako ste je dozvolili. Analitički podaci se u usluzi Google Analytics čuvaju [2/14] meseci, nakon čega se automatski brišu.

**Pravni osnov:** pristanak (član 6. stav 1. tačka a)). Pristanak možete opozvati u svako doba — na veb-sajtu putem banera za kolačiće ili stranice sa preferencijama kolačića, a u mobilnoj aplikaciji putem podešavanja analitike u aplikaciji; opoziv ne utiče na zakonitost obrade pre opoziva.

### 2.15 Push tokeni za obaveštenja

Ako dozvolite obaveštenja — u Oglasino mobilnoj aplikaciji ili u svom veb-pregledaču na našem veb-sajtu — vašem uređaju ili pregledaču izdaje se **push token**: identifikator koji nam omogućava da šaljemo push obaveštenja baš tom uređaju ili pregledaču. Vaš push token čuvamo u našoj bazi podataka i povezujemo ga sa vašim nalogom kako bismo mogli da vam isporučujemo obaveštenja (na primer, upozorenja o novim porukama; vidi odeljak 2.5). U mobilnoj aplikaciji, vaš push token se odvaja od vašeg naloga kada se odjavite, a zamenjuje se ili uklanja kako se vaš uređaj ili njegove dozvole menjaju. Svi sačuvani push tokeni brišu se kada se vaš nalog trajno izbriše. Push obaveštenja isporučuju se preko provajdera navedenih u odeljku 4: u mobilnoj aplikaciji preko usluge Expo Push Service (sa osnovnim push uslugama kompanija Apple i Google), a u pregledaču preko usluge Firebase Cloud Messaging.

**Pravni osnov:** izvršenje ugovora (član 6. stav 1. tačka b)) — isporuka obaveštenja deo je usluge razmene poruka i naloga na koju se prijavljujete. Push obaveštenja možete zaustaviti u svako doba tako što ćete ih isključiti u podešavanjima svog uređaja ili pregledača, ili odjavljivanjem.

### 2.16 Korišćenje mobilne aplikacije — dozvole uređaja i podaci sa uređaja

Naše iOS i Android aplikacije traže vašu dozvolu za korišćenje određenih funkcija uređaja, i to samo kada su funkciji potrebne:

- **Kamera i galerija fotografija** — kada snimite ili izaberete fotografiju za dodavanje u oglas, profilnu sliku, poruku ili recenziju. Aplikacija pristupa kameri ili galeriji samo u trenutku kada odlučite da dodate fotografiju; sa fotografijama koje potom izaberete postupa se kako je opisano u odeljcima 2.2, 2.4, 2.5. i 2.6.
- **Obaveštenja** — da bismo vam slali push obaveštenja. Rezultat je push token za vaš uređaj (vidi odeljak 2.15).

Ove dozvole možete dati ili opozvati u svako doba u podešavanjima svog uređaja.

Da budemo jasni šta mobilna aplikacija **ne** radi: **ne** pristupa lokaciji vašeg uređaja (GPS) — region i grad koji se koriste na platformi su vrednosti na nivou naloga koje sami birate (vidi odeljak 2.3), a ne lokacija vašeg uređaja; **ne** čita vaše kontakte; **ne** prikuplja identifikator za oglašavanje (kao što su Apple-ov IDFA ili Android-ov identifikator za oglašavanje); **ne** koristi Apple-ov sistem App Tracking Transparency; i **ne** prati vas kroz druge aplikacije ili veb-sajtove.

<!-- RESOLVED IN CODE 2026-06-10 — see §2.14 expo note. One nuance for the Play Console data-safety form: the merged Android manifest still *declares* ACCESS_ADSERVICES_AD_ID / ACCESS_ADSERVICES_ATTRIBUTION (transitive from play-services-measurement) while collection is disabled; this sentence about collection stays true, but answer the store form as declared-but-not-collected. -->

Aplikacija takođe čuva određene informacije na vašem uređaju; vidi odeljak 7.

---

## 3. Kako koristimo vaše podatke — pregled

Da bi odeljak 2 bio lakši za snalaženje, evo pregleda svih svrha u koje koristimo vaše podatke i pravnog osnova za svaku. Detaljna obaveštenja nalaze se u odeljku 2.

| Svrha                                                                                    | Pravni osnov      |
| ---------------------------------------------------------------------------------------- | ----------------- |
| Kreiranje i vođenje vašeg naloga                                                         | Ugovor            |
| Prikazivanje vašeg profila drugim korisnicima                                            | Ugovor            |
| Objavljivanje vaših oglasa drugim korisnicima                                            | Ugovor            |
| Isporuka poruka između korisnika                                                         | Ugovor            |
| Prikazivanje recenzija o vama i recenzija koje ste vi napisali                           | Ugovor            |
| Čuvanje vaših omiljenih oglasa i naloga koje pratite                                     | Ugovor            |
| Automatsko prevođenje vašeg sadržaja na ostale podržane jezike                           | Ugovor            |
| Slanje komunikacija na koje ste pristali                                                 | Pristanak         |
| Postavljanje kolačića koji nisu neophodni u vašem pregledaču                             | Pristanak         |
| Merenje i unapređenje platforme pomoću analitike (samo ako pristanete)                   | Pristanak         |
| Isporuka push obaveštenja na vaš uređaj ili u pregledač                                  | Ugovor            |
| Sprečavanje prevara, zloupotreba i neželjenog sadržaja (spama)                           | Legitimni interes |
| Moderacija sadržaja i sprovođenje pravila platforme                                      | Legitimni interes |
| Održavanje bezbednosti i pouzdanosti platforme                                           | Legitimni interes |
| Istraga prijava koje podnose korisnici                                                   | Legitimni interes |
| Čuvanje heširanog revizorskog zapisa o brisanjima naloga i zabranama                     | Legitimni interes |
| Ispunjavanje zakonskih obaveza (na primer, čuvanje podataka po zahtevu nadležnih organa) | Zakonska obaveza  |

---

## 4. Treće strane koje obrađuju vaše podatke

Za rad Oglasina koristimo više eksternih pružalaca usluga. Svaki od njih obrađuje deo vaših podataka u naše ime. Vaše podatke nikada nikome ne prodajemo (vidi odeljak 6 o našim izričitim obavezama).

| Provajder                                                                                                  | Uloga                                                                                                    | Šta obrađuje                                                                                                                                                                                                                                                                                                                                                                                    | Lokacija                                                                                            |
| ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Google Firebase Authentication**                                                                         | Provajder autentifikacije                                                                                | Vaš imejl, lozinka (drži je samo Firebase), tokeni za Google prijavljivanje                                                                                                                                                                                                                                                                                                                     | Globalna Google infrastruktura; obrada može uključivati Sjedinjene Američke Države (vidi odeljak 5) |
| **Google Firestore**                                                                                       | Baza podataka u realnom vremenu za poruke i obaveštenja                                                  | Sadržaj poruka, obaveštenja, prikazana imena pošiljalaca                                                                                                                                                                                                                                                                                                                                        | Frankfurt, Nemačka (region europe-west3)                                                            |
| **Cloudflare R2**                                                                                          | Skladište objekata za slike                                                                              | Profilne slike, fotografije oglasa, prilozi iz ćaskanja, fotografije uz recenzije                                                                                                                                                                                                                                                                                                               | Istočna Evropa (EEUR)                                                                               |
| **Cloudflare**                                                                                             | Mrežna ivica, mreža za isporuku sadržaja (CDN) i usluga pristupa slikama                                 | Sav saobraćaj ka našem veb-sajtu i API-ju prolazi kroz Cloudflare-ovu ivičnu mrežu, koja prekida šifrovane veze i usmerava zahteve ka našem hostingu; Cloudflare kešira statički sadržaj i slike i izvršava uslugu koja proverava potpisane tokene pristupa kada otpremate slike ili pregledate privatne slike, pri čemu u tehničkim dnevnicima obrađuje identifikator vašeg naloga i IP adresu | Globalna ivična mreža                                                                               |
| **OpenAI**                                                                                                 | Automatsko prevođenje sadržaja koji pišu korisnici; opcioni predlozi opisa oglasa                        | Naslovi i opisi oglasa, biografije profila, komentari recenzija (samo tekst — vidi odeljak 2.8)                                                                                                                                                                                                                                                                                                 | Sjedinjene Američke Države                                                                          |
| **DigitalOcean**                                                                                           | Hosting bekend aplikacije (uključujući naše samostalno hostovane usluge keširanja i pretrage)            | Svi podaci koje obrađuju naši bekend servisi                                                                                                                                                                                                                                                                                                                                                    | Frankfurt, Nemačka (region fra1)                                                                    |
| **Vercel**                                                                                                 | Hosting veb frontenda (uključujući renderovanje na serverskoj strani)                                    | Zahtevi za stranice, kolačići sesije, tela zahteva                                                                                                                                                                                                                                                                                                                                              | Frankfurt, Nemačka (region fra1)                                                                    |
| **Google reCAPTCHA**                                                                                       | Detekcija botova na formularima                                                                          | IP adresa, informacije o pregledaču, podaci o mišu i vremenu, kolačići koje postavlja Google                                                                                                                                                                                                                                                                                                    | Sjedinjene Američke Države                                                                          |
| **Google Analytics 4 (Google LLC)**                                                                        | Analitika korišćenja veb-sajta i mobilne aplikacije                                                      | Pseudonimni događaji korišćenja i analitički identifikatori (kolačić `_ga` na vebu; Firebase identifikator instance aplikacije u mobilnoj aplikaciji); prikupljaju se samo ako pristanete na analitiku                                                                                                                                                                                          | Sjedinjene Američke Države                                                                          |
| **Brevo**                                                                                                  | Slanje imejlova naloga i platforme (npr. verifikacija imejla, resetovanje lozinke, obaveštenja o nalogu) | Vaša imejl adresa i sadržaj tih imejlova                                                                                                                                                                                                                                                                                                                                                        | Evropska unija                                                                                      |
| **Expo Push Service, sa uslugama Apple Push Notification service (APNs) i Firebase Cloud Messaging (FCM)** | Isporuka push obaveštenja (mobilna aplikacija preko Expo-a sa APNs/FCM; veb-pregledač preko FCM-a)       | Push token vašeg uređaja ili pregledača i sadržaj obaveštenja (uključujući, za obaveštenja o porukama u ćaskanju, prikazano ime pošiljaoca i tekst poruke)                                                                                                                                                                                                                                      | Sjedinjene Američke Države (Expo; Apple; Google)                                                    |

**Otkrivanje organima vlasti.** Pored gore navedenih obrađivača, podatke o ličnosti možemo otkriti sudovima, organima za sprovođenje zakona ili drugim organima javne vlasti kada primimo pravno obavezujući zahtev, kao i profesionalnim savetnicima (kao što su advokati) kada je to neophodno radi utvrđivanja, ostvarivanja ili odbrane pravnih zahteva. [LAWYER REVIEW: confirm the scope and wording of this authority-disclosure clause.]

---

## 5. Međunarodni prenosi

Najveći deo naše obrade odvija se unutar Evropskog ekonomskog prostora (EEP), koji ima standarde zaštite podataka koje zahteva GDPR.

Nekoliko naših obrađivača ima sedište u Sjedinjenim Američkim Državama ili može obrađivati podatke u njima: **OpenAI** (za prevođenje sadržaja), **Google reCAPTCHA** (za detekciju botova), **Google Analytics** (samo ako pristanete), **Google Firebase Authentication** (koji radi na globalnoj Google infrastrukturi) i provajderi koji isporučuju push obaveštenja (**Expo** i osnovne push usluge kompanija **Apple** i **Google**, uključujući Firebase Cloud Messaging za pregledač). Kada se vaši podaci pošalju ovim obrađivačima, mogu napustiti EEP.

Za ove prenose oslanjamo se na sledeće mere zaštite:

- **OpenAI** — podatke prenosimo na osnovu Standardnih ugovornih klauzula Evropske komisije i pristupanja OpenAI-ja okviru EU-US Data Privacy Framework (gde je primenljivo). Rukovalac (Oglasino) koristi OpenAI API usluge na osnovu ugovora OpenAI Services Agreement. Dodatak o obradi podataka (OpenAI Data Processing Addendum) ugrađen je u taj ugovor i primenjuje se na korišćenje usluga.
- **Google reCAPTCHA** — oslanjamo se na pristupanje Google-a okviru EU-US Data Privacy Framework i Google-ove Standardne ugovorne klauzule.
- **Google Analytics** — za korisnike koji pristanu na analitiku, oslanjamo se na pristupanje Google-a okviru EU-US Data Privacy Framework i Google-ove Standardne ugovorne klauzule.
- **Firebase Authentication** — Google upravlja uslugom Firebase Authentication na globalnoj infrastrukturi, a obrada može uključivati Sjedinjene Američke Države. Oslanjamo se na pristupanje Google-a okviru EU-US Data Privacy Framework i Standardne ugovorne klauzule ugrađene u Google-ove uslove obrade podataka za Firebase/Google Cloud.
- **Isporuka push obaveštenja** — push obaveštenja se usmeravaju preko Expo-a (mobilna aplikacija) i usluge Firebase Cloud Messaging (pregledač), uz Apple-ove (APNs) i Google-ove (FCM) usluge u osnovi. Za Expo se oslanjamo na ugovor o obradi podataka sa Expo-om koji uključuje Standardne ugovorne klauzule Evropske komisije; za Apple i Google oslanjamo se na njihovo pristupanje okviru EU-US Data Privacy Framework i Standardne ugovorne klauzule. [LAWYER REVIEW: confirm that a DPA/SCCs with Expo is actually in force before publication — currently unverified (open item, 2026-06-10).]

Više informacija o ovim merama zaštite možete zatražiti kontaktiranjem na privacy@oglasino.com.

---

## 6. Šta nikada nećemo raditi sa vašim podacima

Pored obaveza koje prema vama imamo po GDPR-u, dajemo dve izričite obaveze u vezi sa vašim podacima:

**Ne prodajemo vaše podatke.** Nikada nismo i nikada nećemo. Ne prodajemo korisničke podatke oglašivačima, posrednicima u trgovini podacima niti bilo kojoj trećoj strani. Podatke delimo sa obrađivačima samo kako je opisano u odeljku 4, i samo u meri neophodnoj za rad platforme.

**Ne obučavamo veštačku inteligenciju na vašim podacima.** Oglasino ne koristi vaš sadržaj — vaše oglase, vaše poruke, vaše recenzije, vaš profil — za obučavanje bilo kog modela veštačke inteligencije ili mašinskog učenja. Ova obaveza se odnosi konkretno na to šta mi radimo sa vašim podacima. Odvojeno od toga, naš provajder prevođenja OpenAI se u svojim API uslovima obavezuje da se podaci poslati putem njihovog API-ja ne koriste za obučavanje njihovih modela; to je obaveza koju OpenAI daje o sopstvenom korišćenju podataka koje im šaljemo.

Ako ikada izmenimo bilo koju od ove dve obaveze, jasno ćemo vam to saopštiti pre nego što promena stupi na snagu i daćemo vam mogućnost da izbrišete svoj nalog pre nego što započne bilo kakva nova obrada.

---

## 7. Kolačići i slične tehnologije

Veb-sajt Oglasino koristi kolačiće i slične mehanizme skladištenja u pregledaču (kao što su localStorage i IndexedDB); naše mobilne aplikacije umesto kolačića koriste slično skladištenje na uređaju. Ovaj odeljak pokriva oboje — prvo veb-sajt, zatim mobilnu aplikaciju. Na veb-sajtu kolačiće i slično skladištenje koristimo u tri svrhe:

**Strogo neophodni** — potrebni da bi veb-sajt funkcionisao. Tu spadaju vaša sesija autentifikacije (kojom upravlja Firebase Authentication, sačuvana u localStorage i IndexedDB), bezbednosni tokeni (na primer, CSRF zaštita), samo stanje pristanka na kolačiće i mali broj sličnih tehničkih kolačića.

**Preferencije** — koriste se za pamćenje vaših izbora, kao što su željeni jezik, izabrani državni portal (Srbija ili Crna Gora) i slična podešavanja.

**Analitika** — koristi se za merenje načina na koji se veb-sajt koristi kako bismo ga razumeli i unapredili. Koristimo Google Analytics 4 (GA4). Analitički kolačići (na primer, `_ga` i `_gid`) i prikupljanje analitike postavljaju se **samo ako pristanete** na kategoriju analitike; ako odbijete, ne postavlja se nijedan analitički kolačić i ne prikupljaju se nikakvi analitički podaci. Koristimo Google Consent Mode v2: od četiri Google-ova signala pristanka, vašim izborom može se uključiti samo skladištenje za analitiku — tri signala povezana sa oglašavanjem (`ad_storage`, `ad_user_data`, `ad_personalization`) trajno su odbijena, jer ne koristimo Google-ove funkcije oglašavanja. Naša analitika je isključivo prvostrana; ne koristimo je za oglašavanje, ciljanje reklama niti praćenje preko više sajtova.

<!-- RESOLVED IN CODE 2026-06-10 (oglasino-web, basic Consent Mode) — see §2.14 note; pending Igor's browser checklist. -->

**Trajanje kolačića.** Kolačić pristanka `og_consent` čuva se 12 meseci. Google-ovi analitički kolačići prate Google-ove podrazumevane vrednosti (`_ga` do 2 godine, `_gid` oko 24 sata) i postavljaju se tek nakon što pristanete. Kolačić `_GRECAPTCHA` postavlja i njime upravlja Google.

Ova lista kolačića može se vremenom ažurirati kako se platforma razvija; ovaj odeljak ćemo održavati ažurnim.

**Pristanak na kolačiće.** Kada prvi put posetite Oglasino, baner za pristanak vam omogućava da izaberete **Prihvati sve**, **Odbij sve** ili **Prilagodi** svoje izbore po kategorijama. Strogo neophodni kolačići ne mogu se odbiti jer sajt bez njih ne može da funkcioniše; kategorije preferencija i analitike su opcione i ostaju isključene dok ih ne dozvolite. Vaš izbor se beleži u malom kolačiću sačuvanom u vašem pregledaču (pod nazivom `og_consent`) — pošto se čuva u vašem pregledaču, a ne na vašem nalogu, izbor važi za taj pregledač. Svoje preferencije kolačića možete promeniti u svako doba na našoj stranici sa preferencijama kolačića na `/owner/cookies` (povezanoj sa stranice podešavanja vašeg naloga), ili ponovnim otvaranjem banera za pristanak pomoću kontrole „Upravljanje kolačićima” u podnožju veb-sajta.

**Skripte trećih strana.** Jedna skripta treće strane na našem sajtu zahteva posebno obaveštenje: **Google reCAPTCHA**. reCAPTCHA se učitava na stranicama sa formularima — registracija, kreiranje oglasa, slanje recenzija, predlozi kategorija i formulari za prijave — radi sprečavanja automatizovanih zloupotreba. Postavlja sopstvene kolačiće (uključujući kolačić `_GRECAPTCHA`) i prikuplja informacije uključujući vašu IP adresu, okruženje pregledača, pokrete miša i sadržaj eventualnih Google kolačića sesije koje već imate. Na obradu reCAPTCHA oslanjamo se po osnovu legitimnog interesa u sprečavanju prevara. Za više informacija pogledajte Google-ova obaveštenja o privatnosti na https://policies.google.com/privacy.

**Mobilna aplikacija — skladištenje na uređaju.** Mobilne aplikacije ne koriste kolačiće. Umesto toga, čuvaju informacije na vašem uređaju kako bi aplikacija radila, uključujući: vašu sesiju prijavljivanja (koju drži Firebase Authentication); izabrani portal, jezik, temu i preferencije prikaza; vaš izbor u vezi sa analitikom (vidi ispod); keširane prevode i podatke kataloga; vremenske oznake koje se koriste da se izbegne ponovno traženje dozvola ili ponovno prikazivanje istog obaveštenja; i keširanu kopiju slika koje je aplikacija prikazala. Najveći deo toga su funkcionalni podaci ili podaci o preferencijama. Vaša sesija prijavljivanja i sačuvana kopija profila vašeg naloga brišu se kada se odjavite; mali broj funkcionalnih stavki koje ne identifikuju ličnost — na primer, oznaka da je vaš nalog već podešen na uređaju i keš slika aplikacije — može ostati na uređaju nakon odjavljivanja dok ih vaš operativni sistem ne oslobodi. Sve možete obrisati odjavljivanjem i brisanjem aplikacije, ili putem kontrola za skladištenje aplikacija na svom uređaju.

**Mobilna aplikacija — pristanak na analitiku.** Analitika u mobilnoj aplikaciji radi drugačije nego na veb-sajtu: nema banera za kolačiće i nema kolačića. Umesto toga, kada prvi put otvorite aplikaciju, pitamo vas da li da dozvolimo analitiku, a analitika je **podrazumevano isključena** — ne uključujemo prikupljanje analitike osim ako je dozvolite. Svoj izbor možete promeniti u svako doba u podešavanjima aplikacije. Analitika mobilne aplikacije je ista prvostrana GA4 opisana iznad, koja izveštava u isto analitičko svojstvo koristeći Firebase identifikator instance aplikacije; **ne** koristi Apple-ov sistem App Tracking Transparency i **ne** prikuplja identifikator za oglašavanje.

<!-- RESOLVED IN CODE 2026-06-10 (oglasino-expo) — see §2.14 note; pending EAS on-device smoke. -->

---

## 8. Koliko dugo čuvamo vaše podatke

Podrazumevano pravilo je jednostavno: dok je vaš nalog aktivan, vaše podatke čuvamo onoliko dugo koliko je potrebno za pružanje usluge. Kada izbrišete svoj nalog, brišemo vaše podatke — uz nekoliko konkretnih, vremenski ograničenih izuzetaka opisanih u nastavku.

### 8.1 Dok je vaš nalog aktivan

Podaci o vašem nalogu, podaci o profilu, oglasi, slike oglasa, poruke, prilozi iz ćaskanja, recenzije, omiljeni oglasi i zapisi o praćenju čuvaju se dok je vaš nalog aktivan. Oglasi trenutno ne ističu automatski; pojedinačne oglase možete izbrisati u svako doba sa svog naloga.

U budućnosti planiramo da uvedemo automatsko brisanje oglasa i poruka iz ćaskanja koji su duže vreme neaktivni. Kada to uvedemo, ažuriraćemo ovu Politiku privatnosti.

### 8.2 IP adrese i operativni dnevnici

- **IP adrese u kratkotrajnim tehničkim zapisima** čuvaju se približno 30 minuta (ograničavanje učestalosti zahteva) i najduže 12 sati (sprečavanje dvostrukog brojanja pregleda) — vidi odeljke 2.4 i 2.12.
- **Aplikacioni dnevnici**, koji uključuju IP adrese i identifikatore naloga (vidi odeljak 2.12), čuvaju se **90 dana**, nakon čega se trajno uklanjaju.
<!-- RESOLVED IN REPO 2026-06-10 (oglasino-backend infra): json-file size caps on all services + infra/prune-container-logs.sh (daily cron deletes the backend container's rotated segments older than 88 days) + infra/LOG-RETENTION.md. The 90-day claim goes live only after Igor's host apply: ship files, `up -d --force-recreate` affected containers, install the root cron. Do not publish before the host apply. -->

### 8.3 Kada izbrišete svoj nalog

Brisanje naloga na Oglasinu je dvostepeni proces osmišljen da vam pruži smislen period za predomišljanje i, podjednako važno, da drugim korisnicima pruži poštenu poslednju priliku da vas prijave ako smatraju da ste na platformi postupali u zloj veri.

**Faza 1 — Privremeno brisanje (7 dana).** Kada iz podešavanja zatražite brisanje naloga, vaš nalog odmah ulazi u stanje „zakazano za brisanje”:

- **Vaš profil ostaje javno vidljiv** i označen je statusnom oznakom „Zakazano za brisanje” kako bi drugi korisnici mogli da vide da ste zatražili brisanje. Namerno ostavljamo vaš profil vidljivim tokom ovog perioda, umesto da ga sakrijemo, kako bi korisnici koji su imali interakciju sa vama i dalje imali priliku da podnesu prijavu ako imaju opravdanu zabrinutost u vezi sa vašim ponašanjem.
- **Vaš broj telefona je sakriven** tokom ovog perioda, kako vas niko ne bi kontaktirao dok je vaše brisanje u toku.
- **Vaši oglasi su sakriveni** od javnosti.
- **Recenzije o vama ostaju vidljive** kako bi kontekst za eventualne prijave bio očuvan.
- **Ne možete slati ni primati poruke** tokom perioda počeka. Postojeći razgovori ostaju dostupni drugim učesnicima radi prijavljivanja, ali se sa vašeg naloga i ka njemu ne mogu slati nove poruke.
- **Prijave protiv vas ostaju otvorene.** Svaki korisnik koji je imao interakciju sa vama i dalje može podneti prijavu tokom perioda počeka.
- Bićete odjavljeni.

Tokom ovih 7 dana, svoj nalog možete vratiti u svako doba jednostavnim ponovnim prijavljivanjem. To radimo kako ne biste izgubili svoje podatke zbog slučajnog brisanja, trenutka frustracije ili promene mišljenja. Ako se prijavite tokom perioda počeka, vaš profil, oglasi, vidljivost broja telefona i razmena poruka u potpunosti se vraćaju.

**Faza 2 — Trajno brisanje (nakon 7 dana).** Osmog dana vaš nalog se trajno briše, uz izuzetke odlaganja opisane u odeljku 8.4. Trajno brisanje znači:

- Zapis vašeg naloga, oglasi, slike oglasa, omiljeni oglasi, zapisi o praćenju, sačuvani push tokeni i obaveštenja u aplikaciji trajno se uklanjaju iz naših baza podataka i skladišta.
- Vaša profilna slika briše se iz našeg skladišta.
- Vaš zapis za autentifikaciju kod Firebase-a se briše.
- **Poruke iz ćaskanja se anonimizuju, a ne brišu.** Vaši razgovori su deo istorije poruka i drugog učesnika, pa sadržaj poruka njemu ostaje dostupan — ali vaš identitet se uklanja: reference vašeg naloga u tim razgovorima zamenjuju se anonimnom oznakom, koja se drugim korisnicima prikazuje kao „Obrisani korisnik”. Isto važi i za slike koje ste priložili porukama. [LAWYER REVIEW: confirm that retaining anonymized message content after an erasure request is the intended and defensible position (Art. 17(1) balanced against the other participant's interest in their own correspondence), or whether message content should be deleted as well.]
- **Recenzije koje ste ostavili o drugim korisnicima:** odobrene recenzije se zadržavaju, pri čemu se vaše ime zamenjuje sa „Obrisani korisnik” — one odražavaju iskustvo korisnika koga ste recenzirali, a njihovo brisanje izbrisalo bi deo njegove evidencije. Ovo ovde otkrivamo kako biste, kada ostavljate recenziju, znali da odobrena recenzija nadživljava brisanje vašeg naloga (anonimizovana). Vaše recenzije koje su još čekale odobrenje, ili su odbijene, brišu se. Recenzije o vama se brišu.
- **Zaostale kopije.** Kopije izbrisanog sadržaja mogu se zadržati ograničeno vreme u tehničkim sistemima koji se osvežavaju po sopstvenom rasporedu — na primer, ivični keševi slika i unosi u našem indeksu pretrage — dok ti sistemi ne isteknu ili se ponovo ne sinhronizuju.

Nakon trajnog brisanja, jedini podaci koje zadržavamo su ograničeni revizorski zapisi opisani u odeljku 8.5 i, ako je vaš nalog bio zabranjen, anonimizovani zapisi o prijavama opisani u odeljku 8.6.

### 8.4 Kada brisanje može biti odloženo

U ograničenim okolnostima možemo odložiti trajno brisanje vašeg naloga i nakon standardnog perioda počeka od 7 dana. To se primenjuje samo u dva slučaja:

- **Pravne istrage.** Kada organ za sprovođenje zakona (kao što su policija ili sud) od nas zatraži da sačuvamo vaše podatke u vezi sa istragom u toku, zakonski smo dužni da postupimo po tome i odložićemo brisanje onoliko dugo koliko pravni zahtev nalaže.
- **Interne istrage zloupotreba.** Kada imamo razloga da verujemo da je vaš nalog korišćen za ozbiljnu zloupotrebu platforme — na primer, prevaru prema više korisnika — i aktivno to istražujemo, možemo odložiti brisanje radi očuvanja dokaza relevantnih za istragu.

U oba slučaja, ako je vaše brisanje odloženo, saopštićemo vam da je brisanje odloženo i, u meri u kojoj nam je to zakonski dozvoljeno, zašto. Tokom odlaganja i dalje ćete biti odjavljeni i nećete moći da koristite nalog; jedini efekat je da je trajno brisanje pauzirano. Kada se pravna ili interna istraga okonča, trajno brisanje se sprovodi.

**Pravni osnov za odlaganje:** zakonska obaveza (član 6. stav 1. tačka c)) za zahteve organa za sprovođenje zakona za očuvanje podataka i legitimni interes (član 6. stav 1. tačka f)) za interne istrage zloupotreba.

### 8.5 Revizorski zapisi koji se zadržavaju nakon brisanja ili zabrane

- **Opšti revizorski zapis o brisanju** — heširana verzija vašeg korisničkog identifikatora i imejla (pomoću SHA-256, jednosmernog heša koji se ne može preokrenuti da bi se povratio vaš originalni imejl), zajedno sa datumom brisanja vašeg naloga i razlogom. Čuva se **30 dana** nakon brisanja, a zatim se automatski uklanja.
- **Revizorski zapis o zabranjenom korisniku** — ako je vaš nalog zabranjen, u trenutku izricanja zabrane beležimo heširanu verziju vašeg imejla i razlog zabrane. Ovaj zapis se koristi da spreči zabranjene korisnike da se ponovo registruju istom imejl adresom i čuva se **12 meseci** od zabrane, a zatim se automatski uklanja. Ostaje na snazi i ako se zabranjeni nalog kasnije izbriše.

Ovi zapisi ne sadrže nikakve čitljive podatke o ličnosti — samo jednosmerne hešove. Heš se ne može preokrenuti da bi otkrio vašu imejl adresu, ali se može uporediti sa poznatom imejl adresom (upravo to poređenje je način na koji radi provera ponovne registracije). Nakon isteka perioda čuvanja, briše se i sam heš. [LAWYER REVIEW: confirm the treatment of hashed identifiers as pseudonymized personal data and the adequacy of this disclosure.]

### 8.6 Prijave koje su podneli korisnici ili koje se odnose na njih

Dok su uključeni nalozi aktivni, prijave se čuvaju kako bi administratori mogli da uoče obrasce ponovljenih zloupotreba; trenutno ih čuvamo bez fiksnog vremenskog ograničenja, a planiramo da uvedemo automatsko uklanjanje rešenih prijava nakon određenog perioda. Šta se dešava sa prijavama kada se nalog koji je u njima učestvovao trajno izbriše zavisi od toga da li je taj nalog bio zabranjen:

- Ako izbrisani nalog **nije bio zabranjen**, prijave koje je podneo i koje su podnete o njemu brišu se zajedno sa nalogom.
- Ako je izbrisani nalog **bio zabranjen**, prijave se zadržavaju — uz uklanjanje svih referenci na nalog, povezane samo sa anonimizovanim zapisom o zabrani iz odeljka 8.5 — i brišu se zajedno sa tim zapisom, najkasnije 12 meseci od zabrane.

### 8.7 Ako je vaš nalog zabranjen

Ako trajno zabranimo vaš nalog (vidi Uslove korišćenja, odeljak 10.2), zabrana stupa na snagu odmah: vaš javni profil i broj telefona prestaju da se prikazuju, vaši oglasi se sakrivaju, a vaše prijavljivanje se onemogućava. Sama zabrana ne pokreće postupak brisanja — podatke naloga zadržavamo onoliko dugo koliko je potrebno da dokumentujemo prekršaj, obradimo eventualnu žalbu, zaštitimo druge korisnike i utvrdimo, ostvarimo ili odbranimo pravne zahteve. Brisanje zabranjenog naloga možete zatražiti u svako doba (vidi odeljak 9), uz slučajeve odlaganja iz odeljka 8.4. Revizorski zapis o zabranjenom korisniku (odeljak 8.5) nastaje u trenutku izricanja zabrane i čuva se 12 meseci.

**Pravni osnov:** legitimni interes (član 6. stav 1. tačka f)) — bezbednost platforme i utvrđivanje, ostvarivanje ili odbrana pravnih zahteva. [LAWYER REVIEW: confirm the basis and proportionality of retaining banned-account data, and whether a fixed maximum retention period should be stated.]

---

## 9. Vaša prava

Na osnovu GDPR-a i srpskih i crnogorskih propisa o zaštiti podataka o ličnosti, imate sledeća prava u vezi sa svojim podacima o ličnosti. Podržavamo svako od njih.

**Pravo na pristup.** Možete zatražiti kopiju podataka o ličnosti koje čuvamo o vama. Pošaljite zahtev imejlom na privacy@oglasino.com. Odgovorićemo u roku od mesec dana, kako to zahteva GDPR.

**Pravo na ispravku.** Većinu svojih podataka o ličnosti možete ispraviti direktno u podešavanjima profila. Za podatke koje ne možete sami izmeniti (kao što su informacije u vezi sa moderacijom), pošaljite imejl na privacy@oglasino.com i ispravićemo sve netačnosti.

**Pravo na brisanje („pravo na zaborav”).** Svoj nalog možete izbrisati u svako doba iz podešavanja naloga; za to šta to podrazumeva vidi odeljak 8.3. Ako ne možete da koristite samouslužni tok brisanja — na primer, zato što je vaš nalog onemogućen — kontaktirajte support@oglasino.com da zatražite brisanje i odgovorićemo u roku od mesec dana, kako to zahteva GDPR. Za sva ostala pitanja o brisanju u vezi sa privatnošću pišite na privacy@oglasino.com.

**Pravo na ograničenje obrade.** Možete od nas zatražiti da ograničimo obradu vaših podataka o ličnosti — na primer, dok osporavate odluku o moderaciji. Pošaljite imejl na privacy@oglasino.com sa zahtevom i razlogom.

**Pravo na prenosivost podataka.** Možete zatražiti kopiju podataka o ličnosti koje ste nam pružili, u strukturiranom, uobičajeno korišćenom i mašinski čitljivom formatu (kao što je JSON). Ovo pravo se odnosi na podatke koje ste nam direktno pružili (vaš profil, oglasi, poruke, recenzije); ne obuhvata podatke koje smo o vama izveli (kao što su vaša ocena ili revizorski zapisi). Pošaljite imejl na privacy@oglasino.com da zatražite prenosivi izvoz.

**Pravo na prigovor.** Možete uložiti prigovor na našu obradu vaših podataka kada se oslanjamo na legitimni interes. Na primanje promotivnih imejlova možete uložiti prigovor isključivanjem odgovarajućeg prekidača u podešavanjima i to ćemo uvek poštovati. Za ostale prigovore pošaljite imejl na privacy@oglasino.com i razmotrićemo vaš prigovor.

**Pravo na opoziv pristanka.** Tamo gde se oslanjamo na vaš pristanak (kolačići i analitika, preferencije komunikacije), možete ga opozvati u svako doba:

- Preferencije kolačića i analitike na veb-sajtu: na stranici sa preferencijama kolačića na `/owner/cookies` (povezanoj iz podešavanja vašeg naloga), ili ponovnim otvaranjem banera za pristanak pomoću kontrole „Upravljanje kolačićima” u podnožju veb-sajta.
- Analitika u mobilnoj aplikaciji: putem podešavanja analitike u aplikaciji.
- Preferencije komunikacije: putem prekidača na stranici podešavanja vašeg naloga.

Opoziv pristanka ne utiče na zakonitost obrade koju smo sproveli pre opoziva.

**Pravo na pritužbu nadzornom organu.** Ako smatrate da nismo zakonito postupali sa vašim podacima o ličnosti, imate pravo da se pritužite organu za zaštitu podataka. U Srbiji je nadležni organ Poverenik za informacije od javnog značaja i zaštitu podataka o ličnosti — https://www.poverenik.rs/. U Crnoj Gori je nadležni organ Agencija za zaštitu ličnih podataka i slobodan pristup informacijama — https://www.azlp.me/. Ako se nalazite u EU, možete se pritužiti i organu za zaštitu podataka u svojoj zemlji prebivališta.

**Kako odgovaramo na zahteve.** Nastojimo da na sve zahteve lica na koja se podaci odnose odgovorimo u roku od mesec dana od prijema. U ograničenim slučajevima posebno složenih zahteva, GDPR nam dozvoljava da ovaj rok produžimo za najviše dodatna dva meseca, u kom slučaju ćemo vas o tome obavestiti u toku prvog meseca.

**Automatizovano donošenje odluka.** Oglasino ne donosi značajne odluke o vama isključivo automatizovanim procesima, uz jedan uzak izuzetak opisan u nastavku. Validacija sadržaja se izvršava automatski kada kreirate ili izmenite oglas (na primer, provera zabranjenog sadržaja), ali ona može samo da odbije slanje uz poruku o grešci — odluke o sprovođenju pravila, kao što su zabrana oglasa, onemogućavanje naloga ili rešavanje prijave, donosi ljudski administrator. Jedan izuzetak: kada zapis za prijavljivanje nekog naloga više ne postoji kod našeg provajdera autentifikacije („napušteni” nalog), a taj nalog ima više nerešenih prijava protiv sebe, naš proces čišćenja uklanja napušteni nalog i automatski beleži blokadu ponovne registracije opisanu u odeljku 8.5, bez pojedinačne ljudske odluke. [LAWYER REVIEW: assess this path against GDPR Art. 22 — an automated decision with a 12-month re-registration effect — and whether human review or an objection route should be added.] Ako u budućnosti uvedemo dodatno automatizovano donošenje odluka, ažuriraćemo ovu Politiku privatnosti.

---

## 10. Bezbednost

Preduzimamo razumne tehničke i organizacione mere da zaštitimo vaše podatke o ličnosti od neovlašćenog pristupa, izmene, otkrivanja i gubitka. One uključuju:

- Šifrovanje podataka u prenosu (HTTPS za sve veze između vašeg pregledača i naših servisa).
- Šifrovanje podataka u mirovanju od strane naših infrastrukturnih provajdera (Firebase, Firestore, Cloudflare R2, Postgres na DigitalOcean-u).
- Kontrole autentifikacije i autorizacije koje ograničavaju pristup korisničkim podacima, uključujući potpisani, vremenski ograničen pristup privatnim slikama.
- Beleženje administratorskog pristupa i administratorskih radnji.
- Heširanje sačuvanih imejl adresa i identifikatora u našim revizorskim zapisima o brisanjima i zabranama.

Nijedan sistem nije savršeno bezbedan i ne možemo garantovati apsolutnu bezbednost. Ako saznamo za povredu podataka o ličnosti koja stvara rizik po vaša prava, obavestićemo nadležni nadzorni organ i, kada to GDPR zahteva, obavestićemo vas direktno.

---

## 11. Deca

Oglasino je namenjen korisnicima sa **18 i više godina**. Svesno ne prikupljamo podatke o ličnosti od bilo koga mlađeg od 18 godina. Ako imate manje od 18 godina, molimo vas da se ne registrujete i ne koristite platformu. Ako smatrate da se maloletno lice registrovalo, kontaktirajte privacy@oglasino.com i odmah ćemo preduzeti mere da uklonimo nalog.

---

## 12. Izmene ove Politike privatnosti

Ovu Politiku privatnosti možemo s vremena na vreme ažurirati — na primer, kada dodamo funkcije, promenimo obrađivače ili kao odgovor na izmene zakona. Kada napravimo materijalnu izmenu:

- Ažuriraćemo datum „Poslednje ažuriranje” na vrhu ovog dokumenta.
- Prikazaćemo obaveštenje na platformi.
- Ponovo ćemo prikazati baner za pristanak na kolačiće kako biste mogli da preispitate svoje preferencije kolačića u odnosu na izmene.
- Za izmene koje materijalno utiču na vaša prava ili na način obrade vaših podataka, zatražićemo vaše izričito prihvatanje pre nastavka korišćenja platforme. [LAWYER REVIEW: align acceptance mechanics with Terms §16.]

Nastavak korišćenja platforme nakon što izmena stupi na snagu znači da prihvatate ažuriranu Politiku privatnosti, osim tamo gde smo zatražili vaše izričito prihvatanje. Ako ne prihvatate izmene, možete izbrisati svoj nalog pre nego što stupe na snagu (vidi odeljak 8.3).

---

## 13. Kontakt

Za svako pitanje o ovoj Politici privatnosti ili o tome kako Oglasino obrađuje vaše podatke, kontaktirajte nas na:

**privacy@oglasino.com**

Za sva ostala pitanja (koja se ne tiču privatnosti), molimo vas da koristite opšte kanale podrške platforme.

---

_Kraj nacrta._
