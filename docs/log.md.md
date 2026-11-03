1. Architektura Hub and spoke mimo ze moglismy uzyc zwyklej plaskiej, i do tego czemu mam y w takim razie podsiec w sieci naszej na wirtualki w chmurze nczy to nam nie koliduje
2. Rezerwacja adresu as pfSense
3. Nie p
4. Tunel zestawiony. Faza 1: IKEv2, AES256 / SHA256 / **MODP_1024**, NAT-T (UDP 4500), pfSense jako inicjator. Azure odrzucił DH 14 i wybrał zapasową DH 2. Status w portalu Azure zmienił się na Connected z kilkuminutowym opóźnieniem po zestawieniu tunelu, bo logi pfSense pokazywały już wymianę DPD.

## 2026-11-03 – etap 2, sesja 1: budowa tunelu S2S

**Zrobione**
- Rezerwacja DHCP w Pi-hole dla WAN FW01 → stały adres 192.168.1.158
- Azure: grupa zasobów `rg-hybrid-s2s` (tagi `project`, `env`), sieć `vnet-hub` 10.20.0.0/16 z podsieciami `snet-servers` 10.20.1.0/24 i `GatewaySubnet` 10.20.255.0/27
- Publiczne IP `pip-vgw-hub` (Standard, statyczne) i gateway `vgw-hub` (Basic, route-based) przez Azure CLI w Cloud Shell
- Local Network Gateway `lng-homelab` (publiczny IP domu, 10.10.10.0/24)
- Router Orange: przekierowanie UDP 500 i 4500 → 192.168.1.158
- pfSense: IPsec IKEv2, faza 1 i faza 2 (10.10.10.0/24 ⇄ 10.20.0.0/16)
- Azure: połączenie `conn-homelab` (Site-to-site, IKEv2, PSK)

**Wynik:** tunel zestawiony – Azure: Connected, pfSense: Established.
- Faza 1: AES256 / SHA256 / **MODP_1024 (DH 2)**, NAT-T (UDP 4500), pfSense jako inicjator
- Faza 2: … (uzupełnić z *Show child SA entries*)

**Problemy i obserwacje**
- WAN pfSense miał adres 192.168.1.158 zamiast udokumentowanego .85 – adres z DHCP bez rezerwacji. Naprawione rezerwacją w Pi-hole (DHCP w sieci domowej robi Pi-hole, nie router Orange).
- Gateway Basic nie da się utworzyć z portalu – tylko CLI/PowerShell.
- Komenda tworzenia gatewaya wykonała się bez `--tags` i `--no-wait`. `Ctrl+C` przerwał tylko oczekiwanie w CLI, nie operację w Azure. Tagi dodane później przez `az tag update --operation Merge`.
- Tworzenie gatewaya: ok. … min (od uruchomienia komendy do `Succeeded`).
- pfSense proponował DH 14 i zapasowo DH 2 – Azure Basic wybrał DH 2. Zgodne z dokumentacją Microsoftu; Basic nie obsługuje własnych polityk IPsec.
- Po zestawieniu tunelu portal Azure przez kilka minut pokazywał „Not connected”, choć logi pfSense pokazywały już wymianę DPD z Azure. Status w portalu aktualizuje się z opóźnieniem – najpierw sprawdzać logi.

**Czego się nauczyłem**
- Bez rezerwacji adresu przekierowanie portów może przestać działać bez żadnego komunikatu.
- Przy pfSense za NAT-em trzeba ustawić *My identifier* na publiczny adres.
- ESP nie przechodzi przez NAT, dlatego potrzebny jest NAT-T (ESP w UDP 4500).
- `<con1|3>` w logach strongSwan to połączenie con1, trzecie IKE SA; puste `INFORMATIONAL` to DPD.

**Następnym razem**
- [ ] Sprawdzić i zapisać algorytmy fazy 2
- [ ] Reguły firewalla na interfejsie IPsec (ICMP do labu, porty AD tylko do DC01)
- [ ] VM testowa w `snet-servers`, testy łączności, DNS, dołączenie do domeny
- [ ] Usunąć `rg-hybrid-s2s` po zakończeniu testów (gateway płatny za godzinę)