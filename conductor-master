#!/usr/bin/env bash
set -euo pipefail

VERSION="Conductor Master v1.0.1"
WG_IF="wg-conductor"
WG_CONF="/etc/wireguard/${WG_IF}.conf"
STATE_DIR="/etc/conductor-master"
STATE_FILE="${STATE_DIR}/state.env"

ENTRY_BUNDLE_DEFAULT="/root/conductor-entry-bundle.env"
EXIT_BUNDLE_DEFAULT="/root/conductor-exit-bundle.env"
ENTRY_POLICY_SERVICE="/etc/systemd/system/conductor-entry-policy.service"
FIX_NAT_SERVICE="/etc/systemd/system/fix-amnezia-nat.service"
FIX_NAT_SCRIPT="/usr/local/sbin/fix-amnezia-nat.sh"

DEFAULT_WG_PORT="51820"
DEFAULT_ENTRY_WG_IP="10.66.66.1"
DEFAULT_EXIT_WG_IP="10.66.66.2"
DEFAULT_CLIENT_SUBNET="10.8.1.0/24"
DEFAULT_AMNEZIA_HOST_SUBNET="172.29.172.0/24"
DEFAULT_AMNEZIA_IF="amn0"
DEFAULT_AMNEZIA_PEER_IP="172.29.172.2"
DEFAULT_TABLE_ID="25182"
DEFAULT_MTU="1420"
DEFAULT_AMNEZIA_CONTAINER="amnezia-awg2"

RED='\033[0;31m'
GRN='\033[0;32m'
YLW='\033[1;33m'
BLU='\033[0;34m'
NC='\033[0m'

log()  { echo -e "${GRN}$*${NC}"; }
warn() { echo -e "${YLW}$*${NC}"; }
err()  { echo -e "${RED}$*${NC}" >&2; }
info() { echo -e "${BLU}$*${NC}"; }

require_root() {
  if [[ "${EUID}" -ne 0 ]]; then
    err "Запустите скрипт от root."
    exit 1
  fi
}

pause() {
  read -rp $'\nНажмите Enter для продолжения...'
}

mkdirs() {
  mkdir -p /etc/wireguard
  mkdir -p "${STATE_DIR}"
  chmod 700 "${STATE_DIR}"
}

backup_file() {
  local f="$1"
  if [[ -f "${f}" ]]; then
    cp -a "${f}" "${f}.bak.$(date +%Y%m%d-%H%M%S)"
  fi
}

load_state() {
  if [[ -f "${STATE_FILE}" ]]; then
    # shellcheck disable=SC1090
    source "${STATE_FILE}"
  fi
}

save_state() {
  mkdirs
  cat > "${STATE_FILE}" <<EOF
ROLE=${ROLE:-}
ENTRY_PUBLIC_IP=${ENTRY_PUBLIC_IP:-}
ENTRY_PUBLIC_IF=${ENTRY_PUBLIC_IF:-}
ENTRY_PUBLIC_KEY=${ENTRY_PUBLIC_KEY:-}
ENTRY_PRIVATE_KEY=${ENTRY_PRIVATE_KEY:-}
EXIT_PUBLIC_IP=${EXIT_PUBLIC_IP:-}
EXIT_WAN_IF=${EXIT_WAN_IF:-}
EXIT_PUBLIC_KEY=${EXIT_PUBLIC_KEY:-}
EXIT_PRIVATE_KEY=${EXIT_PRIVATE_KEY:-}
WG_PORT=${WG_PORT:-}
ENTRY_WG_IP=${ENTRY_WG_IP:-}
EXIT_WG_IP=${EXIT_WG_IP:-}
CLIENT_SUBNET=${CLIENT_SUBNET:-}
AMNEZIA_HOST_SUBNET=${AMNEZIA_HOST_SUBNET:-}
AMNEZIA_IF=${AMNEZIA_IF:-}
AMNEZIA_PEER_IP=${AMNEZIA_PEER_IP:-}
AMNEZIA_CONTAINER=${AMNEZIA_CONTAINER:-}
TABLE_ID=${TABLE_ID:-}
MTU=${MTU:-}
EOF
  chmod 600 "${STATE_FILE}"
}

command_exists() {
  command -v "$1" >/dev/null 2>&1
}

detect_pkg_manager() {
  if command_exists apt-get; then
    echo "apt"
  elif command_exists dnf; then
    echo "dnf"
  elif command_exists yum; then
    echo "yum"
  else
    echo "unknown"
  fi
}

