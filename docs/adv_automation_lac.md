SkyNets Bangladesh এখন ৫০ হাজার কাস্টমারে স্ট্যাবল। Nautobot দিয়ে নেটওয়ার্ক ডকুমেন্ট করছে, PyNautobot দিয়ে বেসিক অটোমেশন করছে। কিন্তু তাদের লক্ষ্য আরো বড় - পরের দুই বছরে ১ লক্ষ কাস্টমার।

জাহাঙ্গীর সাহেব একটা মিটিং ডাকলেন। বললেন, "আমরা এখন যে পদ্ধতিতে কাজ করছি সেটা ৫০ হাজারে চলছে। কিন্তু ১ লক্ষে গেলে আরো বেশি অটোমেশন দরকার।"

তিনি একটা whiteboard এ লিখলেন:

```
বর্তমান সমস্যা:
১. নতুন ডিভাইস add করার পরে ম্যানুয়ালি কনফিগার করতে হয়
২. প্রতিটা ডিভাইসের কনফিগ ব্যাকআপ নেই
৩. কনফিগ drift ট্র্যাক করতে পারি না (কে কখন কী চেঞ্জ করল)
৪. নতুন পপ যুক্ত করতে ২-৩ দিন লাগে

সমাধান:
১. Ansible দিয়ে কনফিগ অটোমেশন
২. Golden Config management
৩. Monitoring integration
৪. পুরো প্রসেস স্ট্রিমলাইন করা
```

এই চ্যাপ্টারে আমরা দেখব কীভাবে এডভান্সড অটোমেশন করতে হয়।

## Ansible - কনফিগ অটোমেশনের রাজা

Ansible হলো একটা অটোমেশন টুল যা দিয়ে নেটওয়ার্ক ডিভাইস কনফিগার করা যায়। এটার সাথে Nautobot integration করলে পাওয়ারফুল কম্বিনেশন হয়।

### Ansible কী করে?

সহজ ভাষায়: Ansible একটা playbook (স্ক্রিপ্ট) পড়ে, তারপর অনেকগুলো ডিভাইসে একসাথে commands execute করে।

উদাহরণ: "সব কোর রাউটারে NTP সার্ভার কনফিগার করো"

Ansible playbook লিখবেন একবার। তারপর ১০টা রাউটার হোক বা ১০০টা - একই playbook চালাবেন।

### Ansible ইনস্টল করা

```bash
# Ubuntu/Linux
sudo apt install ansible

# অথবা pip দিয়ে
pip install ansible

# Check করুন
ansible --version
```

### Nautobot Inventory Plugin

Ansible কে বলতে হবে কোন ডিভাইসগুলোতে কাজ করবে। Nautobot থেকে সেই লিস্ট নেওয়া যায়।

একটা inventory file তৈরি করুন: `nautobot_inventory.yml`

```yaml
plugin: networktocode.nautobot.inventory
api_endpoint: https://nautobot.skynets.bd
token: your-api-token-here
validate_certs: false

# কোন ডিভাইস নেবো
query_filters:
  - status: active

# ডিভাইস গ্রুপ করার নিয়ম
group_by:
  - role
  - location
```

এখন Ansible এর কাছে Nautobot এ যত ডিভাইস আছে সব দেখতে পাবে:

```bash
ansible-inventory -i nautobot_inventory.yml --list
```

### প্রথম Ansible Playbook - NTP কনফিগার করা

সব রাউটারে NTP সার্ভার সেট করব।

`configure_ntp.yml`:

```yaml
---
- name: Configure NTP on all routers
  hosts: core_router  # Nautobot থেকে "Core Router" role এর ডিভাইস
  gather_facts: no
  
  vars:
    ntp_server: 103.108.140.150  # BDIX NTP server
  
  tasks:
    - name: Configure NTP server
      ansible.netcommon.cli_config:
        config: |
          /system ntp client set enabled=yes
          /system ntp client set server-dns-names={{ ntp_server }}
```

চালান:

```bash
ansible-playbook -i nautobot_inventory.yml configure_ntp.yml
```

Ansible এখন Nautobot থেকে সব কোর রাউটারের লিস্ট নেবে, তারপর প্রতিটাতে NTP কনফিগার করবে।

