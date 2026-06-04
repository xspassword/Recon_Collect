# 🎯 Complete Bug Bounty Recon Toolkit — Advanced Field Guide

> Practical commands, real workflows

---

## 📦 Tool Installation (Kali Linux)

```bash
# Go tools (subfinder, httpx, dnsx, naabu, nuclei, katana, gau, anew)
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest
go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install -v github.com/projectdiscovery/katana/cmd/katana@latest
go install -v github.com/lc/gau/v2/cmd/gau@latest
go install -v github.com/tomnomnom/anew@latest

# assetfinder, waybackurls, gowitness
go install -v github.com/tomnomnom/assetfinder@latest
go install -v github.com/tomnomnom/waybackurls@latest
go install -v github.com/sensepost/gowitness@latest

# amass
sudo apt install amass -y

# gitleaks
sudo apt install gitleaks -y

# PATH setup (add to ~/.bashrc or ~/.zshrc)
export PATH=$PATH:$(go env GOPATH)/bin
source ~/.zshrc
```

---

## 🗺️ FULL RECON PIPELINE — Phase by Phase

```
TARGET
  │
  ▼
[Phase 1] Subdomain Discovery    ←── subfinder + amass + assetfinder
  │
  ▼
[Phase 2] DNS Resolution         ←── dnsx
  │
  ▼
[Phase 3] HTTP Probing           ←── httpx
  │
  ▼
[Phase 4] Port Scanning          ←── naabu
  │
  ▼
[Phase 5] URL Discovery          ←── waybackurls + gau + katana
  │
  ▼
[Phase 6] Screenshot             ←── gowitness
  │
  ▼
[Phase 7] Vuln Scan              ←── nuclei
  │
  ▼
[Phase 8] Secrets Hunt           ←── gitleaks
  │
  ▼
💰 FINDINGS
```

---

## 🔴 PHASE 1 — Subdomain Discovery

### 📌 subfinder (fast, passive)
```bash
# Basic scan
subfinder -d target.com -o subs_subfinder.txt

# Verbose + silent mode (clean output)
subfinder -d target.com -silent -o subs_subfinder.txt

# Multiple sources enable korte (config file lagbe)
subfinder -d target.com -all -recursive -o subs_subfinder.txt

# Multiple targets
subfinder -dL domains.txt -o subs_subfinder.txt -t 50
```

**Config file** (`~/.config/subfinder/provider-config.yaml`) te API keys dite hobe:
```yaml
shodan:
  - YOUR_SHODAN_KEY
virustotal:
  - YOUR_VT_KEY
github:
  - YOUR_GITHUB_TOKEN
censys:
  - YOUR_CENSYS_ID:YOUR_CENSYS_SECRET
```
> API keys dile subfinder অনেক বেশি subdomains ধরবে।

---

### 📌 amass (deep, active + passive)
```bash
# Passive (safe, no direct contact with target)
amass enum -passive -d target.com -o subs_amass.txt

# Active (DNS brute-force, more aggressive)
amass enum -active -d target.com -o subs_amass.txt

# Full intel mode (OSINT + passive + active)
amass intel -d target.com -o amass_intel.txt

# Config file diye API keys use korte
amass enum -passive -d target.com -config ~/.config/amass/config.ini

# Subdomain graph visualize korte
amass viz -d3 -d target.com
```

**amass config** (`~/.config/amass/config.ini`):
```ini
[Shodan]
apikey = YOUR_KEY

[VirusTotal]
apikey = YOUR_KEY
```

---

### 📌 assetfinder (quick wins)
```bash
# Basic
assetfinder target.com | tee subs_assetfinder.txt

# Only subdomains (no wildcard)
assetfinder --subs-only target.com | tee subs_assetfinder.txt
```

---

### 📌 সব মিলিয়ে deduplicate করো
```bash
# Combine all subdomain lists and remove duplicates
cat subs_subfinder.txt subs_amass.txt subs_assetfinder.txt | sort -u | anew all_subs.txt

# Word count check
wc -l all_subs.txt
```

---

## 🔵 PHASE 2 — DNS Resolution (dnsx)

