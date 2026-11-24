Każdy playbook puszczamu z hasłem, hasło użytkownika szyfrujemy poleceniem:

    ansible-vault create passwd.yml

W pliku zapisujemy hasło w formacie:

    servers_sudo_pass: twoje_hasło

Następnie przy wywołaniu playbook-a dodajemy parametry jak w poniższym przykładzie:

    ansible-playbook -i inventory.yml --ask-vault-pass --extra-vars '@passwd.yml' 01_general-server-configuration.yml

Wtedy ansible poprosi nas o hasło utworzone w pierwszym kroku. Proponuje użyć hasła systemowego (by je pamiętać)


# Środowisko AWS – 3 serwery Ubuntu

## Cel

Uruchomienie trzech maszyn wirtualnych Ubuntu 24.04 LTS w jednej sieci AWS VPC. Każda maszyna ma publiczny adres IPv4 do połączeń SSH z komputera administratora oraz prywatny adres IPv4 do komunikacji wewnątrz VPC.

## Architektura i sieć

| Element | Wartość |
|---|---|
| Region AWS | `eu-north-1` (Europe – Stockholm) |
| VPC | `vpc-0b9a4452d9800d87a` (Sieć-vpc) |
| CIDR VPC | `10.0.0.0/16` |
| Publiczna podsieć | `subnet-072896c7d7f9a4db0` (Sieć-public-subnet) |
| CIDR publicznej podsieci | `10.0.1.0/24` |
| Internet Gateway | `igw-0c5034b7225ce738b` |
| Publiczna tablica routingu | `rtb-03fba84ac498ba861` (Sieć-public-rtb) |
| Security Group | `sg-0d60812edc24677ec` (ubuntu-security) |

### Routing

Publiczna tablica routingu powinna zawierać:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | `igw-0c5034b7225ce738b` |

**Ważne:** Publiczna podsieć `subnet-072896c7d7f9a4db0` musi być jawnie przypisana do `Sieć-public-rtb`. Podczas diagnostyki wykryto, że do tej tablicy przypisana była prywatna podsieć `10.0.128.0/20`, a publiczna podsieć korzystała z głównej tablicy routingu. To powodowało brak prawidłowej trasy przez Internet Gateway.

Prywatna podsieć powinna pozostać przypisana do prywatnej tablicy routingu.

W ustawieniach publicznej podsieci należy włączyć **Auto-assign public IPv4 address**.

## Specyfikacja maszyn

Wszystkie trzy instancje zostały uruchomione w regionie `eu-north-1`, w strefie dostępności `eu-north-1a`.

| Nazwa | Instance ID | System | Typ | RAM | Publiczny IPv4 | Prywatny IPv4 |
|---|---|---|---|---|---|---|
| Ubuntu Serwer 1 | `i-0bcfaab0204af4067` | Ubuntu Server 24.04 LTS | t3.micro | 1 GiB | `16.171.224.14` | DO UZUPEŁNIENIA |
| Ubuntu Serwer 2 | `i-08636b8b019b2f1dc` | Ubuntu Server 24.04 LTS | t3.micro | 1 GiB | `13.63.154.111` | DO UZUPEŁNIENIA |
| Ubuntu Serwer 3 | `i-07fa981db687c1dd0` | Ubuntu Server 24.04 LTS | t3.micro | 1 GiB | `13.48.29.215` | DO UZUPEŁNIENIA |

Typ `t3.micro` zapewnia **2 vCPU i 1 GiB RAM**. Wszystkie instancje miały status **Running oraz 3/3 checks passed** w chwili przekazania ich danych.

Adresy prywatne należy uzupełnić z widoku **EC2 → Instances → Private IPv4 address**. Do komunikacji między serwerami należy używać właśnie adresów prywatnych, a nie publicznych.

## Security Group – ubuntu-security

Ta sama Security Group powinna być przypisana do wszystkich trzech instancji.

### Reguły przychodzące (Inbound)

| Typ | Protokół / port | Źródło | Przeznaczenie |
|---|---|---|---|
| SSH | TCP / 22 | `93.174.27.158/32` | SSH z komputera administratora |
| SSH | TCP / 22 | `sg-0d60812edc24677ec` | SSH pomiędzy instancjami |
| All ICMP - IPv4 | ICMP / All | `sg-0d60812edc24677ec` | Ping pomiędzy instancjami |