install_packages() {
  local pm
  pm="$(detect_pkg_manager)"

  case "${pm}" in
    apt)
      export DEBIAN_FRONTEND=noninteractive
      apt-get update -y
      apt-get install -y wireguard iptables iproute2 qrencode tcpdump python3
      ;;
    dnf)
      dnf install -y wireguard-tools iptables iproute qrencode tcpdump python3
      ;;
    yum)
      yum install -y wireguard-tools iptables iproute qrencode tcpdump python3
      ;;
    *)
      err "Не удалось определить пакетный менеджер."
      exit 1
      ;;
  esac
}

ensure_wireguard_installed() {
  if command_exists wg && command_exists wg-quick; then
    log "WireGuard уже установлен."
    return
  fi
  warn "WireGuard не найден. Устанавливаю..."
  install_packages
}

get_default_if() {
  ip route get 1.1.1.1 2>/dev/null | awk '/dev/ {for(i=1;i<=NF;i++) if($i=="dev"){print $(i+1); exit}}'
}

get_public_ip() {
  ip -4 route get 1.1.1.1 2>/dev/null | awk '/src/ {for(i=1;i<=NF;i++) if($i=="src"){print $(i+1); exit}}'
}

generate_private_key() {
  wg genkey
}

derive_public_key() {
  echo "$1" | wg pubkey
}

ensure_sysctl_ip_forward() {
  sysctl -w net.ipv4.ip_forward=1 >/dev/null
  mkdir -p /etc/sysctl.d
  cat > /etc/sysctl.d/99-conductor-master.conf <<EOF
net.ipv4.ip_forward = 1
EOF
  sysctl --system >/dev/null 2>&1 || true
}

interface_exists() {
  ip link show "$1" >/dev/null 2>&1
}

get_wg_network_cidr() {
  local ip_addr="${1:-}"
  if [[ -z "${ip_addr}" ]]; then
    return 0
  fi
  echo "${ip_addr%.*}.0/24"
}

check_amnezia_peer_reachability() {
  if [[ -z "${AMNEZIA_PEER_IP:-}" ]]; then
    return 0
  fi

  if ip route get "${AMNEZIA_PEER_IP}" >/dev/null 2>&1; then
    info "Маршрут до Amnezia peer (${AMNEZIA_PEER_IP}) найден."
  else
    warn "Маршрут до Amnezia peer (${AMNEZIA_PEER_IP}) не найден."
    warn "Это не останавливает настройку, но трафик клиентов может не пойти."
  fi
}

show_finalize_summary() {
  echo
  info "Краткая сводка после финализации:"
  echo "-- ip rule --"
  ip rule show | grep -E "${CLIENT_SUBNET:-$DEFAULT_CLIENT_SUBNET}|^0:|32766:|32767:" || ip rule show || true
  echo
  echo "-- table ${TABLE_ID:-$DEFAULT_TABLE_ID} --"
  ip route show table "${TABLE_ID:-$DEFAULT_TABLE_ID}" || true
  echo
  echo "-- conductor-entry-policy.service --"
  systemctl --no-pager --full status conductor-entry-policy.service || true
  echo
  echo "-- fix-amnezia-nat.service --"
  systemctl --no-pager --full status fix-amnezia-nat.service || true
  echo
}

remove_legacy_rules() {
  local old_subnet="${1:-}"
  local wan_if="${2:-}"
  local wg_net=""

  wg_net="$(get_wg_network_cidr "${ENTRY_WG_IP:-$DEFAULT_ENTRY_WG_IP}")"

  if [[ -n "${old_subnet}" ]]; then
    iptables -t nat -D POSTROUTING -s "${old_subnet}" -o "${WG_IF}" -j RETURN 2>/dev/null || true
    ip rule del from "${old_subnet}" table "${TABLE_ID:-$DEFAULT_TABLE_ID}" 2>/dev/null || true
    ip route del default dev "${WG_IF}" table "${TABLE_ID:-$DEFAULT_TABLE_ID}" 2>/dev/null || true

    if [[ -n "${wg_net}" ]]; then
      ip route del "${wg_net}" dev "${WG_IF}" table "${TABLE_ID:-$DEFAULT_TABLE_ID}" 2>/dev/null || true
    fi
  fi

  if [[ -n "${old_subnet}" && -n "${wan_if}" ]]; then
    iptables -t nat -D POSTROUTING -s "${old_subnet}" -o "${wan_if}" -j MASQUERADE 2>/dev/null || true
  fi
}

