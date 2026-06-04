# Bug Bounty Recon Toolkit — Complete Field Guide
### Tools: amass · assetfinder · subfinder · dnsx · httpx · naabu · waybackurls · gau · katana · gowitness · nuclei · gitleaks

> Throughout this guide, `target.com` is used as a placeholder. Replace it with your actual target domain.

---

## Tool Installation (Kali Linux)

```bash
# ProjectDiscovery tools (Go-based)
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest
go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install -v github.com/projectdiscovery/katana/cmd/katana@latest

# Other Go tools
go install -v github.com/lc/gau/v2/cmd/gau@latest
go install -v github.com/tomnomnom/assetfinder@latest
go install -v github.com/tomnomnom/waybackurls@latest
go install -v github.com/tomnomnom/anew@latest
go install -v github.com/sensepost/gowitness@latest

# amass and gitleaks via apt
sudo apt install amass gitleaks -y

# Add Go binaries to PATH (add this to ~/.zshrc or ~/.bashrc)
export PATH=$PATH:$(go env GOPATH)/bin
source ~/.zshrc
```

---

## Full Recon Pipeline Overview

```
TARGET DOMAIN
      │
      ▼
[Phase 1]  Subdomain Discovery     ──  subfinder, amass, assetfinder
      │
      ▼
[Phase 2]  DNS Resolution          ──  dnsx
      │
      ▼
[Phase 3]  HTTP Probing            ──  httpx
      │
      ▼
[Phase 4]  Port Scanning           ──  naabu
      │
      ▼
[Phase 5]  URL Discovery           ──  waybackurls, gau, katana
      │
      ▼
[Phase 6]  Visual Recon            ──  gowitness
      │
      ▼
[Phase 7]  Vulnerability Scanning  ──  nuclei
      │
      ▼
[Phase 8]  Secrets Hunting         ──  gitleaks
      │
      ▼
  FINDINGS
```

---

## Phase 1 — Subdomain Discovery

The goal of this phase is to enumerate as many subdomains of the target as possible. We use three tools and then combine their results.

---

### subfinder

subfinder is a passive subdomain discovery tool. It queries multiple third-party sources (DNS databases, certificate logs, etc.) without directly touching the target.

```bash
# Basic scan
subfinder -d target.com -o subs_subfinder.txt

# Silent mode (clean output, no banner)
subfinder -d target.com -silent -o subs_subfinder.txt

# Enable all sources and recursive discovery
subfinder -d target.com -all -recursive -o subs_subfinder.txt

# Scan multiple targets at once
subfinder -dL domains.txt -o subs_subfinder.txt -t 50
```

**Adding API keys to subfinder** (`~/.config/subfinder/provider-config.yaml`):

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

> Adding API keys significantly increases the number of subdomains discovered. Free tiers are available for all providers listed above.

---

### amass

amass performs both passive (OSINT-based) and active (DNS brute-force) enumeration. It is slower than subfinder but often finds subdomains others miss.

```bash
# Passive mode — safe, does not contact the target directly
amass enum -passive -d target.com -o subs_amass.txt

# Active mode — performs DNS zone transfers, brute-forcing, and more
amass enum -active -d target.com -o subs_amass.txt

# Intel mode — OSINT-focused, finds related domains and ASNs
amass intel -d target.com -o amass_intel.txt

# Use with a config file containing API keys
amass enum -passive -d target.com -config ~/.config/amass/config.ini

# Visualize the subdomain graph
amass viz -d3 -d target.com
```

**amass config file** (`~/.config/amass/config.ini`):

```ini
[Shodan]
apikey = YOUR_KEY

[VirusTotal]
apikey = YOUR_KEY
```

---

### assetfinder

assetfinder is a lightweight, fast tool that queries several sources and returns subdomains quickly. Good for quick wins.