> Shudhu live/valid subdomains rakho, dead গুলো বাদ দাও।

```bash
# Basic DNS resolution
dnsx -l all_subs.txt -o live_subs.txt

# A records (IP address বের করো)
dnsx -l all_subs.txt -a -resp -o live_subs_with_ip.txt

# CNAME records (takeover potential!)
dnsx -l all_subs.txt -cname -resp -o cname_subs.txt

# CNAME check — potential subdomain takeover
dnsx -l all_subs.txt -cname -resp | grep -E "amazonaws|github|heroku|shopify|azurewebsites|unbouncepages"

# Multiple record types
dnsx -l all_subs.txt -a -cname -mx -txt -resp -o dns_full.txt

# Wildcard filter
dnsx -l all_subs.txt -wd target.com -o live_no_wildcard.txt
```

**কী দেখবা CNAME এ?**
যদি কোনো CNAME দেখো যেমন:
`shop.target.com CNAME target.myshopify.com` → এই third-party service unclaimed হলে **Subdomain Takeover** possible!

---

## 🟢 PHASE 3 — HTTP Probing (httpx)

> Kon subdomains আসলে web server চালু করছে সেটা বের করো।

```bash
# Basic probing
httpx -l live_subs.txt -o http_alive.txt

# Full info with status code, title, tech, IP
httpx -l live_subs.txt -status-code -title -tech-detect -ip -o http_full.txt

# Screenshot সহ (basic)
httpx -l live_subs.txt -screenshot -o http_screenshots.txt

# Specific ports probe
httpx -l live_subs.txt -ports 80,443,8080,8443,3000,4443 -o http_multiport.txt

# Specific response code filter
httpx -l live_subs.txt -mc 200,301,302,401,403 -o http_interesting.txt

# Web server filter
httpx -l live_subs.txt -web-server -o http_servers.txt

# Verbose full output (JSON format)
httpx -l live_subs.txt -json -o http_results.json

# 403 bypass check (juicy!)
httpx -l live_subs.txt -mc 403 -o http_403.txt
```

**Interesting targets কী?**
- `401/403` → Auth bypass possible
- `500` → Server error, injection possible
- Unknown tech stack → Manual testing worth it

---

## 🟡 PHASE 4 — Port Scanning (naabu)

> Hidden services, non-standard ports, এবং attack surface expand করো।

```bash
# Top 1000 ports
naabu -l http_alive.txt -top-ports 1000 -o ports.txt

# Full port scan (সব 65535 port)
naabu -l http_alive.txt -p - -o ports_full.txt

# Specific important ports
naabu -l http_alive.txt -p 21,22,23,25,53,80,443,445,1433,3306,3389,5432,6379,8080,8443,9200,27017 -o ports_juicy.txt

# naabu + httpx combined (pipeline)
naabu -l http_alive.txt -top-ports 1000 -silent | httpx -silent -o live_services.txt

# Exclude CDN (Cloudflare etc.)
naabu -l http_alive.txt -top-ports 1000 -exclude-cdn -o ports_no_cdn.txt
```

**কোন ports juicy?**
| Port | Service | Bug Potential |
|------|---------|---------------|
| 6379 | Redis | Unauthenticated access |
| 9200 | Elasticsearch | Data exposure |
| 27017 | MongoDB | No-auth DB |
| 8080/8443 | Alt HTTP | Dev panels |
| 3306 | MySQL | Direct DB access |
| 5432 | PostgreSQL | Direct DB access |

---

## 🟠 PHASE 5 — URL Discovery

### 📌 waybackurls (historical URLs)
```bash
# Target er historical URLs
waybackurls target.com | tee wayback.txt

# Live subdomains থেকে সব
cat http_alive.txt | waybackurls | tee wayback_all.txt

# Interesting extensions filter
cat wayback.txt | grep -E "\.(php|asp|aspx|jsp|json|xml|config|env|bak|log|sql|zip|tar|gz)" | tee wayback_juicy.txt

# Parameters আছে এমন URLs
cat wayback.txt | grep "?" | tee wayback_params.txt
```

---