remove_wrong_container_nat() {
  local container="${AMNEZIA_CONTAINER:-$DEFAULT_AMNEZIA_CONTAINER}"
  local subnet="${CLIENT_SUBNET:-}"

  if [[ -z "${subnet}" ]]; then
    return 0
  fi

  if ! command_exists docker; then
    warn "Docker не найден. Пропускаю проверку NAT в контейнере."
    return 0
  fi

  if ! docker ps --format '{{.Names}}' | grep -qx "${container}"; then
    warn "Контейнер ${container} не найден среди запущенных. Пропускаю проверку NAT."
    return 0
  fi

  info "Проверяю NAT внутри контейнера ${container}..."
  docker exec "${container}" sh -c "iptables -t nat -D POSTROUTING -s ${subnet} -o eth0 -j MASQUERADE" 2>/dev/null || true
  docker exec "${container}" sh -c "iptables -t nat -D POSTROUTING -s ${subnet} -o eth1 -j MASQUERADE" 2>/dev/null || true
}

write_fix_nat_script() {
  backup_file "${FIX_NAT_SCRIPT}"
  cat > "${FIX_NAT_SCRIPT}" <<EOF
#!/usr/bin/env bash
set -euo pipefail

NAME="${AMNEZIA_CONTAINER}"
SUBNET="${CLIENT_SUBNET}"

if ! command -v docker >/dev/null 2>&1; then
  exit 0
fi

for i in \$(seq 1 30); do
  if docker ps --format '{{.Names}}' | grep -qx "\${NAME}"; then
    docker exec "\${NAME}" sh -c "iptables -t nat -D POSTROUTING -s \${SUBNET} -o eth0 -j MASQUERADE" 2>/dev/null || true
    docker exec "\${NAME}" sh -c "iptables -t nat -D POSTROUTING -s \${SUBNET} -o eth1 -j MASQUERADE" 2>/dev/null || true
    exit 0
  fi
  sleep 2
done

exit 0
EOF
  chmod 700 "${FIX_NAT_SCRIPT}"
}

write_fix_nat_service() {
  write_fix_nat_script
  backup_file "${FIX_NAT_SERVICE}"
  cat > "${FIX_NAT_SERVICE}" <<EOF
[Unit]
Description=Remove wrong NAT rules inside ${AMNEZIA_CONTAINER}
After=docker.service conductor-entry-policy.service
Wants=docker.service
Requires=conductor-entry-policy.service

[Service]
Type=oneshot
ExecStart=${FIX_NAT_SCRIPT}
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

  systemctl daemon-reload
  systemctl enable fix-amnezia-nat.service >/dev/null 2>&1 || true
}

disable_fix_nat_service() {
  systemctl disable fix-amnezia-nat.service 2>/dev/null || true
  rm -f "${FIX_NAT_SERVICE}" "${FIX_NAT_SCRIPT}"
  systemctl daemon-reload || true
}

create_entry_bundle() {
  cat > "${ENTRY_BUNDLE_DEFAULT}" <<EOF
ENTRY_PUBLIC_IP=${ENTRY_PUBLIC_IP}
ENTRY_PUBLIC_KEY=${ENTRY_PUBLIC_KEY}
WG_PORT=${WG_PORT}
ENTRY_WG_IP=${ENTRY_WG_IP}
EXIT_WG_IP=${EXIT_WG_IP}
CLIENT_SUBNET=${CLIENT_SUBNET}
AMNEZIA_HOST_SUBNET=${AMNEZIA_HOST_SUBNET}
AMNEZIA_IF=${AMNEZIA_IF}
AMNEZIA_PEER_IP=${AMNEZIA_PEER_IP}
AMNEZIA_CONTAINER=${AMNEZIA_CONTAINER}
TABLE_ID=${TABLE_ID}
MTU=${MTU}
EOF
  chmod 600 "${ENTRY_BUNDLE_DEFAULT}"
}

create_exit_bundle() {
  cat > "${EXIT_BUNDLE_DEFAULT}" <<EOF
EXIT_PUBLIC_IP=${EXIT_PUBLIC_IP}
EXIT_PUBLIC_KEY=${EXIT_PUBLIC_KEY}
WG_PORT=${WG_PORT}
EXIT_WG_IP=${EXIT_WG_IP}
MTU=${MTU}
EOF
  chmod 600 "${EXIT_BUNDLE_DEFAULT}"
}