### আরেকটা উদাহরণ - SNMP Community সেট করা

`configure_snmp.yml`:

```yaml
---
- name: Configure SNMP on all devices
  hosts: all
  gather_facts: no
  
  vars:
    snmp_community: SkyNets_R3ad0nly
  
  tasks:
    - name: Set SNMP community
      ansible.netcommon.cli_config:
        config: |
          /snmp community add name={{ snmp_community }} addresses=192.168.10.0/24
          /snmp set enabled=yes
```

একবার playbook লিখলে ১৫০টা ডিভাইসে একসাথে apply করা যায়। ম্যানুয়ালি করলে দিনের পর দিন লাগত!

### Nautobot + Ansible Workflow

```
১. Nautobot এ নতুন ডিভাইস add করুন
২. Ansible playbook চালান
৩. ডিভাইস অটোমেটিক্যালি কনফিগার হয়ে যাবে
```

এভাবে আসিফ এখন নতুন সুইচ যোগ করে Ansible playbook চালায়। ৫ মিনিটে সব কনফিগ হয়ে যায়!

---

## Golden Config - কনফিগ ম্যানেজমেন্ট

Golden Config মানে হলো "আদর্শ কনফিগারেশন"। প্রতিটা ডিভাইসের কী কনফিগ হওয়া উচিত সেটা ডিফাইন করা, তারপর actual কনফিগ চেক করা।

### সমস্যা যেটা solve করে

**Scenario:** গত সপ্তাহে একটা কোর রাউটারে ইন্টারনেট স্লো হচ্ছিল। দেখা গেল কেউ QoS কনফিগ চেঞ্জ করে দিয়েছে। কে করেছে? কখন করেছে? কেন করেছে? কিছু জানা নেই!

Golden Config এই সমস্যা solve করে:
- প্রতিটা ডিভাইসের কনফিগ ব্যাকআপ রাখে
- কনফিগ চেঞ্জ ট্র্যাক করে
- "কাঙ্ক্ষিত কনফিগ" থেকে বিচ্যুত হলে alert দেয়

### Nautobot Golden Config Plugin

Nautobot এর একটা plugin আছে - `nautobot-golden-config`।

**ইনস্টল করা:**

```bash
pip install nautobot-plugin-golden-config
```

Nautobot এর `nautobot_config.py` ফাইলে যোগ করুন:

```python
PLUGINS = [
    "nautobot_golden_config",
]

PLUGINS_CONFIG = {
    "nautobot_golden_config": {
        "enable_backup": True,
        "enable_compliance": True,
        "enable_intended": True,
    }
}
```

Nautobot restart করুন।

### Golden Config সেটআপ

এখন Nautobot UI তে **Golden Config** মেনু দেখবেন।

**১. Config Backup সেটআপ:**

Golden Config → Settings → Backup

```
Backup Repository: Git repository যেখানে কনফিগ সেভ হবে
Backup Path Template: configs/{{ obj.location.name }}/{{ obj.name }}.cfg
```

**২. Daily Backup Job চালান:**

Golden Config → Jobs → Config Backup

এটা চালালে সব ডিভাইসের কনফিগ ব্যাকআপ নেবে এবং Git এ সেভ করবে।

**৩. Compliance চেক:**

একটা "intended config" template তৈরি করুন যেটা বলে দেয় ডিভাইসে কী কনফিগ থাকা উচিত।

`templates/mikrotik_core_router.j2`:

```jinja2
# NTP Configuration
/system ntp client set enabled=yes server-dns-names=103.108.140.150

# SNMP Configuration  
/snmp community add name=SkyNets_R3ad0nly addresses=192.168.10.0/24
/snmp set enabled=yes contact="NOC Team" location="{{ device.location }}"

# DNS Configuration
/ip dns set servers=8.8.8.8,8.8.4.4

# Logging
/system logging add topics=info action=remote remote=192.168.10.100
```

Compliance Job চালালে দেখাবে কোন ডিভাইস এই স্ট্যান্ডার্ড মেনে চলছে, কোনটা চলছে না।

**Compliance Report:**

