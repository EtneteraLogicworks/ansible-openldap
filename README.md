# Role openldap

Role openldap:

- Nainstaluje `OpenLDAP` server.
- Provede jeho konfiguraci.
- Vytvoří požadované datové objekty v OpenLDAP databázi.
- Zprovozní master <-> backup replikaci.
- Nastaví pravidelné zálohování OpenLDAP databáze.

Závislosti:

- Je nutné mít nainstalovaný balík `python-ldap` na počítači, který spouští Ansible.


## Proměnné

| Proměnná         | Povinná | Výchozí              | Popis                                |
| ---------------- | ------- | -------------------- | ------------------------------------ |
| openldap         | ano     |                      | Slovník pro konfiguraci              |
| .anonymous_bind  | ne      | false                | Umožní anononymní přístup k databázi |
| .datadir_path    | ne      | `/srv/database/ldap` | Cesta k souborům databáze OpenLDAP   |
| .domain          | ne      | logicworks.cz        | Výchozí doména pro konfiguraci       |
| .groups          | ne      |                      | Čárkou oddělený seznam systémových skupin, do kterých bude přidán uživatel `openldap` |
| .indices         | ne      |                      | Pole nastavení indexů OpenLDAP databáze |
| .organization    | ne      | logicworks           | O výchozí databáze                   |
| .searchbase      | ne      | dc=logicworks,dc=cz  | DC výchozí databáze                  |
|                  |         |                      |                                      |
| .admin           | ano     |                      | Slovník nastavení admin uživatele    |
| ..cn             | ne      | admin                | CN OpenLDAP root uživatele. Nesmí být jiné než CN v admin.dn |
| ..dn             | ne      | cn=admin,dc=logicworks,dc=cz | DN OpenLDAP root uživatele. Musí obsahovat admin.cn CN |
| ..password       | ano     |                      | Heslo OpenLDAP root uživatele        |
|                  |         |                      |                                      |
| .backup          | ne      |                      | Slovník pro nastavení zálohování     |
| ..dir            | ne      | /var/backups         | Adresář, kam se bude zálohovat.      |
| ..enabled        | ne      | true                 | Zapíná zálohovací skript             |
| ..schedule.(minute\|hour\|day\|month\|weekday) | ano | | Nastavení intervalu záloh v `cron` |
| ..retention.days | ne      | 1                    | Proměnná pro snippet `remove_backups` |
| ..retention.min  | ne      | 24                   | Proměnná pro snippet `remove_backups` |
|                  |         |                      |                                      |
| .replication     | ne      |                      | Slovník pro nastavení replikace      |
| ..enabled        | ne      | false                | Zapíná replikaci                     |
| ..retry          | ne      | 1 10 10 +            | Retry interval viz replikace         |
| ..user           | ne      |                      | Slovník replikačního uživatele       |
| ...cn            | ne      | replicator           | CN replikačního uživatele. Nesmí být jiné než CN v replication.user.dn |
| ...create        | ne      | true                 | Vytvoří uživatele v OpenLDAP databázi |
| ...dn            | ne      | cn=replicator,dc=logicworks,dc=cz | DN replikačního uživatele. Musí obsahovat replication.user.cn CN |
| ...password      | ano     |                      | Heslo uživatele pro replikaci        |
|                  |         |                      |                                      |
| .tls             | ne      |                      | Slovník nastavení TLS (StartTLS)     |
| ..enabled        | ne      | false                | Zapne `STARTTLS` metodu na standardním portu 389 |
| ..ldaps          | ne      | false                | Zapne *navíc* `LDAPS` na portu 636; `STARTTLS` musí být zapnuto (`tls.enabled` musí být `true`) |
| ..force          | ne      | false                | Vynutí TLS pro každé spojení         |
| ..ca_file        | ne      | /etc/ssl/certs/ca-certificates.crt | Cesta k pem souboru s certifikáty CA. Nemělo by být potřeba nastavovat. |
| ..cert_file      | ne      | /etc/ssl/certs/ssl-cert-snakeoil.pem | Cesta k pem souboru s certifikátem |
| ..key_file       | ne      | /etc/ssl/private/ssl-cert-snakeoil.key | Cesta k souboru s privátním klíčem  |
| .priority_string | ne      | Viz defaults         | Priority string pro GnuTLS, které OpenLDAP využívá v Debianu |
|                  |         |                      |                                      |
| openldap_acl     | ne      |                      | Pole pro nastavení ACL               |
| openldap_data    | ne      |                      | Slovník pro vytvoření objektů v databázi |
| .attributes      | ne      |                      | Pole k nastavení specifických atributů |
| .users           | ne      |                      | Pole k vytvoření specifických uživatelů |
| .objects         | ne      |                      | Pole k vytvoření specifických objektů |
| .usergroups      | ne      |                      | Slovník pro konfiguraci základních skupin a jejich uživatelů viz sekce Vytváření objektů |
| ..users          | ne      |                      | Pole uživatelů                       |
| ...name          | ano     |                      | Jméno uživatele                      |
| ...password      | ano     |                      | Heslo uživatele                      |
| ...description   | ne      | Basic user           | Popis objektu uživatele              |
| ..groups         | ne      |                      | Pole skupin                          |
| ...name          | ano     |                      | Jméno skupiny                        |
| ...members       | ano     |                      | Pole jmen uživatelů, kteří jsou členy skupiny |
| ...description   | ne      | Basic group          | Popis objektu skupiny                |



## Replikace

