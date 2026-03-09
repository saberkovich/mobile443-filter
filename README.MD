## Как это работает

Скрипт собирает актуальные IPv4-префиксы по списку ASN через RIPEstat (`announced-prefixes`), загружает их в `ipset allowed_mobile_443` и навешивает `iptables`-цепочку `FILTER_MOBILE_443` на `INPUT`, `FORWARD` и `DOCKER-USER`.

Логика простая:

- если source IP входит в `allowed_mobile_443` → `ACCEPT`
- если source IP не входит → `DROP`

Обновление безопасное:

- новые префиксы сначала собираются во временный файл
- затем загружаются во временный `ipset`
- после проверки делается `swap`
- если API ответил криво, боевой список не очищается
- последний валидный список хранится в кэше
- обновление запускается ежедневно в `00:00`

## Какие ASN входят

### MTS
- `8359`

### Beeline / VimpelCom
- `3216`
- `16345`

### MegaFon core + related
- `31133`
- `8263`
- `6854`
- `50928`
- `48615`
- `47395`
- `47218`
- `43841`
- `42891`
- `41976`
- `35298`
- `34552`
- `31268`
- `31224`
- `31213`
- `31208`
- `31205`
- `31195`
- `31163`
- `29648`
- `25290`
- `25159`
- `24866`
- `20663`
- `20632`
- `12396`
- `202804`

### T2 regional
- `12958`
- `15378`
- `42437`
- `48092`
- `48190`
- `41330`

### Miranda
- `201776`

### Sberbank-Telecom
- `206673`

### Rostelecom
- `12389`

### T-Mobile / Alfa-Mobile
- `205638`
- `214257`

## Важно

Это **mobile-focused allowlist**, а не идеальный детектор “только SIM”.  
Часть fixed-клиентов крупных операторов (`MTS`, `Beeline`, `MegaFon`, `Rostelecom`) тоже может проходить, потому что их ASN используются не только для мобильного доступа.

# mobile443-filter

Фильтр 443/tcp и 443/udp по allowlist мобильных ASN РФ.

## Установка
```bash
chmod +x asn.sh
sudo ./asn.sh install
### Удаление
sudo ./asn.sh remove

## Диагностика

```bash
# Проверить, входит ли IP в allowlist
ipset test allowed_mobile_443 1.2.3.4

# Количество подсетей в allowlist
ipset list allowed_mobile_443 | grep 'Number of entries'

# Первые 50 подсетей
ipset list allowed_mobile_443 | sed -n '/^Members:/,$p' | tail -n +2 | head -50

# Счётчики ACCEPT / DROP
iptables -L FILTER_MOBILE_443 -n -v --line-numbers

# Смотреть счётчики в реальном времени
watch -n 1 'iptables -L FILTER_MOBILE_443 -n -v --line-numbers'

# Посмотреть, через какую цепочку идёт трафик
iptables -L INPUT -n -v --line-numbers | head -30
iptables -L FORWARD -n -v --line-numbers | head -30
iptables -L DOCKER-USER -n -v --line-numbers

# Посмотреть пакеты на 443
tcpdump -ni any port 443

# Посмотреть пакеты от конкретного IP
tcpdump -ni any host 1.2.3.4 and port 443

# Проверить обновление и таймер
systemctl status mobile443-update.service --no-pager
systemctl status mobile443-update.timer --no-pager
journalctl -u mobile443-update.service -n 100 --no-pager
journalctl -u mobile443-apply.service -n 100 --no-pager