bundle_input_menu() {
  local prompt="$1"
  local default_file="$2"
  local out_file="$3"
  local file_path choice

  : > "${out_file}"

  while true; do
    echo
    echo "== ${prompt} =="
    echo "1) Использовать файл по умолчанию: ${default_file}"
    echo "2) Указать путь к файлу"
    echo "3) Вставить bundle вручную"
    echo "4) Показать файл по умолчанию"
    echo "5) Отмена"
    echo
    read -rp "Выбор [1/2/3/4/5]: " choice
    case "${choice}" in
      1)
        if [[ -f "${default_file}" ]]; then
          cp -f "${default_file}" "${out_file}"
          return 0
        else
          warn "Файл не найден: ${default_file}"
        fi
        ;;
      2)
        read -rp "Путь: " file_path
        if [[ -f "${file_path}" ]]; then
          cp -f "${file_path}" "${out_file}"
          return 0
        else
          warn "Файл не найден: ${file_path}"
        fi
        ;;
      3)
        echo "Вставьте bundle. Пустая строка завершает ввод."
        : > "${out_file}"
        while IFS= read -r line; do
          [[ -z "${line}" ]] && break
          printf '%s\n' "${line}" >> "${out_file}"
        done
        return 0
        ;;
      4)
        if [[ -f "${default_file}" ]]; then
          cat "${default_file}"
        else
          warn "Файл не найден: ${default_file}"
        fi
        ;;
      5)
        return 1
        ;;
      *)
        warn "Неверный выбор."
        ;;
    esac
  done
}

parse_bundle() {
  local file="$1"
  # shellcheck disable=SC1090
  source "${file}"
}

write_entry_wg_conf() {
  backup_file "${WG_CONF}"
  cat > "${WG_CONF}" <<EOF
[Interface]
Address = ${ENTRY_WG_IP}/24
ListenPort = ${WG_PORT}
PrivateKey = ${ENTRY_PRIVATE_KEY}
MTU = ${MTU}
Table = off

[Peer]
PublicKey = ${EXIT_PUBLIC_KEY}
Endpoint = ${EXIT_PUBLIC_IP}:${WG_PORT}
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF
  chmod 600 "${WG_CONF}"
}

write_exit_wg_conf() {
  backup_file "${WG_CONF}"
  cat > "${WG_CONF}" <<EOF
[Interface]
Address = ${EXIT_WG_IP}/24
ListenPort = ${WG_PORT}
PrivateKey = ${EXIT_PRIVATE_KEY}
MTU = ${MTU}
PostUp = iptables -t nat -C POSTROUTING -s ${CLIENT_SUBNET} -o ${EXIT_WAN_IF} -j MASQUERADE || iptables -t nat -A POSTROUTING -s ${CLIENT_SUBNET} -o ${EXIT_WAN_IF} -j MASQUERADE
PostUp = iptables -C FORWARD -i ${WG_IF} -o ${EXIT_WAN_IF} -j ACCEPT || iptables -A FORWARD -i ${WG_IF} -o ${EXIT_WAN_IF} -j ACCEPT
PostUp = iptables -C FORWARD -i ${EXIT_WAN_IF} -o ${WG_IF} -m state --state RELATED,ESTABLISHED -j ACCEPT || iptables -A FORWARD -i ${EXIT_WAN_IF} -o ${WG_IF} -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s ${CLIENT_SUBNET} -o ${EXIT_WAN_IF} -j MASQUERADE 2>/dev/null || true
PostDown = iptables -D FORWARD -i ${WG_IF} -o ${EXIT_WAN_IF} -j ACCEPT 2>/dev/null || true
PostDown = iptables -D FORWARD -i ${EXIT_WAN_IF} -o ${WG_IF} -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || true

[Peer]
PublicKey = ${ENTRY_PUBLIC_KEY}
AllowedIPs = ${ENTRY_WG_IP}/32, ${CLIENT_SUBNET}
PersistentKeepalive = 25
EOF
  chmod 600 "${WG_CONF}"
}

write_entry_policy_script() {
  local script="${STATE_DIR}/entry-policy.sh"
  cat > "${script}" <<EOF
#!/usr/bin/env bash
set -euo pipefail

WG_IF="${WG_IF}"
CLIENT_SUBNET="${CLIENT_SUBNET}"
AMNEZIA_IF="${AMNEZIA_IF}"
AMNEZIA_PEER_IP="${AMNEZIA_PEER_IP}"
EXIT_WG_IP="${EXIT_WG_IP}"
ENTRY_WG_IP="${ENTRY_WG_IP}"
TABLE_ID="${TABLE_ID}"

WG_NET="\${ENTRY_WG_IP%.*}.0/24"

ip route replace "\${CLIENT_SUBNET}" via "\${AMNEZIA_PEER_IP}" dev "\${AMNEZIA_IF}"

if ! ip rule show | grep -Eq "from[[:space:]]+\${CLIENT_SUBNET}[[:space:]].*(lookup|table)[[:space:]]+\${TABLE_ID}"; then
  ip rule add from "\${CLIENT_SUBNET}" table "\${TABLE_ID}"
fi

ip route replace "\${WG_NET}" dev "\${WG_IF}" table "\${TABLE_ID}"
ip route replace default via "\${EXIT_WG_IP}" dev "\${WG_IF}" table "\${TABLE_ID}"
EOF
  chmod 700 "${script}"
}