```bash
# Basic discovery
assetfinder target.com | tee subs_assetfinder.txt

# Only return subdomains (exclude unrelated domains)
assetfinder --subs-only target.com | tee subs_assetfinder.txt
```

---

### Combining and Deduplicating Results

```bash
# Merge all subdomain lists and remove duplicates
cat subs_subfinder.txt subs_amass.txt subs_assetfinder.txt | sort -u | anew all_subs.txt

# Check how many unique subdomains you have
wc -l all_subs.txt
```

---

## Phase 2 — DNS Resolution (dnsx)

Not all discovered subdomains are live. This phase resolves each subdomain via DNS and filters out the dead ones. It also uncovers interesting DNS records like CNAME entries that may be vulnerable to subdomain takeover.

```bash
# Resolve subdomains and keep only live ones
dnsx -l all_subs.txt -o live_subs.txt

# Get A records (IP addresses) along with subdomains
dnsx -l all_subs.txt -a -resp -o live_subs_with_ip.txt

# Get CNAME records (important for subdomain takeover)
dnsx -l all_subs.txt -cname -resp -o cname_subs.txt

# Filter CNAME records pointing to unclaimed third-party services
dnsx -l all_subs.txt -cname -resp | grep -E \
  "amazonaws|github|heroku|azurewebsites|unbouncepages|surge.sh|netlify"

# Get multiple record types at once
dnsx -l all_subs.txt -a -cname -mx -txt -resp -o dns_full.txt

# Filter out wildcard DNS responses
dnsx -l all_subs.txt -wd target.com -o live_no_wildcard.txt
```

**What to look for in CNAME records:**

If a subdomain has a CNAME pointing to a third-party service that no longer exists or is unclaimed, you can register that service and take over the subdomain. Example:

```
staging.target.com  CNAME  target.azurewebsites.net
```

If `target.azurewebsites.net` is not registered, you can register it and serve your own content from `staging.target.com` — this is a **Subdomain Takeover**.

---

## Phase 3 — HTTP Probing (httpx)

DNS resolution tells you a subdomain is alive, but not whether it is serving HTTP/HTTPS. This phase identifies which live subdomains have web servers running on them.

```bash
# Basic probing
httpx -l live_subs.txt -o http_alive.txt

# Full output: status code, page title, detected technologies, IP
httpx -l live_subs.txt -status-code -title -tech-detect -ip -o http_full.txt

# Probe specific ports
httpx -l live_subs.txt -ports 80,443,8080,8443,3000,4443 -o http_multiport.txt

# Filter by specific HTTP response codes
httpx -l live_subs.txt -mc 200,301,302,401,403 -o http_interesting.txt

# Identify web server type
httpx -l live_subs.txt -web-server -o http_servers.txt

# JSON output for further processing
httpx -l live_subs.txt -json -o http_results.json

# Capture only 403 Forbidden responses (potential auth bypass targets)
httpx -l live_subs.txt -mc 403 -o http_403.txt
```

**What each status code means for bug hunting:**

| Status | Meaning | Bug Potential |
|--------|---------|---------------|
| 200 | OK | Normal, inspect the content |
| 401 | Unauthorized | Authentication bypass possible |
| 403 | Forbidden | Authorization bypass, path traversal |
| 500 | Server Error | Injection, unexpected input handling |
| 302/301 | Redirect | Open redirect possible |

---

## Phase 4 — Port Scanning (naabu)

Web servers do not only run on ports 80 and 443. Many targets expose sensitive services on non-standard ports. naabu is a fast port scanner built for bug bounty workflows.

```bash
# Scan top 1000 ports
naabu -l http_alive.txt -top-ports 1000 -o ports.txt

# Full port scan (all 65535 ports — slower)
naabu -l http_alive.txt -p - -o ports_full.txt

# Scan specific high-value ports
naabu -l http_alive.txt \
  -p 21,22,23,25,53,80,443,445,1433,3306,3389,5432,6379,8080,8443,9200,27017 \
  -o ports_juicy.txt

# Chain naabu directly into httpx
naabu -l http_alive.txt -top-ports 1000 -silent | httpx -silent -o live_services.txt

# Exclude CDN-hosted IPs (Cloudflare, Akamai, etc.)
naabu -l http_alive.txt -top-ports 1000 -exclude-cdn -o ports_no_cdn.txt
```

