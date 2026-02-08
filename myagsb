#!/bin/bash
export LANG=en_US.UTF-8

RED="\033[31m"
GREEN="\033[32m"
YELLOW="\033[33m"
PLAIN="\033[0m"

# 项目名称
PROJECT="myagsb"
WORK_DIR="/opt/${PROJECT}"
UUID=$(cat /proc/sys/kernel/random/uuid)
PORT=8080
ARGO_TOKEN=""
ARGO_DOMAIN=""
CDN_HOST=""

[[ $EUID -ne 0 ]] && echo -e "${RED}Error: Must be root.${PLAIN}" && exit 1

clear
echo -e "${GREEN}>>> MyAgSb Installer (VMess + VLESS Only)${PLAIN}"

if [[ $# -eq 0 ]]; then
    # 交互模式
    read -p "1. Argo Token (留空则使用临时隧道): " ARGO_TOKEN
    if [[ -n "$ARGO_TOKEN" ]]; then
        read -p "2. Argo Domain (e.g. arg.site.com): " ARGO_DOMAIN
        [[ -z "$ARGO_DOMAIN" ]] && echo -e "${RED}Domain required for Token mode.${PLAIN}" && exit 1
    fi
    read -p "3. 优选域名/CDN (e.g. www.visa.com): " CDN_HOST
    read -p "4. 本地端口 (Default 8080): " input_port
    [[ -n "$input_port" ]] && PORT=$input_port
else
    # 参数模式
    while [[ "$#" -gt 0 ]]; do
        case $1 in
            --token) ARGO_TOKEN="$2"; shift ;;
            --domain) ARGO_DOMAIN="$2"; shift ;;
            --port) PORT="$2"; shift ;;
            --cdn) CDN_HOST="$2"; shift ;;
            --uuid) UUID="$2"; shift ;;
        esac
        shift
    done
fi

echo -e "${YELLOW}Installing environment...${PLAIN}"

# 安装 Node.js 和 PM2
if ! command -v node &> /dev/null; then
    curl -fsSL https://deb.nodesource.com/setup_20.x | bash - >/dev/null 2>&1
    apt-get install -y nodejs >/dev/null 2>&1
fi
if ! command -v pm2 &> /dev/null; then
    npm install -g pm2 >/dev/null 2>&1
fi

mkdir -p "$WORK_DIR"
cd "$WORK_DIR"

# 生成 package.json
echo '{"name":"myagsb","main":"index.js","dependencies":{"adm-zip":"^0.5.10"}}' > package.json

# 生成核心代码 index.js
cat > index.js << 'EOF'
const fs = require('fs');
const path = require('path');
const { spawn } = require('child_process');
const https = require('https');
const AdmZip = require('adm-zip');

const C = {
    UUID: process.env.UUID,
    PORT: parseInt(process.env.PORT) || 8080,
    TOKEN: process.env.ARGO_TOKEN || '',
    DOMAIN: process.env.ARGO_DOMAIN || '',
    CDN: process.env.CDN_HOST || '',
    V_PORT: 10001
};

const BIN = path.join(__dirname, 'bin');
const XRAY = path.join(BIN, 'xray');
const CF = path.join(BIN, 'cloudflared');
const CONF = path.join(__dirname, 'config.json');

async function dl(url, dest) {
    return new Promise((resolve, reject) => {
        const file = fs.createWriteStream(dest);
        https.get(url, (res) => {
            if (res.statusCode > 300 && res.statusCode < 400) return dl(res.headers.location, dest).then(resolve).catch(reject);
            res.pipe(file);
            file.on('finish', () => file.close(resolve));
        }).on('error', (err) => fs.unlink(dest, () => reject(err)));
    });
}