write_entry_policy_service() {
  write_entry_policy_script
  backup_file "${ENTRY_POLICY_SERVICE}"
  cat > "${ENTRY_POLICY_SERVICE}" <<EOF
[Unit]
Description=Conductor ENTRY policy routing
After=network-online.target wg-quick@${WG_IF}.service docker.service
Wants=network-online.target
Requires=wg-quick@${WG_IF}.service

[Service]
Type=oneshot
ExecStart=${STATE_DIR}/entry-policy.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

  systemctl daemon-reload
  systemctl enable conductor-entry-policy.service >/dev/null 2>&1 || true
}

disable_entry_policy_service() {
  systemctl disable conductor-entry-policy.service 2>/dev/null || true
  rm -f "${ENTRY_POLICY_SERVICE}" "${STATE_DIR}/entry-policy.sh"
  systemctl daemon-reload || true
}

systemd_enable_restart_wg() {
  systemctl daemon-reload
  systemctl enable "wg-quick@${WG_IF}" >/dev/null 2>&1 || true
  if ip link show "${WG_IF}" >/dev/null 2>&1; then
    wg-quick down "${WG_IF}" || true
  fi
  systemctl restart "wg-quick@${WG_IF}"
}

probe_udp() {
  local host="$1"
  local port="$2"
  timeout 2 bash -c "echo test >/dev/udp/${host}/${port}" 2>/dev/null || true
}

wait_handshake() {
  local pubkey="$1"
  local tries=20
  local hs
  while (( tries > 0 )); do
    hs="$(wg show "${WG_IF}" latest-handshakes 2>/dev/null | awk -v k="$pubkey" '$1==k {print $2}')"
    if [[ -n "${hs}" && "${hs}" != "0" ]]; then
      return 0
    fi
    sleep 1
    tries=$((tries - 1))
  done
  return 1
}

show_main_menu() {
  clear || true
  echo "===================================="
  echo " ${VERSION}"
  echo "===================================="
  echo "1) Подготовка ENTRY"
  echo "2) Настройка EXIT"
  echo "3) Завершение ENTRY"
  echo "4) Статус"
  echo "5) Самопроверка"
  echo "6) Очистка"
  echo "7) Полная очистка"
  echo "0) Выход"
  echo
}

