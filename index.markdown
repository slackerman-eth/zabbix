---
layout: default
title: "Установка и настройка Zabbix через Docker"
---

Я установил две ВМ на Ubuntu Server 20.04 через VirtualBox. Так как все находится физически на одном ноуте в настройках сети выбрал Сетевой мост.
**zabbix-server**: тут через docker-compose устанавливается mysql-server, zabbix-server, веб морда. И нативно zabbix-agent
**test-vm**: тут контейнер с python скриптом, который генерит нагрузку на CPU и нативно zabbix-agent
Настроил триггеры на запуск, остановку контейнера docker и на нагрузку на CPU и уведомления в Telegram
Пришлось долго посидеть над docker-compose, чтобы всё заработало как надо и запускалось одной командой. Все время были какие-то проблемы, несовместимость версий, БД не создается автоматически из-за неправильной кодировки, и недостаточных прав, так же пришлось разобраться с сетевыми настройками внутри docker. Ниже будет финальный docker-compose.yaml файл.
![[Pasted image 20250418071433.png]]
### Настройка ВМ zabbixserver
```bash
sudo apt install ssh # SSH чтоб подключиться с не конченого терминала
```

```bash
sudo apt update && sudo apt upgrade -y
```

#### Установка Docker
https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

#### Установка Docker-compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo usermod -aG docker $USER
```

Создадим docker-compose.yaml
```yaml
version: '3.8'

services:
  mysql-server:
    container_name: zabbix-mysql
    image: mysql:8.0
    command: --character-set-server=utf8 --collation-server=utf8_bin --default-authentication-plugin=mysql_native_password --log_bin_trust_function_creators=1
    environment:
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_pwd
      - MYSQL_ROOT_PASSWORD=root_pwd
    volumes:
      - mysql-data:/var/lib/mysql
    restart: unless-stopped
    networks:
      - zabbix-net

  zabbix-server:
    container_name: zabbix-server
    image: zabbix/zabbix-server-mysql:ubuntu-6.0-latest
    ports:
      - "10051:10051"
    environment:
      - DB_SERVER_HOST=zabbix-mysql
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_pwd
      - MYSQL_ROOT_PASSWORD=root_pwd
    depends_on:
      - mysql-server
    restart: unless-stopped
    networks:
      zabbix-net:
        ipv4_address: 172.18.0.100
    extra_hosts:
      - "host.docker.internal:host-gateway"

  zabbix-web:
    container_name: zabbix-web
    image: zabbix/zabbix-web-nginx-mysql:ubuntu-6.0-latest
    ports:
      - "8080:8080"
    environment:
      - ZBX_SERVER_HOST=zabbix-server
      - DB_SERVER_HOST=zabbix-mysql
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_pwd
      - MYSQL_ROOT_PASSWORD=root_pwd
    depends_on:
      - zabbix-server
      - mysql-server
    restart: unless-stopped
    networks:
      - zabbix-net

networks:
  zabbix-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/16

volumes:
  mysql-data:
```
    
`--log_bin_trust_function_creators=1`: Разрешает создание хранимых функций и триггеров без SUPER-привилегий, без этой штуки БД не инициализировалась

`--character-set-server=utf8mb4`: Устанавливает кодировку сервера MySQL в utf8mb4, из документации

`--collation-server=utf8mb4_bin`: Устанавливает правило сортировки utf8mb4_bin, тоже из документации

Тут же прописываем серверу статический ip внутри docker сети 172.18.0.100

#### Установка zabbix-agent
```bash
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu20.04_all.deb
sudo dpkg -i zabbix-release_6.0-4+ubuntu20.04_all.deb 
sudo apt update
sudo apt install zabbix-agent -y
```

Конфиг zabbix-agent
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

```
Server=172.18.0.100
ServerActive=172.18.0.100
Hostname=Zabbix server
```

Запустим и включим
```bash
sudo systemctl start zabbix-agent
sudo systemctl enable zabbix-agent
```

Проверим статус
```bash
sudo systemctl status zabbix-agent
```

Узнать IP адреса контейнеров в сети docker
```bash
docker network inspect zabbix-net
```

В настройках мониторинга в веб морде:
```
DNS name: host.docker.internal
Port: 10050
```

В docker-compose.yaml прописано extra_hosts, чтобы Zabbix сервер мог обращаться к хосту по имени host.docker.internal


### Запуск всех сервисов

1. Запускаем контейнеры Docker: В директории с docker-compose.yaml:
```bash
docker-compose up -d
```
Это поднимет MySQL, Zabbix сервер и веб-интерфейс.

![[Pasted image 20250412012449.png]]
`docker ps` проверяем статус контейнеров (всё запущено)


2. Запускаем Zabbix агент: На хосте включаем и запускаем службу агента:
```bash
sudo systemctl enable zabbix-agent
sudo systemctl start zabbix-agent
```
    
3. Проверка статуса
```bash
sudo systemctl status zabbix-agent
```    
![[Pasted image 20250412012645.png]]
Агент запущен

![[Pasted image 20250412014017.png]]
Хост появился в веб-интерфейсе и горит зеленым


### Настройка ВМ test-vm
```bash
sudo apt install ssh # устанавливаем SSH
```

```bash
sudo apt update && sudo apt upgrade -y
```

Установка Docker:
```bash
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```


#### Создадим простой скрипт, который будет нагружать CPU вычислениями.
Создадим файл cpu_load.py:
```python
import time

