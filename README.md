# k8s-lab

k3s-кластер из двух нод, развёрнутый Ansible-ом, с CI-проверкой конфигурации.

![CI](https://github.com/yomusfu/k8s-lab/actions/workflows/ansible-check.yml/badge.svg)

## Архитектура

    push ───► GitHub Actions (ansible --check)
                          │
        ┌─────────────────┴───────────────────┐
        ▼                                     ▼
┌──────────────────────┐        ┌──────────────────────┐
│ k8s-node1 (VPS)      │        │ k8s-node2 (VPS)      │
│ k3s server (CP)      │◄─ wg0─►│ k3s agent            │
│ zabbix-agent2        │        │ zabbix-agent2        │
└──────────────────────┘        └──────────────────────┘

## Стек

Ansible · k3s · wireguard-native · GitHub Actions · Zabbix Agent2 · Ubuntu 24.04

## Структура

    ansible/
    ├── playbook.yaml            # k3s, потом zabbix-agent
    ├── inventory/
    │   ├── hosts.ini.example    # шаблон (реальный — в .gitignore)
    │   └── host_vars/           # реальные IP и токен — вне git
    └── roles/
        ├── k3s/                 # server + agent, токен из переменной
        └── zabbix-agent/        # метрики обеих нод в Zabbix
    manifests/                   # Kubernetes-манифесты приложений
    .github/workflows/           # CI: dry-run плейбука на каждый push

## Запуск

    cp ansible/inventory/hosts.ini.example ansible/inventory/hosts.ini
    # заполнить IP и создать host_vars/secrets.yml

    ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yaml --check
    ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yaml

    # проверка на control-plane ноде:
    k3s kubectl get nodes

## Грабли и решения

- **Токен k3s**: первая версия читала node-token через slurp — на чистом
  окружении файла ещё нет, и агент не получает токен. Заменил на явный токен
  из переменной: кластер поднимается с чистого листа за один прогон.
- **Конфликт портов**: Zabbix-web уже слушал NodePort 30080 на node1 —
  NodePort приложения смещён на 30081.
- **Пустой kubeconfig**: kubectl падал с `current-context must exist` —
  файл был пуст. Пересоздан через scp с ноды + sed на внешний IP.

## Roadmap

- [x] Кластер k3s из двух нод через Ansible
- [x] CI: dry-run на каждый push
- [x] Zabbix-агент на обеих нодах
- [ ] Деплой первого приложения (Deployment + Service)
- [ ] GitOps: apply из пайплайна