**High-value ports to look for:**

| Port | Service | Why It Matters |
|------|---------|----------------|
| 6379 | Redis | Often unauthenticated, full data access |
| 9200 / 9300 | Elasticsearch | Unauthenticated data exposure |
| 27017 | MongoDB | Often exposed without authentication |
| 5432 | PostgreSQL | Direct database access |
| 3306 | MySQL | Direct database access |
| 8080 / 8443 | Alternative HTTP | Dev panels, staging environments |
| 2375 | Docker API | Remote code execution via containers |
| 4848 | GlassFish Admin | Remote admin panel |

---

## Phase 5 — URL Discovery

This phase collects every known URL belonging to the target — historical URLs from archives, current URLs from crawling, and JavaScript-discovered endpoints. This is where you find the real attack surface.

---

### waybackurls

waybackurls fetches all URLs the Wayback Machine (archive.org) has ever crawled for a domain. These include old endpoints, deleted pages, and exposed files that still work.

```bash
# Fetch historical URLs for a domain
waybackurls target.com | tee wayback.txt

# Fetch for all live subdomains
cat http_alive.txt | waybackurls | tee wayback_all.txt

# Filter for interesting file extensions
cat wayback.txt | grep -E \
  "\.(php|asp|aspx|jsp|json|xml|config|env|bak|log|sql|zip|tar|gz)" \
  | tee wayback_juicy.txt

# Filter URLs that contain parameters
cat wayback.txt | grep "?" | tee wayback_params.txt
```

---

### gau (Get All URLs)

gau queries multiple sources — Wayback Machine, Common Crawl, and AlienVault OTX — and combines the results.

```bash
# Basic URL fetch
gau target.com | tee gau_urls.txt

# Specify sources
gau --providers wayback,commoncrawl,otx target.com | tee gau_urls.txt

# Fetch for all live subdomains
cat http_alive.txt | gau | tee gau_all.txt

# Filter for potential IDOR parameters
cat gau_urls.txt | grep "?" | \
  grep -E "id=|user=|account=|order=|invoice=|file=|page=" \
  | tee gau_idor_candidates.txt

# Remove static asset URLs (noise reduction)
cat gau_urls.txt | \
  grep -vE "\.(css|js|png|jpg|gif|ico|svg|woff|ttf|eot)" \
  | tee gau_filtered.txt
```

---

### katana

katana is an active web crawler. Unlike waybackurls and gau which rely on historical data, katana visits the target and discovers URLs in real time — including those found inside JavaScript files.

```bash
# Basic crawl
katana -u https://target.com -o katana_urls.txt

# Crawl with depth control
katana -u https://target.com -d 5 -o katana_deep.txt

# Parse JavaScript files to find hidden endpoints
katana -u https://target.com -jc -o katana_js.txt

# Use a headless browser (required for heavy JavaScript sites)
katana -u https://target.com -headless -o katana_headless.txt

# Crawl all live hosts from a file
katana -list http_alive.txt -d 3 -jc -o katana_all.txt

# Increase concurrency for speed
katana -u https://target.com -d 4 -c 20 -o katana_fast.txt

# Extract specific fields
katana -u https://target.com -field url,path,qurl -o katana_fields.txt
```

**Important katana flags:**

| Flag | Description |
|------|-------------|
| `-d` | Crawl depth (how many levels deep to follow links) |
| `-jc` / `-js-crawl` | Parse JavaScript files for hidden endpoints |
| `-headless` | Use a real browser engine to render JavaScript |
| `-c` | Number of concurrent requests |
| `-ef` | Exclude file extensions from crawling |