### 📌 gau (Get All URLs — multiple sources)
```bash
# Basic URL fetch
gau target.com | tee gau_urls.txt

# Multiple providers
gau --providers wayback,commoncrawl,otx target.com | tee gau_urls.txt

# From list
cat http_alive.txt | gau | tee gau_all.txt

# Filter interesting params
cat gau_urls.txt | grep "?" | grep -E "id=|user=|admin=|file=|page=|url=|path=|redirect=" | tee gau_idor_params.txt

# Remove boring extensions
cat gau_urls.txt | grep -vE "\.(css|js|png|jpg|gif|ico|svg|woff|ttf)" | tee gau_filtered.txt
```

---

### 📌 katana (active crawling — deep spider)
```bash
# Basic crawl
katana -u https://target.com -o katana_urls.txt

# Deep crawl with depth
katana -u https://target.com -d 5 -o katana_deep.txt

# JavaScript file crawl (hidden endpoints!)
katana -u https://target.com -js-crawl -o katana_js.txt

# Headless browser (JS-heavy sites)
katana -u https://target.com -headless -o katana_headless.txt

# From live hosts list
katana -list http_alive.txt -d 3 -jc -o katana_all.txt

# Form inputs, hidden fields খোঁজো
katana -u https://target.com -field url,path,fqdn,rdn,rurl,qurl -o katana_fields.txt

# Concurrency বাড়াও
katana -u https://target.com -d 4 -c 20 -o katana_fast.txt
```

**katana flags মনে রাখো:**
- `-js-crawl` / `-jc` → JS files parse করে hidden endpoints বের করে
- `-headless` → Real browser দিয়ে crawl (JavaScript render হয়)
- `-d` → Depth (কত level deep যাবে)
- `-c` → Concurrency

---

### 📌 URL Merging + Filtering
```bash
# সব URL একসাথে
cat wayback.txt gau_urls.txt katana_urls.txt | sort -u | anew all_urls.txt

# Interesting parameter URLs
cat all_urls.txt | grep "?" | uro | tee param_urls.txt
# (uro install: go install github.com/s0md3v/uro@latest)

# Potential SSRF, Open Redirect
cat all_urls.txt | grep -E "url=|redirect=|next=|return=|dest=|target=|rurl=" | tee ssrf_candidates.txt

# Potential LFI
cat all_urls.txt | grep -E "file=|path=|include=|page=|dir=|template=" | tee lfi_candidates.txt

# Potential SQLi
cat all_urls.txt | grep "?" | tee sqli_candidates.txt
```

---

## 📸 PHASE 6 — Screenshots (gowitness)

> কোন subdomains দেখতে কেমন — manually check করার আগে screenshot নাও।

```bash
# Single URL
gowitness single https://target.com

# From file
gowitness file -f http_alive.txt

# With custom delay (JS load হওয়ার time দাও)
gowitness file -f http_alive.txt --delay 2

# Custom resolution
gowitness file -f http_alive.txt --resolution 1440,900

# Output folder specify
gowitness file -f http_alive.txt -P ./screenshots/

# Report generate করো (HTML)
gowitness report serve

# Specific ports
gowitness file -f http_alive.txt --ports 80,443,8080

# Nmap XML থেকে directly
gowitness nmap -f nmap_output.xml
```

**Tips:**
- Screenshot report browser এ open করো: `http://localhost:7171`
- Login pages, admin panels, dev dashboards খোঁজো visually

---

## ⚡ PHASE 7 — Vulnerability Scanning (nuclei)

### Templates Update
```bash
nuclei -update-templates
```

### Basic Scans
```bash
# All templates দিয়ে scan
nuclei -l http_alive.txt -o nuclei_results.txt

# Severity filter
nuclei -l http_alive.txt -severity critical,high,medium -o nuclei_important.txt

# Specific tags
nuclei -l http_alive.txt -tags cve,sqli,xss,ssrf,lfi,rce,idor -o nuclei_vulns.txt

# Technology detection
nuclei -l http_alive.txt -tags tech -o nuclei_tech.txt

# Exposed panels (admin, login, dashboards)
nuclei -l http_alive.txt -tags panel,login,admin -o nuclei_panels.txt

# API misconfigurations
nuclei -l http_alive.txt -tags api,misconfig -o nuclei_api.txt

# Specific template দিয়ে
nuclei -l http_alive.txt -t nuclei-templates/cves/2023/ -o nuclei_cve_2023.txt
```