entry_prepare() {
  ensure_wireguard_installed
  mkdirs
  load_state

  echo "== Подготовка ENTRY =="

  local def_ip def_if
  def_ip="${ENTRY_PUBLIC_IP:-$(get_public_ip)}"
  def_if="${ENTRY_PUBLIC_IF:-$(get_default_if)}"

  read -rp "Публичный IP ENTRY [${def_ip}]: " ENTRY_PUBLIC_IP_IN
  ENTRY_PUBLIC_IP="${ENTRY_PUBLIC_IP_IN:-$def_ip}"

  read -rp "Публичный интерфейс ENTRY [${def_if}]: " ENTRY_PUBLIC_IF_IN
  ENTRY_PUBLIC_IF="${ENTRY_PUBLIC_IF_IN:-$def_if}"

  read -rp "Порт WireGuard [${WG_PORT:-$DEFAULT_WG_PORT}]: " WG_PORT_IN
  WG_PORT="${WG_PORT_IN:-${WG_PORT:-$DEFAULT_WG_PORT}}"

  read -rp "IP ENTRY в туннеле [${ENTRY_WG_IP:-$DEFAULT_ENTRY_WG_IP}]: " ENTRY_WG_IP_IN
  ENTRY_WG_IP="${ENTRY_WG_IP_IN:-${ENTRY_WG_IP:-$DEFAULT_ENTRY_WG_IP}}"

  read -rp "IP EXIT в туннеле [${EXIT_WG_IP:-$DEFAULT_EXIT_WG_IP}]: " EXIT_WG_IP_IN
  EXIT_WG_IP="${EXIT_WG_IP_IN:-${EXIT_WG_IP:-$DEFAULT_EXIT_WG_IP}}"

  read -rp "Клиентская подсеть VPN [${CLIENT_SUBNET:-$DEFAULT_CLIENT_SUBNET}]: " CLIENT_SUBNET_IN
  CLIENT_SUBNET="${CLIENT_SUBNET_IN:-${CLIENT_SUBNET:-$DEFAULT_CLIENT_SUBNET}}"

  read -rp "Host subnet Amnezia [${AMNEZIA_HOST_SUBNET:-$DEFAULT_AMNEZIA_HOST_SUBNET}]: " AMNEZIA_HOST_SUBNET_IN
  AMNEZIA_HOST_SUBNET="${AMNEZIA_HOST_SUBNET_IN:-${AMNEZIA_HOST_SUBNET:-$DEFAULT_AMNEZIA_HOST_SUBNET}}"

  read -rp "Интерфейс Amnezia [${AMNEZIA_IF:-$DEFAULT_AMNEZIA_IF}]: " AMNEZIA_IF_IN
  AMNEZIA_IF="${AMNEZIA_IF_IN:-${AMNEZIA_IF:-$DEFAULT_AMNEZIA_IF}}"

  read -rp "IP контейнера/peer Amnezia на bridge [${AMNEZIA_PEER_IP:-$DEFAULT_AMNEZIA_PEER_IP}]: " AMNEZIA_PEER_IP_IN
  AMNEZIA_PEER_IP="${AMNEZIA_PEER_IP_IN:-${AMNEZIA_PEER_IP:-$DEFAULT_AMNEZIA_PEER_IP}}"

  read -rp "Имя контейнера Amnezia [${AMNEZIA_CONTAINER:-$DEFAULT_AMNEZIA_CONTAINER}]: " AMNEZIA_CONTAINER_IN
  AMNEZIA_CONTAINER="${AMNEZIA_CONTAINER_IN:-${AMNEZIA_CONTAINER:-$DEFAULT_AMNEZIA_CONTAINER}}"

  read -rp "ID таблицы маршрутизации [${TABLE_ID:-$DEFAULT_TABLE_ID}]: " TABLE_ID_IN
  TABLE_ID="${TABLE_ID_IN:-${TABLE_ID:-$DEFAULT_TABLE_ID}}"

  read -rp "MTU WireGuard [${MTU:-$DEFAULT_MTU}]: " MTU_IN
  MTU="${MTU_IN:-${MTU:-$DEFAULT_MTU}}"

  if ! interface_exists "${AMNEZIA_IF}"; then
    warn "Интерфейс ${AMNEZIA_IF} сейчас не найден."
    warn "На финализации он должен существовать."
  fi

  if [[ -z "${ENTRY_PRIVATE_KEY:-}" ]]; then
    ENTRY_PRIVATE_KEY="$(generate_private_key)"
    ENTRY_PUBLIC_KEY="$(derive_public_key "${ENTRY_PRIVATE_KEY}")"
    log "Сгенерирован новый ключ ENTRY."
  else
    ENTRY_PUBLIC_KEY="$(derive_public_key "${ENTRY_PRIVATE_KEY}")"
    warn "Использую существующий ключ ENTRY."
  fi

  ROLE="ENTRY"
  save_state
  create_entry_bundle

  log "Подготовка ENTRY завершена."
  echo "Bundle создан: ${ENTRY_BUNDLE_DEFAULT}"
  echo
  cat "${ENTRY_BUNDLE_DEFAULT}"
  echo
}