```
R-DN-MIR-CORE-01: ✓ Compliant
R-DN-UTT-CORE-01: ✗ Non-Compliant
  - Missing: NTP configuration
  - Mismatch: SNMP community name different
R-DN-GUL-CORE-01: ✓ Compliant
```

এখন জাহাঙ্গীর সাহেব জানেন উত্তরা কোর রাউটারে কনফিগ ঠিক নেই। সেখানে কাজ করতে হবে।

### Config Drift Detection

প্রতিদিন একবার backup নিন। তারপর দেখুন কোন কনফিগ চেঞ্জ হয়েছে কিনা।

**Drift Report:**

```
R-DN-MIR-CORE-01
  Changed: 2027-02-08 14:35:22
  Modified by: asif@192.168.10.25
  Changes:
    - Added: /ip firewall filter rule
    - Removed: Old ACL rule
```

এখন জানা যাচ্ছে আসিফ গতকাল দুপুর ২:৩৫ এ একটা ফায়ারওয়াল রুল যোগ করেছে। যদি কোনো সমস্যা হয়, তাহলে আগের কনফিগ restore করা যাবে।

---

## Monitoring Integration - Nautobot + Prometheus/Grafana

Nautobot শুধু ডকুমেন্টেশন না - এটা monitoring এর সাথেও integrate করা যায়।

### ধারণা

Nautobot থেকে ডিভাইস লিস্ট নিয়ে Prometheus/Zabbix/LibreNMS এ automatically add করা যায়।

**Python স্ক্রিপ্ট:**

`sync_to_prometheus.py`:

```python
from pynautobot import api
import yaml

nb = api(url="https://nautobot.skynets.bd", token="your-token")

# সব active ডিভাইস নিয়ে আসুন
devices = nb.dcim.devices.filter(status="active")

# Prometheus targets তৈরি করুন
targets = []

for device in devices:
    # Management IP খুঁজুন
    mgmt_ips = nb.ipam.ip_addresses.filter(device=device.name)
    
    if mgmt_ips:
        ip = str(mgmt_ips[0].address).split('/')[0]
        
        targets.append({
            'targets': [f"{ip}:161"],  # SNMP port
            'labels': {
                'device': device.name,
                'location': str(device.location),
                'role': str(device.role)
            }
        })

# Prometheus config ফাইলে লিখুন
with open('/etc/prometheus/targets/nautobot.yml', 'w') as f:
    yaml.dump(targets, f)

print(f"✓ Synced {len(targets)} devices to Prometheus")
```

এই স্ক্রিপ্ট চালালে Nautobot থেকে সব ডিভাইস নিয়ে Prometheus এ add করবে। Prometheus restart করলেই monitoring শুরু হবে।

**Cron দিয়ে অটোমেট:**

```bash
# প্রতি ঘন্টায় sync করুন
0 * * * * /usr/bin/python3 /opt/scripts/sync_to_prometheus.py
```

এখন Nautobot এ নতুন ডিভাইস add করলে ১ ঘন্টার মধ্যে automatically monitoring এ যুক্ত হবে!

---

## স্কেলিং মাইলস্টোন - ১০k থেকে ১০০k এর যাত্রা

SkyNets এর জার্নি:

```
২০২৩: ৮k কাস্টমার - Nautobot সেটআপ
২০২৫: ৩০k কাস্টমার - PyNautobot অটোমেশন শুরু
২০২৭: ৫০k কাস্টমার - Ansible integration
২০২৯: ১০০k কাস্টমার (লক্ষ্য) - Full automation
```

### প্রতিটা স্তরে কী দরকার

**১০k - ৩০k কাস্টমার:**
- Location hierarchy ঠিক রাখা
- Basic PyNautobot scripts
- Weekly data audit

**৩০k - ৫০k কাস্টমার:**
- Ansible playbooks
- Golden Config শুরু করা
- Monitoring integration

**৫০k - ১০০k কাস্টমার:**
- Full automation pipeline
- CI/CD for network changes
- AI-assisted troubleshooting (ভবিষ্যৎ)

### ১০০k এ পৌঁছানোর রোডম্যাপ