### Advanced nuclei
```bash
# Fuzzing mode (parameter fuzzing)
nuclei -l param_urls.txt -tags fuzzing -o nuclei_fuzz.txt

# Rate limit control
nuclei -l http_alive.txt -severity critical,high -rl 50 -c 10 -o nuclei_ratelimited.txt

# Custom header (authenticated scan)
nuclei -l http_alive.txt -H "Authorization: Bearer YOUR_TOKEN" -o nuclei_auth.txt

# JSON output (better parsing)
nuclei -l http_alive.txt -json -o nuclei_results.json

# Silent mode (only findings print)
nuclei -l http_alive.txt -silent -severity critical,high

# Workflow: URL list থেকে pipe
cat param_urls.txt | nuclei -tags sqli,xss -o nuclei_params.txt
```

---

## 🔐 PHASE 8 — Secrets Hunting (gitleaks)

```bash
# Local repo scan
gitleaks detect --source /path/to/repo -v

# Specific commit range
gitleaks detect --source /path/to/repo --log-opts="HEAD~10..HEAD"

# Full history scan
gitleaks detect --source /path/to/repo --log-opts="--all"

# Report generate
gitleaks detect --source /path/to/repo --report-path gitleaks_report.json --report-format json

# Remote repo (clone করে scan)
git clone https://github.com/target/repo /tmp/target_repo
gitleaks detect --source /tmp/target_repo --log-opts="--all" -v

# Specific file scan
gitleaks detect --source /tmp/target_repo --no-git

# Custom config (extra rules)
gitleaks detect --source . --config ~/gitleaks_custom.toml
```

**কী খুঁজবা?**
- AWS/GCP/Azure credentials
- GitHub/GitLab tokens
- Stripe/PayPal API keys
- Database passwords
- JWT secrets
- Shopify API keys/secrets
- Private keys (RSA, SSH)

---

## 🚀 ONE-LINER AUTOMATION CHAINS

### Chain 1: Full Subdomain Discovery
```bash
TARGET="target.com"
subfinder -d $TARGET -silent | \
  anew subs.txt && \
assetfinder --subs-only $TARGET | \
  anew subs.txt && \
amass enum -passive -d $TARGET -silent | \
  anew subs.txt && \
echo "[+] Total unique subs: $(wc -l < subs.txt)"
```

### Chain 2: Subdomain → Live Hosts
```bash
cat subs.txt | \
  dnsx -silent -a | \
  httpx -silent -status-code -title -tech-detect | \
  tee live_hosts.txt
```

### Chain 3: URL Discovery Pipeline
```bash
TARGET="target.com"
echo $TARGET | \
  waybackurls | anew all_urls.txt && \
gau $TARGET | anew all_urls.txt && \
katana -u https://$TARGET -jc -silent | anew all_urls.txt && \
echo "[+] Total URLs: $(wc -l < all_urls.txt)"
```

### Chain 4: Quick Vuln Scan
```bash
cat live_hosts.txt | \
  nuclei -severity critical,high,medium \
         -tags cve,sqli,xss,ssrf,lfi,rce \
         -silent \
         -o quick_vulns.txt
```