exit_setup() {
  ensure_wireguard_installed
  mkdirs
  load_state
  ensure_sysctl_ip_forward

  echo "== Настройка EXIT =="

  local bundle_file
  bundle_file="$(mktemp)"
  bundle_input_menu "Bundle от ENTRY" "${ENTRY_BUNDLE_DEFAULT}" "${bundle_file}" || {
    rm -f "${bundle_file}"
    return 0
  }
  parse_bundle "${bundle_file}"
  rm -f "${bundle_file}"

  local def_exit_ip def_wan_if
  def_exit_ip="${EXIT_PUBLIC_IP:-$(get_public_ip)}"
  def_wan_if="${EXIT_WAN_IF:-$(get_default_if)}"

  read -rp "Публичный IP EXIT [${def_exit_ip}]: " EXIT_PUBLIC_IP_IN
  EXIT_PUBLIC_IP="${EXIT_PUBLIC_IP_IN:-$def_exit_ip}"

  read -rp "WAN интерфейс EXIT [${EXIT_WAN_IF:-$def_wan_if}]: " EXIT_WAN_IF_IN
  EXIT_WAN_IF="${EXIT_WAN_IF_IN:-${EXIT_WAN_IF:-$def_wan_if}}"

  read -rp "MTU WireGuard [${MTU:-$DEFAULT_MTU}]: " MTU_IN
  MTU="${MTU_IN:-${MTU:-$DEFAULT_MTU}}"

  if ! interface_exists "${EXIT_WAN_IF}"; then
    err "WAN интерфейс ${EXIT_WAN_IF} не найден."
    return 1
  fi

  if [[ -z "${EXIT_PRIVATE_KEY:-}" ]]; then
    EXIT_PRIVATE_KEY="$(generate_private_key)"
    EXIT_PUBLIC_KEY="$(derive_public_key "${EXIT_PRIVATE_KEY}")"
    log "Сгенерирован новый ключ EXIT."
  else
    EXIT_PUBLIC_KEY="$(derive_public_key "${EXIT_PRIVATE_KEY}")"
    warn "Использую существующий ключ EXIT."
  fi

  ROLE="EXIT"
  save_state
  remove_legacy_rules "${AMNEZIA_HOST_SUBNET:-}" "${EXIT_WAN_IF:-}"
  write_exit_wg_conf
  systemd_enable_restart_wg
  create_exit_bundle

  log "EXIT настроен."
  echo "Bundle создан: ${EXIT_BUNDLE_DEFAULT}"
  echo
  cat "${EXIT_BUNDLE_DEFAULT}"
  echo
}

entry_finalize() {
  ensure_wireguard_installed
  mkdirs
  load_state

  echo "== Завершение настройки ENTRY =="

  if [[ -z "${ENTRY_PRIVATE_KEY:-}" || -z "${ENTRY_PUBLIC_KEY:-}" || -z "${CLIENT_SUBNET:-}" || -z "${AMNEZIA_IF:-}" || -z "${AMNEZIA_PEER_IP:-}" ]]; then
    err "Сначала выполните 'Подготовка ENTRY'."
    return 1
  fi

  local bundle_file
  bundle_file="$(mktemp)"
  bundle_input_menu "Bundle от EXIT" "${EXIT_BUNDLE_DEFAULT}" "${bundle_file}" || {
    rm -f "${bundle_file}"
    return 0
  }
  parse_bundle "${bundle_file}"
  rm -f "${bundle_file}"

  if ! interface_exists "${AMNEZIA_IF}"; then
    err "Интерфейс ${AMNEZIA_IF} не найден."
    return 1
  fi

  check_amnezia_peer_reachability

  remove_legacy_rules "${AMNEZIA_HOST_SUBNET:-}" ""
  write_entry_wg_conf
  systemd_enable_restart_wg
  write_entry_policy_service
  systemctl restart conductor-entry-policy.service || true
  write_fix_nat_service
  systemctl start fix-amnezia-nat.service || true
  remove_wrong_container_nat
  probe_udp "${EXIT_PUBLIC_IP}" "${WG_PORT}"

  if wait_handshake "${EXIT_PUBLIC_KEY}"; then
    log "Handshake с EXIT есть."
  else
    warn "Handshake пока не обнаружен."
  fi

  echo
  echo "Трафик клиентов ${CLIENT_SUBNET} будет идти через ${WG_IF}."
  echo "Маршрут к клиентской сети: via ${AMNEZIA_PEER_IP} dev ${AMNEZIA_IF}"
  echo "Policy table: ${TABLE_ID}"

  show_finalize_summary
}

status_show() {
  echo "== Статус =="

  if [[ -f "${STATE_FILE}" ]]; then
    echo "-- STATE --"
    cat "${STATE_FILE}"
    echo
  fi

  echo "-- wg --"
  wg show || true
  echo

  echo "-- ip rule --"
  ip rule show || true
  echo

  echo "-- main routes --"
  ip route show || true
  echo

  echo "-- table ${TABLE_ID:-$DEFAULT_TABLE_ID} --"
  ip route show table "${TABLE_ID:-$DEFAULT_TABLE_ID}" || true
  echo

  echo "-- nat --"
  iptables -t nat -S || true
  echo

  echo "-- filter --"
  iptables -S || true
  echo

  echo "-- systemd wg --"
  systemctl status "wg-quick@${WG_IF}" --no-pager || true
  echo

  if systemctl list-unit-files | grep -q '^conductor-entry-policy.service'; then
    echo "-- systemd entry policy --"
    systemctl status conductor-entry-policy.service --no-pager || true
    echo
  fi

  if systemctl list-unit-files | grep -q '^fix-amnezia-nat.service'; then
    echo "-- systemd fix nat --"
    systemctl status fix-amnezia-nat.service --no-pager || true
    echo
  fi
}