**বছর ১ (২০২৭-২০২৮):**
- আরো ১০টা পপ যোগ করা
- সব নতুন পপে day-1 automation
- Golden Config সব ডিভাইসে enable করা

**বছর ২ (২০২৮-২০২৯):**
- Network-as-Code approach
- Self-service portal (কাস্টমার নিজেই কিছু কাজ করতে পারবে)
- Predictive maintenance (AI দিয়ে সমস্যা হওয়ার আগেই ধরা)

### টেকনিক্যাল রিকোয়ারমেন্ট ১০০k এর জন্য

**Infrastructure:**
- Nautobot server: 16 GB RAM, 8 cores
- PostgreSQL optimization
- Redis caching
- Load balancing (যদি অনেক ইউজার একসাথে access করে)

**Team:**
- ৩-৪ জন Network Automation Engineer
- ১০+ জন NOC
- ১৫+ জন Field Operations

**Process:**
- সব change Nautobot দিয়ে ট্র্যাক করতে হবে
- কোনো manual change নয় (emergency ছাড়া)
- Weekly automation sprint (নতুন automation যোগ করা)

---

## AI এবং LLM Integration - ভবিষ্যৎ সম্ভাবনা

এটা একটু ফিউচারিস্টিক, কিন্তু সম্ভব।

### ChatGPT/Claude দিয়ে Network Assistant

একটা chatbot যেটা Nautobot API access করতে পারে:

**কথোপকথন:**

```
You: "Show me all devices in Gulshan POP"
Bot: *calls Nautobot API*
     "Found 24 devices in Gulshan POP:
      - 1 Core Router
      - 2 Distribution Switches  
      - 21 Access Switches"

You: "Which ones are offline?"
Bot: "2 devices are offline:
      - SW-DN-GUL-ACC-15 (offline since 2027-02-08)
      - SW-DN-GUL-ACC-22 (under maintenance)"

You: "Add a new access switch SW-DN-GUL-ACC-30"
Bot: *calls Nautobot API*
     "✓ Device created successfully!
      Do you want me to configure it with Ansible?"
```

এরকম chatbot তৈরি করা যায় LangChain দিয়ে। Nautobot API কে function calling এর মাধ্যমে integrate করতে হবে।

### Predictive Maintenance

Machine learning দিয়ে predict করা:

- কোন ডিভাইস শীঘ্রই fail করতে পারে (historical data থেকে)
- কোন prefix শীঘ্রই ফুল হবে (growth trend থেকে)
- কোন লিংকে bandwidth upgrade দরকার

এগুলো ২০২৮-২০২৯ এ আসবে যখন যথেষ্ট ডেটা জমা হবে।

---

## Performance at Scale - ১০০k এর জন্য প্রস্তুতি

### Database Optimization

PostgreSQL tuning:

```sql
# /etc/postgresql/16/main/postgresql.conf

shared_buffers = 8GB
effective_cache_size = 24GB
work_mem = 128MB
maintenance_work_mem = 2GB

max_connections = 200

# Indexing
CREATE INDEX idx_device_location ON dcim_device(location_id);
CREATE INDEX idx_ipaddress_device ON ipam_ipaddress(assigned_object_id);
```

### Caching Strategy

Redis caching enable করুন:

```python
# nautobot_config.py

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'TIMEOUT': 300,  # 5 minutes
    }
}
```

### API Rate Limiting

অনেক scripts একসাথে চললে API overload হতে পারে। Rate limiting সেট করুন:

```python
# nautobot_config.py

REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
}
```

SkyNets Bangladesh এখন একটা solid automation foundation পেয়েছে। তারা জানে কীভাবে ১ লক্ষ কাস্টমারে স্কেল করতে হবে। Nautobot শুধু একটা documentation tool না - এটা তাদের পুরো network operations এর হৃদয়।

এই বইয়ের শেষে এসে আমরা দেখলাম কীভাবে একটা ছোট ISP (৮ হাজার কাস্টমার) একটা মিডিয়াম ISP (৫০ হাজার) হয়ে বড় ISP (১ লক্ষ+) এর দিকে যাত্রা করতে পারে। আর সেই যাত্রায় Network Source of Truth (Nautobot) হলো মূল ভিত্তি।

---