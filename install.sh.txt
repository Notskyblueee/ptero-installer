#!/usr/bin/env bash


# ===============================
# notsky08 • Pterodactyl Installer
# IPv6 ONLY VPS
# ===============================

BLUE="\e[34m"
CYAN="\e[36m"
GREEN="\e[32m"
RED="\e[31m"
YELLOW="\e[33m"
NC="\e[0m"

clear

banner() {
echo -e "${CYAN}"
cat << "EOF"
███╗   ██╗ ██████╗ ████████╗███████╗██╗  ██╗██╗   ██╗ ██████╗  █████╗ 
████╗  ██║██╔═══██╗╚══██╔══╝██╔════╝██║ ██╔╝╚██╗ ██╔╝██╔═████╗██╔══██╗
██╔██╗ ██║██║   ██║   ██║   ███████╗█████╔╝  ╚████╔╝ ██║██╔██║╚█████╔╝
██║╚██╗██║██║   ██║   ██║   ╚════██║██╔═██╗   ╚██╔╝  ████╔╝██║██╔══██╗
██║ ╚████║╚██████╔╝   ██║   ███████║██║  ██╗   ██║   ╚██████╔╝╚█████╔╝
╚═╝  ╚═══╝ ╚═════╝    ╚═╝   ╚══════╝╚═╝  ╚═╝   ╚═╝    ╚═════╝  ╚════╝ 

        IPv6 ONLY • PTERODACTYL INSTALLER
EOF
echo -e "${NC}"
}

pause() {
echo
read -rp "Press ENTER to continue..."
}

install_dependencies() {
apt update -y
apt install -y curl wget sudo git tar unzip nginx mariadb-server \
php php-cli php-mysql php-mbstring php-curl php-zip php-xml php-bcmath redis-server
}

panel_install() {
clear
banner
echo -e "${GREEN}▶ Installing Pterodactyl Panel (IPv6)...${NC}"

install_dependencies

curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt install -y nodejs composer

mkdir -p /var/www/pterodactyl
cd /var/www/pterodactyl || exit

curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
tar -xzf panel.tar.gz

chmod -R 755 storage/* bootstrap/cache/
cp .env.example .env

composer install --no-dev --optimize-autoloader
php artisan key:generate --force

echo -e "${YELLOW}Now follow panel environment setup manually${NC}"
php artisan p:environment:setup
php artisan p:environment:database
php artisan migrate --seed --force
php artisan p:user:make

chown -R www-data:www-data /var/www/pterodactyl

echo -e "${GREEN}✔ Panel installation completed${NC}"
pause
}

wings_install() {
clear
banner
echo -e "${GREEN}▶ Installing Wings (IPv6)...${NC}"

curl -sSL https://get.docker.com | sh
systemctl enable --now docker

mkdir -p /etc/pterodactyl
curl -Lo /usr/local/bin/wings https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_amd64
chmod +x /usr/local/bin/wings

cat <<EOF >/etc/systemd/system/wings.service
[Unit]
Description=Pterodactyl Wings (IPv6)
After=docker.service
Requires=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
ExecStart=/usr/local/bin/wings
Restart=on-failure
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reexec
systemctl enable wings

echo -e "${GREEN}✔ Wings installed (configure node from panel)${NC}"
pause
}

panel_update() {
clear
banner
cd /var/www/pterodactyl || exit

php artisan down
curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
tar -xzf panel.tar.gz

composer install --no-dev --optimize-autoloader
php artisan migrate --seed --force
php artisan up

echo -e "${GREEN}✔ Panel updated${NC}"
pause
}

system_info() {
clear
banner
echo -e "${CYAN}System Information${NC}"
echo "---------------------"
hostnamectl
echo
ip -6 addr
echo
df -h
pause
}

uninstall_tools() {
clear
banner
echo -e "${RED}This will NOT remove panel data.${NC}"
apt remove -y nginx mariadb-server docker docker.io
echo -e "${GREEN}✔ Tools removed${NC}"
pause
}

main_menu() {
while true; do
clear
banner
echo -e "${BLUE}MAIN MENU${NC}"
echo "--------------------------------"
echo "1) Panel Installation"
echo "2) Wings Installation"
echo "3) Panel Update"
echo "4) Uninstall Tools"
echo "5) System Information"
echo "0) Exit"
echo "--------------------------------"
read -rp "Select an option [0-5]: " choice

case $choice in
1) panel_install ;;
2) wings_install ;;
3) panel_update ;;
4) uninstall_tools ;;
5) system_info ;;
0) exit 0 ;;
*) echo -e "${RED}Invalid option${NC}"; sleep 1 ;;
esac
done
}

main_menu