self_test() {
  load_state
  echo "== Самопроверка =="

  echo "[1] Интерфейсы"
  ip -br addr
  echo

  echo "[2] WireGuard"
  wg show || true
  echo

  echo "[3] ip rule"
  ip rule show
  echo

  echo "[4] main routes"
  ip route show
  echo

  echo "[5] table ${TABLE_ID:-$DEFAULT_TABLE_ID}"
  ip route show table "${TABLE_ID:-$DEFAULT_TABLE_ID}" || true
  echo

  echo "[6] iptables nat"
  iptables -t nat -S
  echo

  echo "[7] iptables filter"
  iptables -S
  echo

  if [[ -n "${CLIENT_SUBNET:-}" ]]; then
    echo "[8] route get from client subnet"
    local first_ip
    first_ip="$(python3 - "${CLIENT_SUBNET}" <<'PY'
import ipaddress, sys
n=ipaddress.ip_network(sys.argv[1], strict=False)
hosts=list(n.hosts())
print(hosts[0] if hosts else "")
PY
)"
    if [[ -n "${first_ip}" ]]; then
      ip route get 8.8.8.8 from "${first_ip}" || true
    fi
    echo
  fi

  if [[ -n "${AMNEZIA_IF:-}" ]]; then
    echo "[9] Проверка интерфейса Amnezia"
    ip link show "${AMNEZIA_IF}" || true
    echo
  fi

  if [[ -n "${AMNEZIA_PEER_IP:-}" ]]; then
    echo "[10] Route to Amnezia peer"
    ip route get "${AMNEZIA_PEER_IP}" || true
    echo
  fi

  if command_exists docker && [[ -n "${AMNEZIA_CONTAINER:-}" ]]; then
    echo "[11] NAT в контейнере ${AMNEZIA_CONTAINER}"
    docker exec "${AMNEZIA_CONTAINER}" sh -c 'iptables -t nat -S' 2>/dev/null || true
    echo
  fi

  if systemctl list-unit-files | grep -q '^fix-amnezia-nat.service'; then
    echo "[12] Сервис фикса NAT"
    systemctl status fix-amnezia-nat.service --no-pager || true
    echo
  fi
}

cleanup() {
  load_state
  echo "== Очистка =="

  systemctl stop fix-amnezia-nat.service 2>/dev/null || true
  disable_fix_nat_service

  systemctl stop "wg-quick@${WG_IF}" 2>/dev/null || true
  systemctl disable "wg-quick@${WG_IF}" 2>/dev/null || true
  wg-quick down "${WG_IF}" 2>/dev/null || true

  disable_entry_policy_service

  if [[ -n "${CLIENT_SUBNET:-}" && -n "${TABLE_ID:-}" ]]; then
    ip rule del from "${CLIENT_SUBNET}" table "${TABLE_ID}" 2>/dev/null || true
    ip route flush table "${TABLE_ID}" 2>/dev/null || true
  fi

  if [[ -n "${CLIENT_SUBNET:-}" && -n "${EXIT_WAN_IF:-}" ]]; then
    iptables -t nat -D POSTROUTING -s "${CLIENT_SUBNET}" -o "${EXIT_WAN_IF}" -j MASQUERADE 2>/dev/null || true
  fi

  if [[ -n "${EXIT_WAN_IF:-}" ]]; then
    iptables -D FORWARD -i "${WG_IF}" -o "${EXIT_WAN_IF}" -j ACCEPT 2>/dev/null || true
    iptables -D FORWARD -i "${EXIT_WAN_IF}" -o "${WG_IF}" -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || true
  fi

  rm -f "${WG_CONF}"
  systemctl daemon-reload || true
  log "Очистка завершена."
}

full_cleanup() {
  cleanup
  rm -f "${STATE_FILE}" "${ENTRY_BUNDLE_DEFAULT}" "${EXIT_BUNDLE_DEFAULT}"
  rm -rf "${STATE_DIR}"
  log "Полная очистка завершена."
}

main() {
  require_root
  mkdirs

  while true; do
    show_main_menu
    read -rp "Выбор: " choice
    case "${choice}" in
      1) entry_prepare; pause ;;
      2) exit_setup; pause ;;
      3) entry_finalize; pause ;;
      4) status_show; pause ;;
      5) self_test; pause ;;
      6) cleanup; pause ;;
      7) full_cleanup; pause ;;
      0) exit 0 ;;
      *) warn "Неверный выбор."; sleep 1 ;;
    esac
  done
}

main "$@"