---

### Merging and Filtering All URLs

```bash
# Combine all URL sources and deduplicate
cat wayback.txt gau_urls.txt katana_urls.txt | sort -u | anew all_urls.txt

echo "[+] Total unique URLs: $(wc -l < all_urls.txt)"

# URLs with parameters only
cat all_urls.txt | grep "?" | tee param_urls.txt

# Potential SSRF candidates
cat all_urls.txt | grep -E "url=|redirect=|next=|return=|dest=|proxy=|fetch=" \
  | tee ssrf_candidates.txt

# Potential Open Redirect candidates
cat all_urls.txt | grep -E "redirect=|next=|return=|goto=|target=|url=http" \
  | tee redirect_candidates.txt

# Potential LFI candidates
cat all_urls.txt | grep -E "file=|path=|include=|page=|template=|dir=" \
  | tee lfi_candidates.txt

# Potential IDOR candidates
cat all_urls.txt | grep -E "id=[0-9]+|user_id=|account=|order_id=" \
  | tee idor_candidates.txt

# Exposed sensitive files
cat all_urls.txt | \
  grep -E "\.(env|bak|backup|sql|db|config|cfg|conf|log|old|tar\.gz|zip)" \
  | tee sensitive_files.txt

# API endpoints
cat all_urls.txt | grep -E "/api/|/v1/|/v2/|/v3/|/graphql|/rest/" \
  | tee api_endpoints.txt

# Admin and login panels
cat all_urls.txt | \
  grep -E "admin|panel|dashboard|login|manager|console|control|portal" \
  | tee admin_panels.txt
```

---

## Phase 6 — Visual Recon (gowitness)

Taking screenshots lets you quickly visually review hundreds of subdomains without visiting each one manually. This helps you spot login panels, admin dashboards, development environments, and exposed internal tools.

```bash
# Take a screenshot of a single URL
gowitness single https://target.com

# Screenshot all live hosts from a file
gowitness file -f http_alive.txt

# Add a delay to allow JavaScript to fully load
gowitness file -f http_alive.txt --delay 2

# Custom screen resolution
gowitness file -f http_alive.txt --resolution 1440,900

# Save screenshots to a specific folder
gowitness file -f http_alive.txt -P ./screenshots/

# Generate an HTML report for easy browsing
gowitness report serve
# Then open: http://localhost:7171

# Screenshot from naabu/nmap port scan output
gowitness nmap -f nmap_output.xml
```

**What to look for in screenshots:**
- Login pages on non-standard ports (could be internal tools)
- Admin panels (Jenkins, phpMyAdmin, Grafana, Kibana)
- Default "It works!" pages (server misconfiguration)
- Development or staging environments
- Error pages leaking framework/version information

---

## Phase 7 — Vulnerability Scanning (nuclei)

nuclei is a template-based vulnerability scanner. It uses YAML-based templates to detect specific vulnerabilities, misconfigurations, CVEs, and exposed panels at scale.

```bash
# Always update templates first
nuclei -update-templates

# Scan all live hosts
nuclei -l http_alive.txt -o nuclei_results.txt

# Only scan for high and critical severity
nuclei -l http_alive.txt -severity critical,high -o nuclei_high.txt

# Scan for specific vulnerability types
nuclei -l http_alive.txt -tags cve,sqli,xss,ssrf,lfi,rce,idor -o nuclei_vulns.txt

# Detect technologies and software versions
nuclei -l http_alive.txt -tags tech -o nuclei_tech.txt

# Find exposed admin and login panels
nuclei -l http_alive.txt -tags panel,login,admin -o nuclei_panels.txt

# Detect API misconfigurations
nuclei -l http_alive.txt -tags api,misconfig -o nuclei_api.txt

# Scan using only CVE templates from a specific year
nuclei -l http_alive.txt -t nuclei-templates/cves/2024/ -o nuclei_cve_2024.txt

# Fuzz parameters from URL list
nuclei -l param_urls.txt -tags fuzzing -o nuclei_fuzz.txt

# Authenticated scan with a session token
nuclei -l http_alive.txt \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Cookie: session=YOUR_COOKIE" \
  -severity critical,high -o nuclei_auth.txt

# Rate-limited scan to avoid WAF detection
nuclei -l http_alive.txt -severity critical,high -rl 30 -c 5 -o nuclei_ratelimited.txt

# JSON output for automation and parsing
nuclei -l http_alive.txt -json -o nuclei_results.json

# Pipe URL list directly into nuclei
cat param_urls.txt | nuclei -tags sqli,xss,ssrf -o nuclei_params.txt
```