**Adres `93.174.27.158/32` musi być aktualnym publicznym adresem IP komputera administratora.** Jeśli adres się zmieni, należy zaktualizować regułę SSH.

### Reguły wychodzące (Outbound)

Zalecana reguła dla tego laboratorium:

| Typ | Protokół / port | Cel |
|---|---|---|
| All traffic | All | `0.0.0.0/0` |

**Nie należy otwierać SSH dla całego Internetu (`0.0.0.0/0`).** Dostęp administracyjny powinien być ograniczony do zaufanego adresu IP.

## Tworzenie instancji EC2

Dla każdej z trzech maszyn zastosowano następujące ustawienia:

1. Otworzyć **EC2 → Instances → Launch instances**.
2. Wybrać obraz Ubuntu Server 24.04 LTS, architektura 64-bit x86 (AMD64).
3. Wybrać typ instancji `t3.micro` (2 vCPU, 1 GiB RAM).
4. Wybrać parę kluczy SSH. W środowisku używany jest klucz `Krzysztof-key` (plik prywatny `.pem` należy bezpiecznie przechowywać).
5. W ustawieniach sieci wybrać VPC `vpc-0b9a4452d9800d87a` i publiczną podsieć `subnet-072896c7d7f9a4db0`.
6. Ustawić **Auto-assign public IP: Enable**.
7. Przypisać Security Group `ubuntu-security`.
8. Uruchomić instancję i zaczekać na status **3/3 checks passed**.

## Połączenie SSH z komputera administratora

Na komputerze z Ubuntu, w katalogu zawierającym plik klucza:

```bash
chmod 400 Krzysztof-key.pem
```

Połączenie z poszczególnymi serwerami:

**Ubuntu Serwer 1:**

```bash
ssh -i Krzysztof-key.pem ubuntu@16.171.224.14
```

**Ubuntu Serwer 2:**

```bash
ssh -i Krzysztof-key.pem ubuntu@13.63.154.111
```

**Ubuntu Serwer 3:**

```bash
ssh -i Krzysztof-key.pem ubuntu@13.48.29.215
```

Domyślny użytkownik dla oficjalnego obrazu Ubuntu na EC2 to `ubuntu`.

## Testy sieci

### Sprawdzenie adresów IP na każdej maszynie

```bash
hostname -I
```

Test komunikacji między serwerami należy wykonać z użyciem ich prywatnych adresów:

```bash
ping <PRYWATNY_IP_UBUNTU_2>
ping <PRYWATNY_IP_UBUNTU_3>
```

Po skonfigurowaniu uwierzytelniania kluczami można również testować SSH pomiędzy serwerami:

```bash
ssh ubuntu@<PRYWATNY_IP_INNEJ_MASZYNY>
```

Do takiego połączenia potrzebna jest odpowiednia konfiguracja kluczy SSH; **nie należy kopiować prywatnego klucza administratora `.pem` na wszystkie serwery.**

## Uwagi i bezpieczeństwo

- Publiczne adresy IPv4 instancji mogą się zmienić po zatrzymaniu i ponownym uruchomieniu maszyny. Jeśli wymagane są stałe adresy, należy rozważyć przypisanie **Elastic IP**.
- Publiczny dostęp SSH jest ograniczony do adresu administratora `93.174.27.158/32`.
- Prywatne adresy IP należy uzupełnić w tabeli powyżej po odczytaniu ich z konsoli AWS.
- Po zakończeniu laboratorium należy zatrzymać lub usunąć nieużywane zasoby, aby uniknąć niepotrzebnych kosztów. Samo zatrzymanie instancji nie musi zwalniać wszystkich opłat za powiązane zasoby, np. woluminy EBS lub przypisane Elastic IP.
- Wcześniejsza diagnostyka wykazała błąd w przypisaniu podsieci do tablicy routingu. Po poprawce należy potwierdzić, że publiczna podsieć jest przypisana do publicznej tablicy routingu oraz że SSH działa. W rozmowie nie ma końcowego potwierdzenia udanego logowania, dlatego warto wykonać test SSH po sprawdzeniu tej asocjacji.