async function init() {
    if (!fs.existsSync(BIN)) fs.mkdirSync(BIN);
    if (!fs.existsSync(XRAY)) {
        const zip = path.join(BIN, 'x.zip');
        await dl("https://github.com/XTLS/Xray-core/releases/latest/download/Xray-linux-64.zip", zip);
        new AdmZip(zip).extractAllTo(BIN, true);
        fs.chmodSync(XRAY, '755');
        fs.unlinkSync(zip);
    }
    if (!fs.existsSync(CF)) {
        await dl("https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64", CF);
        fs.chmodSync(CF, '755');
    }
}

function genConf() {
    const c = {
        log: { loglevel: "none" },
        inbounds: [
            {
                port: C.PORT, protocol: "vless",
                settings: { 
                    clients: [{ id: C.UUID }], 
                    decryption: "none", 
                    fallbacks: [{ path: "/vm", dest: C.V_PORT }] 
                },
                streamSettings: { network: "ws", wsSettings: { path: "/vl" } }
            },
            {
                port: C.V_PORT, listen: "127.0.0.1", protocol: "vmess",
                settings: { clients: [{ id: C.UUID }] },
                streamSettings: { network: "ws", wsSettings: { path: "/vm" } }
            }
        ],
        outbounds: [{ protocol: "freedom" }]
    };
    fs.writeFileSync(CONF, JSON.stringify(c));
}

function show(d) {
    const h = C.CDN || d;
    const sni = d;
    console.log(`\n=== MyAgSb LIST ===\nUUID: ${C.UUID}\nHost: ${sni}\nAddr: ${h}`);
    
    // VLESS Link
    console.log(`\n[VLESS-Argo] vless://${C.UUID}@${h}:443?encryption=none&security=tls&type=ws&host=${sni}&path=%2Fvl#Argo-VLESS`);
    
    // VMESS Link
    const v = { v: "2", ps: "Argo-VMess", add: h, port: "443", id: C.UUID, aid: "0", scy: "auto", net: "ws", type: "none", host: sni, path: "/vm", tls: "tls" };
    console.log(`\n[VMess-Argo] vmess://${Buffer.from(JSON.stringify(v)).toString('base64')}`);
    
    console.log(`\n=== END ===\n`);
}

async function run() {
    try {
        await init();
        genConf();
        spawn(XRAY, ['-c', CONF], { stdio: 'ignore' });

        const args = ['tunnel', '--no-autoupdate'];
        if (C.TOKEN) {
            args.push('run', '--token', C.TOKEN);
            setTimeout(() => show(C.DOMAIN), 3000);
        } else {
            args.push('--url', `http://localhost:${C.PORT}`);
            spawn(CF, args).stderr.on('data', d => {
                const m = d.toString().match(/https:\/\/[\w-]+\.trycloudflare\.com/);
                if (m) show(m[0].replace("https://", ""));
            });
        }
    } catch (e) { console.error(e); }
}
run();
EOF

echo -e "${YELLOW}Installing deps...${PLAIN}"
npm install --silent >/dev/null 2>&1

pm2 delete ${PROJECT} >/dev/null 2>&1
pm2 start index.js --name ${PROJECT} -- \
    --env UUID="$UUID" --env PORT="$PORT" \
    --env ARGO_TOKEN="$ARGO_TOKEN" --env ARGO_DOMAIN="$ARGO_DOMAIN" \
    --env CDN_HOST="$CDN_HOST" >/dev/null 2>&1

pm2 save >/dev/null 2>&1
pm2 startup | grep "sudo" | bash >/dev/null 2>&1

cat > /usr/bin/${PROJECT} <<EOF
#!/bin/bash
case "\$1" in
    list)    pm2 logs ${PROJECT} --lines 15 --nostream ;;
    stop)    pm2 stop ${PROJECT} ;;
    start)   pm2 start ${PROJECT} ;;
    restart) pm2 restart ${PROJECT} ;;
    *)       echo "Usage: myagsb {list|start|stop|restart}" ;;
esac
EOF
chmod +x /usr/bin/${PROJECT}

echo -e "${GREEN}Success!${PLAIN}"
echo -e "Command: ${YELLOW}myagsb list${PLAIN} to see links."
sleep 3
myagsb list