**Important nuclei flags:**

| Flag | Description |
|------|-------------|
| `-severity` | Filter by severity: critical, high, medium, low, info |
| `-tags` | Filter by template tags |
| `-t` | Use a specific template or directory |
| `-rl` | Rate limit (requests per second) |
| `-c` | Number of concurrent templates |
| `-H` | Add custom headers (for authenticated scans) |
| `-json` | Output in JSON format |
| `-silent` | Only print findings, no progress output |

---

## Phase 8 — Secrets Hunting (gitleaks)

Developers sometimes accidentally commit sensitive information to Git repositories — API keys, passwords, tokens, and private keys. gitleaks scans Git history to find these secrets.

```bash
# Scan a local repository
gitleaks detect --source /path/to/repo -v

# Scan the full commit history of a repository
gitleaks detect --source /path/to/repo --log-opts="--all"

# Scan only the last 10 commits
gitleaks detect --source /path/to/repo --log-opts="HEAD~10..HEAD"

# Generate a JSON report
gitleaks detect --source /path/to/repo \
  --report-path gitleaks_report.json \
  --report-format json

# Clone a target's public repository and scan it
git clone https://github.com/target-org/some-repo /tmp/target_repo
gitleaks detect --source /tmp/target_repo --log-opts="--all" -v

# Scan files without Git history (non-Git directories)
gitleaks detect --source /tmp/target_files --no-git

# Use a custom rule configuration
gitleaks detect --source . --config ~/custom_gitleaks.toml
```

**What to look for:**
- AWS, GCP, or Azure credentials
- GitHub or GitLab personal access tokens
- Stripe, Twilio, or payment gateway API keys
- Database connection strings with passwords
- JWT signing secrets
- Private keys (RSA, SSH, PGP)
- Internal API endpoints or IP addresses
- `.env` file contents committed by mistake

---

## One-Liner Automation Chains

### Chain 1: Full subdomain discovery
```bash
TARGET="target.com"
subfinder -d $TARGET -silent | anew subs.txt
assetfinder --subs-only $TARGET | anew subs.txt
amass enum -passive -d $TARGET -silent | anew subs.txt
echo "[+] Total unique subdomains: $(wc -l < subs.txt)"
```

### Chain 2: Subdomains to live HTTP hosts
```bash
cat subs.txt | \
  dnsx -silent -a | \
  httpx -silent -status-code -title -tech-detect | \
  tee live_hosts.txt
```

### Chain 3: Full URL collection pipeline
```bash
TARGET="target.com"
waybackurls $TARGET | anew all_urls.txt
gau $TARGET | anew all_urls.txt
katana -u https://$TARGET -jc -silent | anew all_urls.txt
echo "[+] Total unique URLs: $(wc -l < all_urls.txt)"
```

### Chain 4: Port scan into HTTP probe
```bash
naabu -l live_hosts.txt -top-ports 1000 -silent | \
  httpx -silent -status-code -title | \
  tee services.txt
```

### Chain 5: Quick vulnerability scan
```bash
cat live_hosts.txt | \
  nuclei -severity critical,high,medium \
         -tags cve,sqli,xss,ssrf,lfi,rce \
         -silent \
         -o quick_vulns.txt
```

