
		Korvaava laboratorio – Verkon automaatio Ansiblella



1. Ympäristö

Harjoituksessa käytin kurssin Containerlab-ympäristöä ja kolmea FRRouting-reititintä (r1, r2 ja r3). SSH-yhteydet eivät aluksi toimineet, joten 
asensin reitittimille tekoälyn avulla OpenSSH:n ja lisäsin Ansiblen julkisen SSH-avaimen. Tämän 
jälkeen yhteydet toimivat.

Tarkistin ympäristön komennolla: sudo containerlab inspect -t golden.clab.yml Kaikki kontit olivat käynnissä.
Testasin SSH-yhteydet ja Ansiblen toiminnan: ansible routers -i inventory/hosts.yml -m raw -a "hostname"
Tulos:
r1-edge | CHANGED | rc=0 >> r1
r2-core | CHANGED | rc=0 >> r2
r3-branch | CHANGED | rc=0 >> r3


2. Inventointi

Tein Ansible-inventaarion ja facts.yml-playbookin, jolla keräsin reitittimien hostnamet, IP-osoitteet, interfacet, käyttöjärjestelmän tiedot ja
uptimen. Kaikissa reitittimissä oli Alpine Linux 3.16.2. Niillä näkyi vain lo- ja eth0-interfacet, vaikka FRRoutingin konfiguraatiossa oli myös
muita interfaceja. Hallintaverkon IP-osoitteet olivat eri reitittimillä erilaiset.

Tarkistin inventaarion komennolla: ansible-inventory -i inventory/hosts.yml --graph
Tulos:
@all:
  |--@ungrouped:
  |--@routers:
  |  |--r1-edge
  |  |--r2-core
  |  |--r3-branch

Keräsin reitittimien tiedot komennolla: ansible-playbook -i inventory/hosts.yml playbooks/facts.yml
Tulos:
r1-edge   : ok=5 changed=0 failed=0
r2-core   : ok=5 changed=0 failed=0
r3-branch : ok=5 changed=0 failed=0

3. Varmuuskopiointi ja Git

Tein backup.yml-playbookin, joka haki reitittimien konfiguraatiot ja tallensi ne tiedostoihin. Lisäsin myöhemmin myös Linuxin IP-osoitteet
varmuuskopioihin, jotta loopbackin muutokset näkyivät. Tallensin alkuperäisen verkkotilan Gitiin ja tein uuden commitin loopbackin lisäämisen
jälkeen. 

Versionhallinnasta on hyötyä, koska sen avulla näkee mitä asetuksia on muutettu ja pystyy vertaamaan aikaisempia konfiguraatioita.

Otin varmuuskopiot komennolla: ansible-playbook -i inventory/hosts.yml playbooks/backup.yml
Tulos:
r1-edge   : ok=3 changed=1 failed=0
r2-core   : ok=3 changed=1 failed=0
r3-branch : ok=3 changed=1 failed=0

Varmuuskopiot tallentuivat backups-kansioon.

Tallensin verkon tilan Gitiin:git add . && git commit -m "Initial network state"
Myöhemmin tarkistin historian: git log --oneline
Tulos:
6108706 Fix loopback recovery
5e72da9 Add branch loopback interface
f56425d Initial network state


4. Loopback ja idempotenssi

Lisäsin r3-reitittimelle lo10-interfacen osoitteella 192.0.2.3/32 Ansiblen avulla. Ensimmäisellä suorituksella changed oli 1 ja toisella 0.
Tämä tarkoittaa idempotenssia, eli sama playbook voidaan suorittaa uudestaan ilman että tehdään turhia muutoksia.

Suoritin playbookin kahdesti: ansible-playbook -i inventory/hosts.yml playbooks/loopback.yml

Ensimmäinen suoritus:
r3-branch : ok=2 changed=1 failed=0

Toinen suoritus:
r3-branch : ok=1 changed=0 skipped=1 failed=0


5. Virhetilanne ja palautuminen

Muutin lo10-interfacen IP-osoitteen tarkoituksella vääräksi (192.0.2.99/32). Otin uuden varmuuskopion ja Gitillä pystyin näkemään, mikä osoite oli
muuttunut. Korjasin loopback.yml-playbookia niin, että se osasi palauttaa oikean IP-osoitteen. Lopuksi verify.yml tarkisti, että kaikki kolme
reititintä olivat saavutettavissa ja lo10:llä oli oikea osoite.

Tarkistin muutoksen Gitillä: git diff -- backups/r3-branch.conf

Tulos:
- inet 192.0.2.3/32
+ inet 192.0.2.99/32

Palautin oikean osoitteen Ansiblen avulla: ansible-playbook -i inventory/hosts.yml playbooks/loopback.yml
Tulos
r3-branch : ok=3 changed=1 failed=0 skipped=1

Tarkistin lopuksi palautuksen:
ansible-playbook -i inventory/hosts.yml playbooks/verify.yml
Tulos:
r1-edge   : ok=1 changed=0 failed=0
r2-core   : ok=1 changed=0 failed=0
r3-branch : ok=3 changed=0 failed=0


6. Pohdinta

- Mitä hyötyä Ansible tarjoaa verrattuna reitittimien käsin konfigurointiin?

Ansiblella voi tehdä samoja muutoksia useille laitteille automaattisesti. Se säästää aikaa ja vähentää käsin tehtäviä virheitä.


- Mitä Ansiblen idempotenssi tarkoittaa käytännössä?

Sitä, että saman playbookin voi ajaa useita kertoja eikä se tee turhia muutoksia, jos asetukset ovat jo oikein.


- Mitä hyötyä Git-versionhallinnasta verkkolaitteiden hallinnassa?

Gitillä voi seurata muutoksia ja tarkistaa, millainen versio oli aikaisemmin. Tämä auttaa myös virheiden selvittämisessä.


- Miten virtuaaliset FRRouting-reitittimet eroavat oikeista verkkolaitteista?

FRRouting toimii tavallisilla Linux-komennoilla. Oikeissa verkkolaitteissa käytetään usein valmistajan
omia komentoja ja käyttöjärjestelmiä. Virtuaaliympäristössä on myös helpompi harjoitella ilman että oikea verkko menee sekaisin.


- Mitä tapahtui simuloidussa virhetilanteessa ja miten palautit oikean konfiguraation?

Loopbackin IP-osoite vaihdettiin vääräksi. Gitin avulla näin muutoksen ja palautin oikean osoitteen. Lopuksi tarkistin, että palautus
onnistui.