Pro master<->backup využíváme OpenLDAP multimaster [replikaci](https://www.openldap.org/doc/admin24/replication.html).
Používáme režim `olcMirrorMode`, díky kterému je možné zapisovat do master i backup databáze.
Předpokládá se, že souběžný zápis si ošetříme sami, což děláme pomocí `keepalived`.

Atribut `olcSyncrepl` nastavujeme následovně (více info v [man](https://www.openldap.org/software/man.cgi?query=slapd-config&sektion=5&apropos=0&manpath=OpenLDAP+2.4-Release)
stránce):

- `rid=001` : ID replikačního procesu
- `provider=ldap://{{ failover_mirror }}` : URL serveru, ze kterého replikujeme
- `starttls={{ "critical" if openldap.tls.force else "yes" }}` : Vynucujeme TLS pomocí `critical`, když víme, že je TLS nastaveno.
- `searchbase="{{ openldap.searchbase }}"`
- `binddn="{{ openldap.replication.user.dn }}"` : Uživatel pro replikaci.
- `credentials="{{ openldap.replication.user.password }}"` : Heslo uživatele pro replikaci.
- `retry="{{ openldap.replication.retry }}"` : Nastavení intervalu pokusů o znovunavázání spojení.
- `schemachecking=on` : Validita každého write update je ověřena vůči schématu.
- `type=refreshAndPersist` : Nastavení neustálé replikace.
- `interval=00:00:01:00` : Nastavujeme dd:hh:mm:ss interval pro jistotu. Z man stránky by
  se dalo usuzovat, že když nastane nějaký problém během neustálé replikace, mohl by další
  pokud proběhnout až za `interval`, který je defaultně den.


## Vytváření objektů

V LDAP databázi můžeme pomocí vytvořit objekty. K tomu používáme slovník `openldap_data`.
Máme dvě možnosti, jak objekty vytvářet.

### Jednoduché skupiny a uživatelé

Pro snadnou obsluhu základní LDAP skupin a uživatelů, můžeme použít proměnnou
`openldap_data.usergroups`. Její existence zajistí vytvoření:

1. objektů `ou=groups,{{ openldap.searchbase }}` a `ou=users,{{ openldap.searchbase }}`,
2. uživatelů z `openldap_data.usergroups.users` v `ou=users,{{ openldap.searchbase }}`
   a nastavení jejich hesla,
3. skupin z `openldap_data.usergroups.groups`  v `ou=groups,{{ openldap.searchbase }}`
   a nastavení jejich členů.

Příklad:

```yaml
openldap_data:
  usergroups:
    users:
      - name: 'someuser'
        password: '{{ somepassword }}'
        description: 'Some user description'
    groups:
      - name: 'somegroup'
        description: "Some group description"
        members:
          - 'someuser'
```

### Specifické objekty

Pokud chceme upravovat objekty či jejich atributy kdekoliv v LDAP stromu, použijeme
proměnné `openldap_data.attributes`, `openldap_data.objects` a speciálně pro uživatele
`openldap_data.users`.

Příklad:

```yaml
openldap_data:
  attributes:
    - dn: 'dc=logicworks,dc=cz'
      attribute: 'description'
      value: 'LDAP je cool'
  users:
    - dn: 'cn=drak,dc=logicworks,dc=cz'
      attributes:
        description: "Drak user"
        cn: 'drak'
        userPassword: 'heslo'
  objects:
    - dn: 'ou=users,dc=logicworks,dc=cz'
      objectClass: organizationalUnit
      attributes:
        description: 'LDAP access users'
        ou: 'users'
```

Proměnná `openldap_data.users` existuje pro usnadnění nastavení hesla objektu uživatele.
Uživatele můžeme vytvořit také pomocí proměnné `openldap_data.objects`, ale musíme již
v Ansible správně vyřešit hash hesla.



## Příklad použití

Nastavení TLS. Použijeme Let's Encrypt certifikát získaný pomocí `simp_le`.
Také vynutíme zabezpečená spojení.


```yaml
openldap:
  groups: 'letsencrypt'
  tls:
    enabled: true
    force: true
    cert_file: '/var/lib/letsencrypt/simp_le/{{ inventory_hostname_short }}/fullchain.pem'
    key_file: '/var/lib/letsencrypt/simp_le/{{ inventory_hostname_short }}/key.pem'
```


Nastavení indexů:

```yaml
openldap:
  indices:
    - 'o,ou eq'
    - 'mail eq'
    - 'entryCSN,entryUUID eq'
```


Nastavení ACL:

```yaml
openldap_acl:
  - >-
    {0}to attrs=userPassword
    by self write
    by group.exact="cn=ldapwrite,ou=groups,dc=logicworks,dc=cz" write
    by dn="{{ openldap.replication.user.dn }}" read
    by anonymous auth
    by * none
  - >-
    {1}to attrs=shadowLastChange
    by self write
    by group.exact="cn=ldapwrite,ou=groups,dc=logicworks,dc=cz" write
    by dn="{{ openldap.replication.user.dn }}" read
    by * read
  - >-
    {2}to *
    by anonymous auth
    by group.exact="cn=ldapwrite,ou=groups,dc=logicworks,dc=cz" write
    by users read
```

## Anonymní bind a autentizace

Původně jsme chtěli nastavit parametr `olcRequires` -> `authc` i pro frontend databázi.
To však nefunguje dobře s macOS klienty LDAP. Budeme tedy používat zákaz anonymního bindu
`olcDisallows` -> `bind_anon`, vyžádání autentizace pro backend databázi a ACL.

Rozdíl mezi parametry se řeší např. v OpenLDAP [mailing listu](https://www.openldap.org/lists/openldap-software/200305/msg00712.html).

Původní Ansible task k vyžádání autentizace na frontend databázi.

```yaml
- name: Vyžádání autentizace
  ldap_attr:
    dn: 'olcDatabase={-1}frontend,cn=config'
    name: 'olcRequires'
    values: 'authc'
    state: "exact"
```