---

## Full Automated Recon Script

Save this as `recon.sh`, make it executable, and run it against any target.

```bash
#!/bin/bash

TARGET=$1

if [ -z "$TARGET" ]; then
  echo "Usage: ./recon.sh <domain>"
  exit 1
fi

mkdir -p recon/$TARGET/{subs,http,ports,urls,screenshots,vulns}

echo "[*] Phase 1: Subdomain Enumeration"
subfinder -d $TARGET -all -silent -o recon/$TARGET/subs/subfinder.txt
assetfinder --subs-only $TARGET > recon/$TARGET/subs/assetfinder.txt
amass enum -passive -d $TARGET -silent -o recon/$TARGET/subs/amass.txt
cat recon/$TARGET/subs/*.txt | sort -u > recon/$TARGET/subs/all_subs.txt
echo "[+] Total subdomains: $(wc -l < recon/$TARGET/subs/all_subs.txt)"

echo "[*] Phase 2: DNS Resolution"
dnsx -l recon/$TARGET/subs/all_subs.txt -silent \
  -o recon/$TARGET/subs/live_dns.txt
echo "[+] Live subdomains: $(wc -l < recon/$TARGET/subs/live_dns.txt)"

echo "[*] Phase 3: HTTP Probing"
httpx -l recon/$TARGET/subs/live_dns.txt -silent \
  -status-code -title -tech-detect \
  -o recon/$TARGET/http/alive.txt
echo "[+] Live HTTP hosts: $(wc -l < recon/$TARGET/http/alive.txt)"

echo "[*] Phase 4: Port Scanning"
naabu -l recon/$TARGET/subs/live_dns.txt \
  -top-ports 1000 -silent \
  -o recon/$TARGET/ports/open_ports.txt
echo "[+] Open ports found: $(wc -l < recon/$TARGET/ports/open_ports.txt)"

echo "[*] Phase 5: URL Discovery"
waybackurls $TARGET > recon/$TARGET/urls/wayback.txt
gau $TARGET > recon/$TARGET/urls/gau.txt
katana -list recon/$TARGET/http/alive.txt -jc -silent \
  -o recon/$TARGET/urls/katana.txt
cat recon/$TARGET/urls/*.txt | sort -u > recon/$TARGET/urls/all_urls.txt
echo "[+] Total unique URLs: $(wc -l < recon/$TARGET/urls/all_urls.txt)"

echo "[*] Phase 6: Screenshots"
gowitness file -f recon/$TARGET/http/alive.txt \
  -P recon/$TARGET/screenshots/ --delay 2

echo "[*] Phase 7: Vulnerability Scanning"
nuclei -l recon/$TARGET/http/alive.txt \
  -severity critical,high,medium \
  -silent \
  -o recon/$TARGET/vulns/nuclei.txt
echo "[+] Findings: $(wc -l < recon/$TARGET/vulns/nuclei.txt)"

echo ""
echo "[✓] Recon complete. Results saved to: recon/$TARGET/"
```

```bash
chmod +x recon.sh
./recon.sh target.com
```

---

## Recommended Folder Structure

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

## Free API Keys That Improve Results

| Provider | URL | Used By |
|----------|-----|---------|
| Shodan | shodan.io | subfinder, amass |
| VirusTotal | virustotal.com | subfinder, amass |
| GitHub | github.com/settings/tokens | subfinder, amass |
| Censys | search.censys.io | subfinder, amass |
| SecurityTrails | securitytrails.com | subfinder |
| AlienVault OTX | otx.alienvault.com | gau |
| URLScan | urlscan.io | subfinder |

---

*This guide covers passive reconnaissance, active enumeration, and automated vulnerability discovery as part of an ethical bug bounty workflow. Always operate within the defined scope of your target's bug bounty program.*