def cpu_load():
	while True:
		x = 1.0001
		for _ in range(1000000):
			x = x * 1.0001
		time.sleep(0.1)  # Пауза, чтобы не перегружать систему полностью

if __name__ == "__main__":
	cpu_load()
```
Этот скрипт выполняет бесконечные вычисления с небольшой паузой, создавая нагрузку на CPU.

---

#### Запуск скрипта в Docker-контейнере

1. Создадим Dockerfile в той же директории, где лежит cpu_load.py:
```Dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY cpu_load.py .
CMD ["python", "cpu_load.py"]
```
    
2. Собираем Docker-образ:
```bash
docker build -t cpu-load-image .
```
    
3. Запускаем контейнер:
```bash
docker run -d --name cpu-load-container cpu-load-image
```
Флаг -d запускает контейнер в фоновом режиме, а --name задаёт ему имя.

![[Pasted image 20250412030322.png]]
Контейнер создан и запущен. Через `htop`Видим возросшую нагрузку на CPU
![[Pasted image 20250412030416.png]]
Стоп, старт контейнера
```bash
docker stop cpu-load-container
docker start cpu-load-container
```

#### Установка zabbix-agent
```bash
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu20.04_all.deb
sudo dpkg -i zabbix-release_6.0-4+ubuntu20.04_all.deb 
sudo apt update
sudo apt install zabbix-agent -y
```

Настроем конфиг: 
Файл /etc/zabbix/zabbix_agentd.conf:
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

```
Server=192.168.0.109  # IP Zabbix-сервера (не конейнера,а машины) (hostname -I)
Hostname=test-vm    # Уникальное имя для этой ВМ
```
Примечание: Не используем 172.18.0.100 (IP контейнера), так как он недоступен вне Docker-сети zabbix-net.

Запустим агент
```bash
sudo systemctl enable zabbix-agent # добавить в автозагрузку
sudo systemctl start zabbix-agent
```

Добавим этот сервер в админке:
![[Pasted image 20250417152952.png]]
![[Pasted image 20250417153038.png]]

#### Запустим контейнер с Python скриптом:
```bash
docker start cpu-load-container
```

На графике видим возросшую нагрузку на CPU:
![[Pasted image 20250417153824.png]]

#### Добавим триггер на test-vm. Будет срабатывать при >=5% CPU utilization
![[Pasted image 20250417161027.png]]

Ещё раз запустим cpu-load-container и дождемся срабатывания триггера:
![[Pasted image 20250417161329.png]]

Остановим cpu-load-container и дождемся когда триггер зарезолвится.
![[Pasted image 20250417161727.png]]



Теперь замониторим статус контейнера cpu-load-container и настроим триггеры на запуск или остановку контейнера
Для мониторинга Docker-контейнера Zabbix агенту нужно иметь доступ к Docker API. Буду использовать команду `docker inspect` через пользовательский параметр в конфигурации агента.

Добавил пользователя zabbix в группу docker
```bash
sudo usermod -aG docker zabbix
```

В конфиг добавляем пользовательский параметр
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

```
UserParameter=docker.container_status[*],docker inspect --format '{{.State.Status}}' $1
```
Этот параметр позволяет Zabbix запрашивать статус контейнера (например, running, exited) через команду `docker inspect`

Проверим:
```bash
zabbix_agentd -t docker.container_status[cpu-load-container]
```
![[Pasted image 20250418062451.png]]
Всё ок, данные поступают


### Создадим элемент данных
![[Pasted image 20250418062749.png]]

И два триггера на = "running" и <> "running"
![[Pasted image 20250418062939.png]]
![[Pasted image 20250418062858.png]]

Остановим контейнер cpu-load-container
![[Pasted image 20250418063339.png]]

Запустим контейнер cpu-load-container, после запуска так же сработал триггер на нагрузку на CPU
![[Pasted image 20250418063802.png]]


Так же настроил оповещения в Telegram через Media внутри Zabbix
![[IMG_CD69E8F549E2-1.jpeg]]
