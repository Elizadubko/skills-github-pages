<header>

<!--
  <<< https://www.transfernow.net/dl/20250526xtHawM6c/NPIkx8xh Модуль 1
Таблица 1. Итоговая таблица адресации (Модуль 1)
Устройство	IP-адрес	Маска	Шлюз
ISP	DHCP	DHCP	DHCP
	172.16.4.1	255.255.255.240	-
	172.16.5.1	255.255.255.240	-
HQ-RTR	172.16.4.4	255.255.255.240	172.16.4.1
	-	-	-
	192.168.1.1	255.255.255.192	-
	192.168.2.1	255.255.255.240	-
	192.168.99.1	255.255.255.248	-
	192.168.5.1	255.255.255.252	-
HQ-SRV	192.168.1.10	255.255.255.192	192.168.1.1
HQ-CLI	DHCP (192.168.2.10)	DHCP (255.255.255.240)	192.168.2.1
BR-RTR	172.16.5.5	255.255.255.240	172.16.5.1
	192.168.3.1	255.255.255.224	-
	192.168.5.2	255.255.255.252	-
BR-SRV	192.168.3.10	255.255.255.224	192.168.3.1

Шаг 4: настройка виртуального коммутатора (VLAN) на HQ-RTR
Создание директорий конфигурации: Для каждого VLAN-подынтерфейса (vlan100, vlan200, vlan999) была создана соответствующая директория в /etc/net/ifaces/.
Определение параметров VLAN: В файле options для каждого подынтерфейса указан тип vlan, базовый физический интерфейс (HOST=ens19) и идентификатор VLAN (VID=100, VID=200, VID=999).
Назначение IP-адресов: В файле ipv4address для каждого VLAN-подынтерфейса был настроен статический IP-адрес и маска подсети, соответствующая требованиям задания (или размерам подсетей из Таблицы 1 данного пособия):
vlan100: 192.168.1.1/26 // ВАРИАТИВНО: Маска может отличаться.
vlan200: 192.168.2.1/28 // ВАРИАТИВНО: Маска может отличаться.
vlan999: 192.168.99.1/29 // ВАРИАТИВНО: Маска может отличаться.
Применение конфигурации: Сетевая служба была перезапущена (systemctl restart network) для активации созданных интерфейсов и применения настроек.
Итог: В результате выполненных настроек на маршрутизаторе HQ-RTR реализована виртуальная коммутация на базе VLAN. Интерфейс ens19 функционирует как транковый порт, а созданные VLAN-подынтерфейсы (vlan100, vlan200, vlan999) обеспечивают маршрутизацию между соответствующими подсетями и остальной частью сети. Трафик между сервером, клиентом и сетью управления логически изолирован на канальном уровне.

Шаг 6: настройка IP-туннеля (GRE) между HQ-RTR и BR-RTR
Основные шаги конфигурации:
Создание директорий конфигурации: На обоих маршрутизаторах (HQ-RTR и BR-RTR) была создана директория /etc/net/ifaces/gre1.
Назначение IP-адресов туннелю: В файле ipv4address для интерфейса gre1 были настроены статические IP-адреса из подсети 192.168.5.0/30:
HQ-RTR: 192.168.5.1/30
BR-RTR: 192.168.5.2/30
Определение параметров GRE-туннеля: В файле options для интерфейса gre1 на каждом маршрутизаторе были указаны:
Тип интерфейса: TYPE=iptun
Тип туннеля: TUNTYPE=gre
Локальный IP-адрес: TUNLOCAL (внешний IP маршрутизатора на ens18)
Удалённый IP-адрес: TUNREMOTE (внешний IP другого маршрутизатора на ens18)
TTL для туннельных пакетов: TUNTTL=64
Применение конфигурации: Сетевая служба была перезапущена (systemctl restart network) на обоих маршрутизаторах.
Итог: В результате настройки между HQ-RTR и BR-RTR создан GRE-туннель. Виртуальные интерфейсы gre1 на обоих маршрутизаторах имеют IP-адреса и могут использоваться для маршрутизации трафика между локальными сетями HQ и BR через публичную сеть ISP. Это создаёт основу для дальнейшей настройки маршрутизации между офисами.

