РАЗВЕРТЫВАНИЕ VK-TURN VPN НА НОВОМ СЕРВЕРЕ (Linux + Docker) — v2 (2026-10-01)
==============================================================================
Схема: клиенты -> звонки ВК (белый список, ТСПУ пропускает)
       -> ВАШ_СЕРВЕР:56000/UDP (vk_turn, wrap-мимикрия под SRTP)
       -> WireGuard 10.88.0.0/24 -> интернет
Наружу торчит ТОЛЬКО UDP 56000. WG-порт 51820 наружу не светится.

Референс-деплой этой конфигурации: сервер + ПК + телефон работают,
скорость 7.6 Мбит на ПК (wrap ON).

ЧТО В ПАПКЕ (после распаковки)
------------------------------
docker-compose.yml  — два контейнера: vk_turn (форк MYSOREZ v1.5.3, wrap ON)
                      и vk_wg (WireGuard + NAT)
wg0.conf            — конфиг WG-сервера: 10.88.0.1 + 2 пира (ПК .2, телефон .3)
SECRETS.txt         — ВСЕ ключи и пароль wrap (chmod 600, никому не показывать)
README-SERVER.txt   — этот файл

ТРЕБОВАНИЯ К СЕРВЕРУ
--------------------
- Linux x86_64 (Ubuntu/Debian любой свежий), публичный IPv4
- UDP не блокируется провайдером (обычный VPS — ок)
- 1 vCPU / 1 GB RAM достаточно (референс: 2 vCPU / 1.9 GB, Xray туда же помещается)
- Docker + compose plugin. Если нет:
    curl -fsSL https://get.docker.com | sh

ШАГ 1. ЗАЛИТЬ И РАСПАКОВАТЬ
---------------------------
Архив vk-turn-deploy-server.zip переслать на сервер (scp с ПК:
  scp vk-turn-deploy-server.zip root@IP_СЕРВЕРА:/root/
), затем на сервере:
  apt install -y unzip    # если unzip нет
  cd /root && unzip vk-turn-deploy-server.zip && cd vk-turn
  chmod 600 SECRETS.txt wg0.conf

ШАГ 2. ЗАПУСК
-------------
  docker compose up -d
(проект можно запускать с любым именем: docker compose -p vkturn up -d)
Первый запуск скачает образы (~50 МБ).

ШАГ 3. ФАЙРВОЛ
--------------
Открыть UDP 56000 (если включён ufw/облако-firewall):
  ufw allow 56000/udp
Проверить в панели провайдера (Security Group), что UDP 56000 разрешён.

ШАГ 4. ПРОВЕРКА СЕРВЕРА
-----------------------
  docker ps                          # vk_turn и vk_wg: Up
  docker logs vk_turn --tail 10
     ОЖИДАЕМ: "... wrap=true ..." и "WRAP mode enabled: RTP-AEAD obfuscation"
     Если wrap=false — compose не применился: docker compose up -d --force-recreate
  docker exec vk_wg wg show          # interface wg0, 2 peers, без handshake (пока некому)
  docker exec vk_wg iptables -t nat -L POSTROUTING -n | grep -i masq
     ОЖИДАЕМ: строка MASQUERADE. Если ПУСТО (грабля RX=0):
     docker exec vk_wg sh -c "iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT"
  ss -uln | grep 56000               # порт слушается

ШАГ 5. КЛИЕНТЫ
--------------
- ПК: архив vk-turn-deploy-pc.zip (инструкция внутри)
- Телефон: архив vk-turn-deploy-android.zip (инструкция внутри)
Порядок всегда: сначала клиент устанавливает релей -> потом включается WG.

ЗВОНКИ ВК (одна ссылка на все устройства)
-----------------------------------------
Нужен аккаунт ВК: vk.com -> звонки -> создать звонок -> скопировать ссылку
https://vk.com/call/join/...  НЕ нажимать «Завершить звонок для всех» —
ссылка живёт вечно. Один участник — нормальный режим.
Лимиты ВК на звонки: 9-18 потоков на аккаунт стабильны (больше — риск
капч и 486 Allocation Quota). Референс: -n 9 на ПК, -n 9 на телефоне.

ОБСЛУЖИВАНИЕ
------------
| Хочу | Команда |
|---|---|
| перезапуск контейнеров | docker compose restart |
| применить правку compose | docker compose up -d |
| применить правку wg0.conf | docker compose up -d --force-recreate (bind-mount держит inode!) |
| логи turn | docker logs vk_turn -f |
| статистика WG | docker exec vk_wg wg show |
| выключить всё | docker compose down (файлы остаются) |

ГРАБЛИ (все проверены на живом деплое)
--------------------------------------
1. Entrypoint образа MYSOREZ игнорирует command и не умеет -password.
   В этом compose он УЖЕ переопределён: entrypoint: ["./vk-turn-proxy"].
   Не удаляйте эту строку, иначе сервер стартует с wrap=false, а клиенты
   ловят «WRAP_AUTH_TIMEOUT: пароль не подтверждён».
2. Скорость падает до 0.1-1 Мбит = ВК включил шейпинг голого протокола.
   Лечится wrap (уже включён). Проверка: wrap=true в логах.
3. Клиент не может достучаться до ВК (context deadline exceeded) —
   у клиента нет прямого интернета до vk.com; клиент должен запускаться
   ДО включения WG.
4. WG-handshake есть, интернета нет (RX растёт, TX=0 у клиента):
   NAT не применился — команда из ШАГА 4 (iptables руками).
5. Правка wg0.conf не применяется рестартом: только
   docker compose up -d --force-recreate (bind-mount держит inode).
6. ТЕЛЕФОН: со стоковыми ядрами НЕ работает ни одно — нужен самосборный
   client-android-arm64-v6 «MASYA-PATCH» из android-архива (грабля №8:
   позиционный peer + -udp + SELinux netlink; детали в README-ANDROID).
7. Новый peer (третий телефон и т.п.): добавить [Peer] в wg0.conf с новым
   IP из 10.88.0.0/24 и AllowedIPs = 10.88.0.X/32, затем force-recreate;
   на клиенте сгенерировать пару ключей (wg genkey | tee privatekey |
   wg pubkey > publickey) и вписать PublicKey в серверный wg0.conf.
