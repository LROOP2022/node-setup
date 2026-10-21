# hack-node-setup

Ansible для кластера хакатона: k3s (1 server + 2 agents), ingress-nginx, cert-manager,
Sealed Secrets, Argo CD. Запускается с рабочей машины, не cron-pull.

## Что делает плейбук

| Этап | Хосты | Результат |
|---|---|---|
| `common` | все | пакеты, chrony, проверки (публичный IP на интерфейсе, модуль wireguard) |
| `k3s` | node-1 | k3s server закреплённой версии, Traefik выключен, flannel wireguard-native, метки `role=platform` и servicelb только здесь; admin kubeconfig скачивается в `artifacts/` |
| `k3s` | node-2, node-3 | k3s agents, метки `role=data` и `role=apps` |
| `ingress`, `cert-manager`, `sealed-secrets`, `argocd` | node-1 (через kubectl) | компоненты закреплены на node-1; ClusterIssuer `letsencrypt-staging` и `letsencrypt-prod`; публичный ключ Sealed Secrets скачивается в `artifacts/` |
| `root` | node-1 | **только по явному запросу**: root Application из `hack-manifests` |

## Требования на рабочей машине

- `ansible-core` (проверено на 2.21), доступ по ключу `~/.ssh/hack_ed25519` ко всем трём нодам
- Ноды на Debian/Ubuntu, публичный IP назначен прямо на интерфейс (так в `ip a`)

## Первый запуск

```sh
# 1. Секреты (токен кластера и email для Let's Encrypt), пример в vault.yml.example
openssl rand -hex 32
ansible-vault create inventories/hack/group_vars/all/vault.yml

# 2. Проверка без изменений на нодах
ansible-playbook playbooks/site.yml --ask-vault-pass --syntax-check
ansible all -m ping --ask-vault-pass

# 3. Установка
ansible-playbook playbooks/site.yml --ask-vault-pass
```

Повторный запуск безопасен. Отдельные этапы: `--tags common|k3s|ingress|cert-manager|sealed-secrets|argocd`.

## После установки

```sh
export KUBECONFIG=$PWD/artifacts/kubeconfig-hack.yaml
kubectl get nodes -L role                      # три ноды Ready, метки platform/data/apps
kubectl get pods -A -o wide                    # ingress, cert-manager, argocd, sealed-secrets на node-1
kubectl get clusterissuer                      # staging и prod Ready
```

- `artifacts/sealed-secrets.pub.pem` — публичный ключ этого кластера, копируется в `hack-manifests` (`make pubkey-update`).
- Argo CD (доступ только через port-forward):

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443    # https://localhost:8080, логин admin
```

## Подключение GitOps

После того как в `hack-manifests` (ветка `main`) закоммичены `root-application.yml` и каталог `applications/`:

```sh
ansible-playbook playbooks/site.yml --ask-vault-pass --tags root
```

## Пересоздание кластера

Ключ Sealed Secrets у каждого кластера свой. После пересоздания снова выгружается `artifacts/sealed-secrets.pub.pem`,
и все секреты в `hack-manifests` нужно запечатать заново.

## Известные ограничения (техдолг)

- **Нет firewall.** Порты 6443 (API) и 10250 (kubelet) открыты в интернет, защита только токеном и сертификатами. Закрыть, как только появится постоянный IP или VPN.
- SSH под root, вход по ключу, парольный вход не отключался: hardening отложен.
- ingress-nginx официально выведен из поддержки (март 2026), закреплён последний выпуск v1.15.1.
- Установка компонентов по схеме «если нет»: смена версии в `vars.yml` на живом кластере ничего не обновит.
- Компоненты применяются через `kubectl apply` по URL из интернета.