Шаг 7: настройка динамической маршрутизации (OSPF) между HQ-RTR и BR-RTR
Основные шаги конфигурации и защита:
Установка FRR: На оба маршрутизатора (HQ-RTR и BR-RTR) был установлен пакет Free Range Routing (frr).
Включение OSPF: В конфигурации FRR (/etc/frr/daemons) был активирован демон ospfd.
Настройка OSPF (/etc/frr/frr.conf):
Заданы уникальные Router ID (192.168.5.1 на HQ-RTR, 192.168.5.2 на BR-RTR).
Интерфейсы gre1 и локальные (vlanXXX на HQ, ens19 на BR) включены в Area 0.0.0.0 (ip ospf area 0.0.0.0).
Установлена опция passive-interface default для запрета установления соседства на локальных интерфейсах по соображениям безопасности.
Интерфейс gre1 сделан активным (no ip ospf passive) для разрешения установления OSPF-соседства через туннель.
Защита протокола: На интерфейсе gre1 на обоих маршрутизаторах настроена простая парольная аутентификация (ip ospf authentication) с одинаковым ключом P@$$word (ip ospf authentication-key P@$$word). Это предотвращает установление неавторизованного соседства.
Применение конфигурации: Служба frr перезапущена (systemctl restart frr).
Итог: На маршрутизаторах HQ-RTR и BR-RTR настроена динамическая маршрутизация OSPF, работающая через GRE-туннель с использованием парольной аутентификации. Маршрутизаторы должны обмениваться информацией о доступных локальных сетях, обеспечивая сквозную связность между офисами HQ и BR.

Шаг 9: настройка протокола динамической конфигурации хостов (DHCP) на HQ-RTR
Основные шаги конфигурации:
Установка dnsmasq: Пакет dnsmasq установлен на HQ-RTR.
Конфигурация /etc/dnsmasq.conf на HQ-RTR:
Настроено прослушивание только на интерфейсе vlan200 (listen-address=192.168.2.1).
Определён динамический пул адресов 192.168.2.2 - 192.168.2.14 (dhcp-range). Примечание: Диапазон и маска могут отличаться в зависимости от варианта задания.
Добавлена статическая привязка (dhcp-host) для выдачи IP-адреса 192.168.2.10 клиенту с Client ID "hq-cli-exam-id".
Настроена передача DHCP-опций: DNS-сервер 192.168.1.10 (опция 6) и DNS-суффикс au-team.irpo (опция 15).
Настройка клиента (HQ-CLI):
Сетевой интерфейс ens18 настроен на получение адреса по DHCP (BOOTPROTO=dhcp).
В файле /etc/dhcpcd.conf активирована директива clientid hq-cli-exam-id (или другой ID из задания).
Запуск служб: Служба dnsmasq на HQ-RTR включена и перезапущена. Сетевая служба на HQ-CLI перезапущена.
Итог: На маршрутизаторе HQ-RTR настроен DHCP-сервер dnsmasq, выдающий IP-адреса в сети VLAN 200. Благодаря статической аренде по Client ID и соответствующей настройке DHCP-клиента на HQ-CLI, машина HQ-CLI гарантированно получает IP-адрес 192.168.2.10 и остальные необходимые сетевые параметры.
>>>
  Include a 1280×640 image, course title in sentence case, and a concise description in emphasis.
  In your repository settings: enable template repository, add your 1280×640 social image, auto delete head branches.
  Add your open source license, GitHub uses MIT license.
-->

# GitHub Pages

_Create a site or blog from your GitHub repositories with GitHub Pages._

</header>

<!--
  <<< Author notes: Step 1 >>>
  Choose 3-5 steps for your course.
  The first step is always the hardest, so pick something easy!
  Link to docs.github.com for further explanations.
  Encourage users to open new tabs for steps!
-->

## Step 1: Enable GitHub Pages

_Welcome to GitHub Pages and Jekyll :tada:!_

The first step is to enable GitHub Pages on this [repository](https://docs.github.com/en/get-started/quickstart/github-glossary#repository). When you enable GitHub Pages on a repository, GitHub takes the content that's on the main branch and publishes a website based on its contents.

### :keyboard: Activity: Enable GitHub Pages

1. Open a new browser tab, and work on the steps in your second tab while you read the instructions in this tab.
1. Under your repository name, click **Settings**.
1. Click **Pages** in the **Code and automation** section.
1. Ensure "Deploy from a branch" is selected from the **Source** drop-down menu, and then select `main` from the **Branch** drop-down menu.
1. Click the **Save** button.
1. Wait about _one minute_ then refresh this page (the one you're following instructions from). [GitHub Actions](https://docs.github.com/en/actions) will automatically update to the next step.
   > Turning on GitHub Pages creates a deployment of your repository. GitHub Actions may take up to a minute to respond while waiting for the deployment. Future steps will be about 20 seconds; this step is slower.
   > **Note**: In the **Pages** of **Settings**, the **Visit site** button will appear at the top. Click the button to see your GitHub Pages site.

<footer>

<!--
  <<< Author notes: Footer >>>
  Add a link to get support, GitHub status page, code of conduct, license link.
-->

---

Get help: [Post in our discussion board](https://github.com/orgs/skills/discussions/categories/github-pages) &bull; [Review the GitHub status page](https://www.githubstatus.com/)

&copy; 2023 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

</footer>