### Chain 5: Full Automated Pipeline (Production)
```bash
#!/bin/bash
TARGET=$1
mkdir -p recon/$TARGET/{subs,http,urls,screenshots,vulns,ports}

echo "[*] Phase 1: Subdomain Enumeration"
subfinder -d $TARGET -all -silent -o recon/$TARGET/subs/subfinder.txt
assetfinder --subs-only $TARGET > recon/$TARGET/subs/assetfinder.txt
amass enum -passive -d $TARGET -silent -o recon/$TARGET/subs/amass.txt
cat recon/$TARGET/subs/*.txt | sort -u > recon/$TARGET/subs/all_subs.txt
echo "[+] Subs found: $(wc -l < recon/$TARGET/subs/all_subs.txt)"

echo "[*] Phase 2: DNS Resolution"
dnsx -l recon/$TARGET/subs/all_subs.txt -silent -o recon/$TARGET/subs/live_dns.txt

echo "[*] Phase 3: HTTP Probing"
httpx -l recon/$TARGET/subs/live_dns.txt -silent \
      -status-code -title -tech-detect \
      -o recon/$TARGET/http/alive.txt

echo "[*] Phase 4: Port Scanning"
naabu -l recon/$TARGET/subs/live_dns.txt \
      -top-ports 1000 -silent \
      -o recon/$TARGET/ports/open_ports.txt

echo "[*] Phase 5: URL Discovery"
waybackurls $TARGET > recon/$TARGET/urls/wayback.txt
gau $TARGET > recon/$TARGET/urls/gau.txt
katana -list recon/$TARGET/http/alive.txt -jc -silent -o recon/$TARGET/urls/katana.txt
cat recon/$TARGET/urls/*.txt | sort -u > recon/$TARGET/urls/all_urls.txt
echo "[+] URLs found: $(wc -l < recon/$TARGET/urls/all_urls.txt)"

echo "[*] Phase 6: Screenshots"
gowitness file -f recon/$TARGET/http/alive.txt \
               -P recon/$TARGET/screenshots/ --delay 2

echo "[*] Phase 7: Nuclei Scan"
nuclei -l recon/$TARGET/http/alive.txt \
       -severity critical,high,medium \
       -silent \
       -o recon/$TARGET/vulns/nuclei.txt

echo "[+] DONE! Check recon/$TARGET/"
```

**Use করো:**
```bash
chmod +x recon.sh
./recon.sh target.com
```

---

## 💡 INTERESTING PATTERNS — কী খুঁজবা?

### IDOR Candidates (URL patterns)
```bash
cat all_urls.txt | grep -E "id=[0-9]+|user_id=|account=|order=" | tee idor_candidates.txt
```

### Open Redirect Candidates
```bash
cat all_urls.txt | grep -E "redirect=|next=|return=|goto=|url=http" | tee redirect_candidates.txt
```

### SSRF Candidates
```bash
cat all_urls.txt | grep -E "url=|fetch=|dest=|api=|endpoint=|proxy=" | tee ssrf_candidates.txt
```

### Sensitive Files (Backup/Config)
```bash
cat all_urls.txt | grep -E "\.(env|bak|backup|sql|db|config|cfg|conf|log|old|tar\.gz|zip)" | tee sensitive_files.txt
```

### Admin/Login Panels
```bash
cat all_urls.txt | grep -E "admin|panel|dashboard|login|manager|console|control" | tee admin_panels.txt
```

### API Endpoints
```bash
cat all_urls.txt | grep -E "/api/|/v1/|/v2/|/v3/|/graphql|/rest/" | tee api_endpoints.txt
```

---

## 📁 Folder Structure (Best Practice)

```
recon/
└── target.com/
    ├── subs/
    │   ├── subfinder.txt
    │   ├── amass.txt
    │   ├── assetfinder.txt
    │   ├── all_subs.txt
    │   └── live_dns.txt
    ├── http/
    │   └── alive.txt
    ├── ports/
    │   └── open_ports.txt
    ├── urls/
    │   ├── wayback.txt
    │   ├── gau.txt
    │   ├── katana.txt
    │   └── all_urls.txt
    ├── screenshots/
    ├── vulns/
    │   └── nuclei.txt
    └── secrets/
        └── gitleaks_report.json
```

---

## 🔑 API Keys কোথায় পাবা (Free)

| Provider | URL | কাজ |
|----------|-----|-----|
| Shodan | shodan.io | Internet-wide scan data |
| VirusTotal | virustotal.com | Subdomain, DNS info |
| GitHub | github.com/settings/tokens | Code search |
| Censys | search.censys.io | IP, cert data |
| SecurityTrails | securitytrails.com | DNS history |
| AlienVault OTX | otx.alienvault.com | Threat intel |

---

*Built for Rivn — HackerOne @zerodayvigil | Bug Bounty Hunter*
