# Monitoring-Stack
Monitoring Stack Installation using Docker Compose
This guide helps you set up a complete monitoring stack using Docker and Docker Compose. The stack includes:
* ##### Prometheus : Monitoring and alerting toolkit.
* ##### SNMP Exporter: Prometheus exporter for SNMP-enabled devices.
* ##### Grafana: Dashboarding and visualization tool.
* ##### Blackbox Exporter: Prometheus exporter for probing endpoints over HTTP, HTTPS, DNS, TCP, ICMP, etc.
* ##### Zabbix: Enterprise-grade monitoring platform.
* ##### PostgreSQL: Database for Zabbix server.

## Prerequisites
1- Docker: Make sure you have Docker installed. You can install Docker by following the official instructions [here](https://docs.docker.com/engine/install/ubuntu/).

2- Docker Compose: Make sure you have Docker Compose installed. You can install it by following the official instructions [here](https://docs.docker.com/compose/install/).

## Directory Structure
Before running the Docker Compose commands, create the following directory structure and files on your host machine:

```
├── docker-compose.yml                  # For Prometheus, Grafana, SNMP Exporter, Blackbox Exporter
├── zabbix-docker-compose.yml            # For Zabbix and PostgreSQL
├── config
│   └── prometheus
│       └── prometheus.yml               # Prometheus configuration file
├── blackboxexporter
│   └── config.yml                       # Blackbox Exporter configuration file
├── postgres-data                        # Directory for PostgreSQL data persistence
└── zabbix_server.conf                   # Zabbix server configuration file
```

1- docker-compose.yml (**Monitoring Stack**)
This file contains the services for **Prometheus**, **SNMP Exporter**, **Grafana**, and **Blackbox Exporter**.
‍
```
version: '3'

volumes:
  prometheus_data:
    driver: local
  grafana_data:
    driver: local
  db_data:
    driver: local

networks:
  monitoring:

services:
  prometheus:
    container_name: prometheus
    image: prom/prometheus
    restart: unless-stopped
    volumes:
      - 'prometheus_data:/prometheus'
      - './config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml'
    networks:
      - monitoring 
    ports:
      - 9090:9090

  snmp-exporter:
    container_name: snmp-exporter
    image: prom/snmp-exporter
    restart: unless-stopped
    networks:
      - monitoring
    environment:
      - SNMP_EXPORTER_TARGETS=192.168.200.253:161  # Replace with your switch IP
      - SNMP_EXPORTER_COMMUNITY=private
    ports:
      - 9116:9116

  grafana:
    container_name: grafana
    image: grafana/grafana
    restart: unless-stopped
    volumes:
      - 'grafana_data:/var/lib/grafana'
    networks:
      - monitoring
    ports:
      - 80:3000

  blackbox_exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox_exporter
    networks:
      - monitoring
    ports:
      - 9115:9115
    volumes:
      - ./blackboxexporter/:/etc/blackboxexporter/
    command:
      - '--config.file=/etc/blackboxexporter/config.yml'
    restart: unless-stopped
```
2* zabbix-docker-compose.yml (**Zabbix Stack**)
This file contains the services for **Zabbix**, **Zabbix Web Interface**, and **PostgreSQL**.

```
version: '3'

services:
  zabbix-server:
    image: zabbix/zabbix-server-pgsql:latest
    container_name: zabbix-server
    restart: unless-stopped
    networks:
      - zabbix-net
    ports:
      - "10051:10051"
    environment:
      - DB_SERVER_HOST=db
      - POSTGRES_DB=zabbix
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=zabbix_pwd
    volumes:
      - ./zabbix_server.conf:/etc/zabbix/zabbix_server.conf
    depends_on:
      - db
      - zabbix-web

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:latest
    container_name: zabbix-web
    restart: unless-stopped
    networks:
      - zabbix-net
    ports:
      - "8080:8080"
    environment:
      - DB_SERVER_HOST=db
      - POSTGRES_DB=zabbix
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=zabbix_pwd
      - ZBX_SERVER_HOST=zabbix-server
    depends_on:
      - db

  db:
    image: postgres:alpine
    container_name: zabbix-db
    restart: unless-stopped
    networks:
      - zabbix-net
    environment:
      - POSTGRES_DB=zabbix
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=zabbix_pwd
    volumes:
      - ./postgres-data:/var/lib/postgresql/data

networks:
  zabbix-net:
```