এতক্ষণ আমরা শিখলাম Nautobot-এর বিভিন্ন ফিচার - কীভাবে সাইট তৈরি করতে হয়, ডিভাইস এড করতে হয়, আইপি ম্যানেজ করতে হয়, ক্যাবল ডকুমেন্ট করতে হয়। এখন সময় এসেছে এই সব জ্ঞান একসাথে কাজে লাগানোর। এই চ্যাপ্টারে আমরা দেখব কীভাবে একটা ছোট আইএসপির জন্য শুরু থেকে শেষ পর্যন্ত Nautobot সেটআপ করতে হয়।

আমাদের উদাহরণ হিসেবে থাকবে "SkyNet Bangladesh"। তারা ঢাকার একটা এলাকায় ইন্টারনেট সার্ভিস দেয়। বর্তমানে তাদের প্রায় পাঁচ হাজার রেসিডেনশিয়াল কাস্টমার আছে। দুটো পপ - একটা মিরপুরে, আরেকটা কল্যাণপুরে। তারা এক্সেল শিট আর হাতে লেখা ডায়াগ্রাম দিয়ে নেটওয়ার্ক ম্যানেজ করছিল। এখন Nautobot-এ মুভ করার সিদ্ধান্ত নিয়েছে।

### শুরুর আগে - পরিকল্পনা করা

Nautobot-এ ডেটা এন্ট্রি শুরু করার আগে একটা পরিকল্পনা করা জরুরি। না হলে মাঝপথে গিয়ে দেখবেন গোলমাল হয়ে গেছে, আবার নতুন করে শুরু করতে হচ্ছে।

### SkyNet-এর বর্তমান নেটওয়ার্ক পরিস্থিতি

আমরা প্রথমে দেখি SkyNet-এর নেটওয়ার্ক কেমন:

**মিরপুর পপ:**

- ১টা কোর রাউটার (MikroTik CCR2004)
- ২টা ডিস্ট্রিবিউশন সুইচ (TP-Link TL-SG3428)
- ১০টা এক্সেস সুইচ (TP-Link TL-SG1024D)
- আনুমানিক ৩ হাজার কাস্টমার
- BTCL থেকে ৫ Gbps আপলিংক

**কল্যাণপুর পপ:**

- ১টা কোর রাউটার (MikroTik CCR2004)
- ১টা ডিস্ট্রিবিউশন সুইচ (TP-Link TL-SG3428)
- ৬টা এক্সেস সুইচ (TP-Link TL-SG1024D)
- আনুমানিক ২ হাজার কাস্টমার
- Summit থেকে ৩ Gbps আপলিংক

**আইপি রিসোর্স:**

- একটা /22 পাবলিক আইপি ব্লক (103.125.40.0/22)
- ম্যানেজমেন্টের জন্য প্রাইভেট আইপি (10.10.0.0/16)

### নেমিং কনভেনশন ডিফাইন করা

সবার আগে একটা নেমিং কনভেনশন ঠিক করে ফেলি। SkyNet-এর জন্য এই স্ট্যান্ডার্ড:

**ডিভাইস নেমিং:**
```
Format: [Type]-[Site]-[Role]-[Number]

Type:
  R = Router
  SW = Switch
  OLT = OLT (যদি ফাইবার নেটওয়ার্ক থাকে)
  FW = Firewall

Site:
  MIR = Mirpur
  KAL = Kalyanpur

Role:
  CORE = Core device
  DIST = Distribution
  ACC = Access

Number: 01, 02, 03... (zero-padded)

উদাহরণ:
  R-MIR-CORE-01 = Router, Mirpur, Core, #1
  SW-KAL-ACC-03 = Switch, Kalyanpur, Access, #3
```

**ইন্টারফেস ডেসক্রিপশন:**
```
Format: [Purpose] - [Remote End/Details]

উদাহরণ:
  "Uplink to BTCL"
  "Link to SW-MIR-DIST-01 port 1"
  "Building-A Distribution"
```

**VLAN নেমিং:**
```
VLAN ID + Descriptive Name

উদাহরণ:
  VLAN 10: MGMT_NETWORK
  VLAN 100: RESIDENTIAL_MIR
  VLAN 101: RESIDENTIAL_KAL
  VLAN 200: CORPORATE
```

এই কনভেনশন একটা ডকুমেন্টে লিখে রাখুন। টিমের সবাইকে শেয়ার করুন।

### সপ্তাহ ১: ফাউন্ডেশন সেটআপ

প্রথম সপ্তাহে আমরা বেসিক স্ট্রাকচার তৈরি করব। কোনো ডিভাইস এড করব না, শুধু ফাউন্ডেশন।

### দিন ১: Nautobot ইনস্টলেশন এবং বেসিক সেটিংস

Nautobot ইনস্টল করুন (আগের চ্যাপ্টার দেখুন)। ইনস্টলেশন হয়ে গেলে:

১. **টাইম জোন সেট করুন:** Settings থেকে নিশ্চিত করুন Asia/Dhaka সিলেক্ট করা আছে।

২. **প্রথম সুপারইউজার ছাড়া আরো একজন ব্যাকআপ এডমিন তৈরি করুন:** যদি প্রাইমারি এডমিন অ্যাকাউন্ট কোনো সমস্যায় পড়ে, ব্যাকআপ থাকবে।

৩. **ডেটাবেস ব্যাকআপ সেটআপ করুন:** প্রতিদিন অটোমেটিক ব্যাকআপ চালু করুন (আগের চ্যাপ্টারের স্ক্রিপ্ট দেখুন)।

### দিন ২-৩: Location এবং Rack সেটআপ

**Location Type তৈরি:**

```
Name: POP
Description: Point of Presence
Nestable: ✓
```

**মিরপুর পপ তৈরি:**

```
Name: Mirpur POP
Location Type: POP
Status: Active
Physical Address: House 25, Road 10, Mirpur-12, Dhaka-1216
Latitude: 23.8103
Longitude: 90.3654
Contact Name: Jahangir Khan
Contact Phone: +880 1712-345678
Contact Email: jahangir@skynet.bd
Description: Primary POP serving Mirpur area. Approximately 3000 residential customers.
```

**কল্যাণপুর পপ তৈরি:**

```
Name: Kalyanpur POP
Location Type: POP
Status: Active
Physical Address: Plot 15, Road 3, Kalyanpur, Dhaka-1207
Latitude: 23.7765
Longitude: 90.3568
Contact Name: Asif Rahman
Contact Phone: +880 1811-234567
Contact Email: asif@skynet.bd
Description: Secondary POP serving Kalyanpur area. Approximately 2000 residential customers.
```

**Rack তৈরি করুন:**

মিরপুরে একটা:
```
Name: Rack-MIR-A
Location: Mirpur POP
Status: Active
Type: 4-post cabinet
Width: 19 inches
Height: 42U
```

কল্যাণপুরে একটা:
```
Name: Rack-KAL-A
Location: Kalyanpur POP
Status: Active
Type: 4-post cabinet
Width: 19 inches
Height: 42U
```

### দিন ৪: Manufacturer এবং Device Types

**Manufacturers:**

```
1. MikroTik
2. TP-Link
3. Cisco (ভবিষ্যতের জন্য)
```

**Device Types:**

MikroTik CCR2004:
```
Manufacturer: MikroTik
Model: CCR2004-1G-12S+2XS
Height: 1U
Is Full Depth: ✓
Description: Cloud Core Router
```

TP-Link Managed Switch:
```
Manufacturer: TP-Link
Model: TL-SG3428
Height: 1U
Is Full Depth: ✓
Description: 28-Port Gigabit L2+ Managed Switch
```

TP-Link Unmanaged Switch:
```
Manufacturer: TP-Link
Model: TL-SG1024D
Height: 1U
Is Full Depth: ✓
Description: 24-Port Gigabit Unmanaged Switch
```

**Interface Templates যোগ করুন** (গুরুত্বপূর্ণ):

MikroTik CCR2004-এর জন্য:

- ether1 (1000BASE-T)
- sfp-sfpplus1 থেকে sfp-sfpplus12 (10GBASE-X-SFP+)
- sfp28-1, sfp28-2 (25GBASE-X-SFP28)

TP-Link TL-SG3428-এর জন্য:

- ether1 থেকে ether24 (1000BASE-T)
- sfp1 থেকে sfp4 (1000BASE-X-SFP)

TP-Link TL-SG1024D-এর জন্য:

- ether1 থেকে ether24 (1000BASE-T)

### দিন ৫: Device Roles এবং Tags

**Device Roles:**

```
1. Core Router (Color: Red)
2. Distribution Switch (Color: Orange)
3. Access Switch (Color: Green)
4. Firewall (Color: Purple) - ভবিষ্যতের জন্য
```

**Tags তৈরি:**

```
1. Production (Green)
2. Staging (Orange)
3. Critical (Red)
4. End-of-Life (Gray)
5. Backup (Blue)
```

### সপ্তাহ ২: মিরপুর পপ ডকুমেন্টেশন

এখন আসল ডেটা এন্ট্রি শুরু। প্রথমে মিরপুর পপ কমপ্লিট করব।

### দিন ৬-৭: Provider এবং Circuit

**BTCL Provider:**

```
Name: BTCL
ASN: 17494
Account Number: SKY-BTCL-2023-MIR
Portal URL: https://isp.btcl.gov.bd
NOC Contact: +880 2-9555555
Admin Contact: noc@btcl.gov.bd
Comments: 
- Contract Start: 01-Mar-2023
- Contract Duration: 2 years
- Monthly Cost: BDT 400,000
- Renewal Date: 01-Mar-2025
```

**Circuit Type:**

```
Name: Fiber Optic
Description: Fiber optic connectivity
```

**BTCL Circuit:**

```
Circuit ID: BTCL-MIR-001
Provider: BTCL
Type: Fiber Optic
Status: Active
Install Date: 2023-03-01
Commit Rate: 5000 Mbps
Description: Primary 5Gbps uplink from Mirpur POP
Comments:
- Installation completed: 15-Mar-2023
- Circuit Test Report: Passed
- Handover Date: 20-Mar-2023
```

**Circuit Termination:**

Termination A (BTCL Side):
```
Term Side: A
Port Speed: 5 Gbps
Upstream Speed: 5000 Mbps
PP Info: BTCL-PP-MIR-12
```

### দিন ৮-৯: কোর এবং ডিস্ট্রিবিউশন ডিভাইস

**মিরপুর কোর রাউটার:**

```
Name: R-MIR-CORE-01
Device Type: MikroTik CCR2004-1G-12S+2XS
Role: Core Router
Location: Mirpur POP
Rack: Rack-MIR-A
Position: 25U
Face: Front
Status: Active
Serial Number: ABC1234MIR001
Asset Tag: SKY-RTR-001
Tags: Production, Critical
Description: Primary core router for Mirpur POP
Comments:
- Purchased: Dec 2022
- Vendor: TechSource BD
- Purchase Order: PO-2022-12-015
- Warranty Expires: Dec 2025
```

**Circuit Termination Z যোগ করুন:**

এখন BTCL Circuit-এ ফিরে যান:

```
Term Side: Z
Location: Mirpur POP
Device: R-MIR-CORE-01
Interface: sfp-sfpplus1
Port Speed: 5 Gbps
Description: Connected to core router uplink port
```

**মিরপুর ডিস্ট্রিবিউশন সুইচ (২টা):**

CSV দিয়ে একসাথে এড করুন (`mirpur-dist-switches.csv`):

```csv
name,device_type,role,location,rack,position,status,serial,asset_tag,tags,description
SW-MIR-DIST-01,TP-Link TL-SG3428,Distribution Switch,Mirpur POP,Rack-MIR-A,20,Active,TPL1234DIST01,SKY-SW-001,production,"Primary distribution switch"
SW-MIR-DIST-02,TP-Link TL-SG3428,Distribution Switch,Mirpur POP,Rack-MIR-A,19,Active,TPL1234DIST02,SKY-SW-002,production,"Secondary distribution switch"
```

### দিন ১০: এক্সেস সুইচ (১০টা)

CSV ফাইল (`mirpur-access-switches.csv`):

```csv
name,device_type,role,location,rack,position,status,serial,asset_tag,tags,description
SW-MIR-ACC-01,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,15,Active,TPLACC001,SKY-SW-011,production,"Building-A access"
SW-MIR-ACC-02,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,14,Active,TPLACC002,SKY-SW-012,production,"Building-B access"
SW-MIR-ACC-03,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,13,Active,TPLACC003,SKY-SW-013,production,"Building-C access"
SW-MIR-ACC-04,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,12,Active,TPLACC004,SKY-SW-014,production,"Building-D access"
SW-MIR-ACC-05,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,11,Active,TPLACC005,SKY-SW-015,production,"Building-E access"
SW-MIR-ACC-06,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,10,Active,TPLACC006,SKY-SW-016,production,"Building-F access"
SW-MIR-ACC-07,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,9,Active,TPLACC007,SKY-SW-017,production,"Building-G access"
SW-MIR-ACC-08,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,8,Active,TPLACC008,SKY-SW-018,production,"Building-H access"
SW-MIR-ACC-09,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,7,Active,TPLACC009,SKY-SW-019,production,"Building-I access"
SW-MIR-ACC-10,TP-Link TL-SG1024D,Access Switch,Mirpur POP,Rack-MIR-A,6,Active,TPLACC010,SKY-SW-020,production,"Building-J access"
```

এই CSV ইমপোর্ট করুন। দশটা সুইচ একসাথে তৈরি হয়ে যাবে।

### দিন ১১-১২: Cable Connections ডকুমেন্ট করা

এখন সব ডিভাইস কীভাবে কানেক্টেড সেটা ডকুমেন্ট করুন।

**কোর থেকে ডিস্ট্রিবিউশন:**

```
Cable 1:
  From: R-MIR-CORE-01, sfp-sfpplus2
  To: SW-MIR-DIST-01, sfp1
  Type: SMF (Single-Mode Fiber)
  Color: Blue
  Length: 3m
  Label: CORE-TO-DIST-01

Cable 2:
  From: R-MIR-CORE-01, sfp-sfpplus3
  To: SW-MIR-DIST-02, sfp1
  Type: SMF
  Color: Blue
  Length: 3m
  Label: CORE-TO-DIST-02
```

**ডিস্ট্রিবিউশন থেকে এক্সেস (১০টা):**

এগুলো একটা একটা করে করতে হবে। প্রতিটা এক্সেস সুইচ একটা ডিস্ট্রিবিউশন সুইচের সাথে কানেক্টেড। যেমন:

```
Cable 3:
  From: SW-MIR-DIST-01, ether1
  To: SW-MIR-ACC-01, ether24
  Type: CAT6
  Color: Yellow
  Length: 20m
  Label: DIST01-TO-ACC01
```

এভাবে সব ১০টা এক্সেস সুইচের জন্য ক্যাবল তৈরি করুন। ৫টা যাবে DIST-01-এ, ৫টা যাবে DIST-02-এ (লোড ব্যালান্সিং)।

### সপ্তাহ ৩: IPAM এবং VLAN সেটআপ

এখন নেটওয়ার্কের লজিক্যাল দিক - আইপি আর ভিল্যান।

### দিন ১৩-১৪: IP Namespace এবং Parent Prefixes

**IP Namespace:**

```
Name: Global
Description: Primary IP namespace for SkyNet Bangladesh
```

**Parent Prefix (পাবলিক আইপি):**

```
Prefix: 103.125.40.0/22
Type: Container
Status: Active
Description: Assigned public IP block from BTCL
```

**Parent Prefix (প্রাইভেট/ম্যানেজমেন্ট):**

```
Prefix: 10.10.0.0/16
Type: Container
Status: Active
Description: Internal management network
```

### দিন ১৫: Child Prefixes তৈরি

**পাবলিক আইপি ভাগ করা:**

```
Parent: 103.125.40.0/22
  ├─ 103.125.40.0/24 - Residential Mirpur
  ├─ 103.125.41.0/24 - Residential Kalyanpur
  ├─ 103.125.42.0/25 - Corporate Clients
  ├─ 103.125.42.128/25 - Infrastructure
  └─ 103.125.43.0/24 - Reserved for future
```

এগুলো Nautobot-এ এড করুন:

```
Prefix: 103.125.40.0/24
Parent: 103.125.40.0/22
Type: Network
Status: Active
Location: Mirpur POP
Description: Residential customer IPs - Mirpur
```

একইভাবে বাকিগুলো।

**প্রাইভেট আইপি ভাগ করা:**

```
Parent: 10.10.0.0/16
  ├─ 10.10.1.0/24 - Mirpur Infrastructure
  ├─ 10.10.2.0/24 - Kalyanpur Infrastructure
  ├─ 10.10.10.0/24 - Management Network
  └─ 10.10.100.0/22 - Corporate Private IPs
```

### দিন ১৬: VLANs তৈরি

**মিরপুর পপের জন্য:**

```
VLAN 10:
  Name: MGMT_MIRPUR
  Status: Active
  Location: Mirpur POP
  Description: Management VLAN for network devices

VLAN 20:
  Name: UPLINK_MIRPUR
  Status: Active
  Location: Mirpur POP
  Description: Uplink to BTCL

VLAN 100:
  Name: RESIDENTIAL_MIR
  Status: Active
  Location: Mirpur POP
  Description: Residential customer traffic
```

**কল্যাণপুরের জন্য (একই ভিল্যান আইডি কিন্তু আলাদা লোকেশন):**

```
VLAN 10:
  Name: MGMT_KALYANPUR
  Status: Active
  Location: Kalyanpur POP

VLAN 21:
  Name: UPLINK_KALYANPUR
  Status: Active
  Location: Kalyanpur POP

VLAN 101:
  Name: RESIDENTIAL_KAL
  Status: Active
  Location: Kalyanpur POP
```

**শেয়ারড ভিল্যান (উভয় সাইটে):**

```
VLAN 200:
  Name: CORPORATE
  Status: Active
  Description: Corporate client VLAN (across all sites)
```

### দিন ১৭: IP Address Assignment

**কোর রাউটার (মিরপুর):**

Loopback:
```
IP Address: 10.10.1.1/32
Status: Active
Role: Loopback
Assigned to: R-MIR-CORE-01, Interface: lo0
DNS Name: r-mir-core-01.skynet.bd
Description: Management loopback
```

Uplink:
```
IP Address: 103.125.42.130/30
Status: Active
Role: Infrastructure
Assigned to: R-MIR-CORE-01, Interface: sfp-sfpplus1
DNS Name: r-mir-core-01-uplink.skynet.bd
Description: BTCL uplink IP
```

Management:
```
IP Address: 10.10.10.1/24
Status: Active
Role: Secondary
Assigned to: R-MIR-CORE-01, Interface: ether1
Description: Management interface
```

**ডিস্ট্রিবিউশন সুইচ:**

SW-MIR-DIST-01:
```
IP Address: 10.10.10.11/24
Status: Active
Assigned to: SW-MIR-DIST-01, Interface: vlan10
DNS Name: sw-mir-dist-01.skynet.bd
```

SW-MIR-DIST-02:
```
IP Address: 10.10.10.12/24
Status: Active
Assigned to: SW-MIR-DIST-02, Interface: vlan10
DNS Name: sw-mir-dist-02.skynet.bd
```

**এক্সেস সুইচ (১০টা):**

CSV দিয়ে করা যায়:

```csv
address,status,assigned_object_type,assigned_object,dns_name,description
10.10.10.21/24,Active,dcim.interface,SW-MIR-ACC-01:vlan10,sw-mir-acc-01.skynet.bd,Management IP
10.10.10.22/24,Active,dcim.interface,SW-MIR-ACC-02:vlan10,sw-mir-acc-02.skynet.bd,Management IP
10.10.10.23/24,Active,dcim.interface,SW-MIR-ACC-03:vlan10,sw-mir-acc-03.skynet.bd,Management IP
10.10.10.24/24,Active,dcim.interface,SW-MIR-ACC-04:vlan10,sw-mir-acc-04.skynet.bd,Management IP
10.10.10.25/24,Active,dcim.interface,SW-MIR-ACC-05:vlan10,sw-mir-acc-05.skynet.bd,Management IP
10.10.10.26/24,Active,dcim.interface,SW-MIR-ACC-06:vlan10,sw-mir-acc-06.skynet.bd,Management IP
10.10.10.27/24,Active,dcim.interface,SW-MIR-ACC-07:vlan10,sw-mir-acc-07.skynet.bd,Management IP
10.10.10.28/24,Active,dcim.interface,SW-MIR-ACC-08:vlan10,sw-mir-acc-08.skynet.bd,Management IP
10.10.10.29/24,Active,dcim.interface,SW-MIR-ACC-09:vlan10,sw-mir-acc-09.skynet.bd,Management IP
10.10.10.30/24,Active,dcim.interface,SW-MIR-ACC-10:vlan10,sw-mir-acc-10.skynet.bd,Management IP
```

### সপ্তাহ ৪: কল্যাণপুর পপ (দ্রুততর)

মিরপুর পপ কমপ্লিট হয়ে গেছে। এখন কল্যাণপুর পপ করতে অনেক কম সময় লাগবে, কারণ আমরা জানি কী করতে হবে।

### দিন ১৮-১৯: ডিভাইস এবং ক্যাবল

**Summit Provider এবং Circuit:**

```
Provider: Summit Communications
ASN: 45928
Circuit ID: SUMMIT-KAL-001
Commit Rate: 3000 Mbps
Monthly Cost: BDT 250,000
```

**ডিভাইস:**

```
R-KAL-CORE-01 (MikroTik CCR2004)
SW-KAL-DIST-01 (TP-Link TL-SG3428)
SW-KAL-ACC-01 to SW-KAL-ACC-06 (TP-Link TL-SG1024D)
```

CSV দিয়ে সব ডিভাইস একসাথে ইমপোর্ট করুন।

**Cable Connections:**

একইভাবে সব কানেকশন ডকুমেন্ট করুন।

### দিন ২০-২১: IPAM এবং ফাইনাল চেক

কল্যাণপুরের জন্য আইপি অ্যাসাইন করুন:

```
R-KAL-CORE-01 Loopback: 10.10.2.1/32
R-KAL-CORE-01 Uplink: 103.125.42.134/30
Management Network: 10.10.10.50-60 range
```

সব করা হয়ে গেলে একবার পুরো সেটআপ রিভিউ করুন।

### Custom Fields যোগ করা

SkyNet-এর জন্য কিছু দরকারি Custom Fields:

### Warranty Tracking

```
Content Type: dcim | device
Label: Warranty Expiry Date
Key: warranty_expiry
Type: Date
Description: Device warranty expiration date
```

এখন প্রতিটা ডিভাইসে ওয়ারেন্টি এক্সপায়ারি ডেট এন্ট্রি করুন। তাহলে একটা রিপোর্ট চালিয়ে দেখতে পারবেন কোন কোন ডিভাইসের ওয়ারেন্টি শীঘ্রই শেষ হবে।

### Building Information

```
Content Type: dcim | device
Label: Building Name
Key: building_name
Type: Text
Description: Building where device is located
```

এক্সেস সুইচগুলোতে বিল্ডিং নাম দিন:
- SW-MIR-ACC-01: Building-A
- SW-MIR-ACC-02: Building-B
- ইত্যাদি

### Customer Count

```
Content Type: dcim | device
Label: Approximate Customers
Key: customer_count
Type: Integer
Description: Approximate number of customers served by this device
```

এক্সেস সুইচে কাস্টমার কাউন্ট দিন:

- SW-MIR-ACC-01: 280
- SW-MIR-ACC-02: 310
- ইত্যাদি

এতে ক্যাপাসিটি প্ল্যানিং সহজ হবে।

### User এবং Permission সেটআপ

SkyNet-এর তিনজন টেকনিশিয়ান আছে:

**১. জাহাঙ্গীর (সিনিয়র নেটওয়ার্ক এডমিন):**

```
Username: jahangir
Group: Network Administrators
Permissions: প্রায় সব (শুধু ইউজার ম্যানেজমেন্ট বাদে)
```

**২. আসিফ (জুনিয়র টেকনিশিয়ান):**

```
Username: asif
Group: Network Operators
Permissions: View সব, Edit/Add ডিভাইস এবং আইপি, কিন্তু Delete করতে পারবে না
```

**৩. রফিক (NOC অপারেটর):**

```
Username: rafiq
Group: Read-Only Users
Permissions: শুধু View
```

এভাবে টিমের প্রত্যেকের দায়িত্ব অনুযায়ী অ্যাক্সেস দিন।

### ডেটা ভ্যালিডেশন এবং ক্লিনআপ

সব ডেটা এন্ট্রি শেষ হলে ভ্যালিডেশন করুন:

### চেকলিস্ট:

**১. সব ডিভাইসের নাম নেমিং কনভেনশন মেনেছে কিনা:**

Devices লিস্ট দেখুন। সব নাম `[Type]-[Site]-[Role]-[Number]` ফরম্যাটে আছে তো?

**২. সব ডিভাইসের স্ট্যাটাস সঠিক:**

যেগুলো চালু আছে সেগুলো Active, যেগুলো অফলাইন সেগুলো Offline।

**৩. সব ইন্টারফেসে ডেসক্রিপশন আছে:**

গুরুত্বপূর্ণ ইন্টারফেসগুলোতে (আপলিংক, ডিস্ট্রিবিউশন লিংক) পরিষ্কার ডেসক্রিপশন দেওয়া আছে তো?

**৪. সব আইপি সঠিক রেঞ্জে:**

103.125.40.0/22 ব্লকের বাইরে কোনো আইপি দেওয়া হয়নি তো?

**৫. সব ক্যাবল সঠিকভাবে কানেক্টেড:**

একটা দুটো ডিভাইসে Trace চালিয়ে দেখুন পুরো পাথ ঠিকমতো দেখাচ্ছে কিনা।

**৬. DNS নাম ডুপ্লিকেট নেই:**

IP Addresses লিস্ট এক্সপোর্ট করে চেক করুন কোনো DNS নাম দুইবার ব্যবহার করা হয়নি তো।

### ডুপ্লিকেট চেক করার স্ক্রিপ্ট

একটা সিম্পল Python স্ক্রিপ্ট লিখতে পারেন যা CSV এক্সপোর্ট চেক করবে:

```python
import csv
from collections import Counter

# Devices CSV এক্সপোর্ট করুন
with open('devices.csv', 'r') as f:
    reader = csv.DictReader(f)
    names = [row['name'] for row in reader]
    
# ডুপ্লিকেট খুঁজুন
duplicates = [name for name, count in Counter(names).items() if count > 1]

if duplicates:
    print("ডুপ্লিকেট পাওয়া গেছে:")
    for dup in duplicates:
        print(f"  - {dup}")
else:
    print("কোনো ডুপ্লিকেট নেই। ভালো!")
```

### ডকুমেন্টেশন তৈরি করা

Nautobot-এ সব ডেটা আছে, কিন্তু একটা সিম্পল ডকুমেন্টও বানান যেটা প্রিন্ট করা যায়।

### নেটওয়ার্ক ওভারভিউ ডকুমেন্ট

একটা ওয়ার্ড ডকুমেন্ট বা PDF বানান:

```
SkyNet Bangladesh - Network Documentation
Generated: [Date]

1. Network Overview
   - Total Sites: 2
   - Total Devices: 19
   - Total Customers: ~5000

2. Sites
   - Mirpur POP
     * Address: House 25, Road 10, Mirpur-12
     * Devices: 13
     * Customers: ~3000
     * Uplink: BTCL 5Gbps
   
   - Kalyanpur POP
     * Address: Plot 15, Road 3, Kalyanpur
     * Devices: 7
     * Customers: ~2000
     * Uplink: Summit 3Gbps

3. IP Resources
   - Public: 103.125.40.0/22
   - Private: 10.10.0.0/16

4. Critical Contacts
   - Mirpur: Jahangir Khan (+880 1712-345678)
   - Kalyanpur: Asif Rahman (+880 18xx-234567)

5. Nautobot Access
   - URL: https://nautobot.skynet.bd
   - Admin: [credentials stored securely]
```

এই ডকুমেন্ট প্রিন্ট করে অফিসে রাখুন। ইমার্জেন্সিতে কাজে লাগবে।

### ৩ মাসের রোডম্যাপ

এখন পর্যন্ত আমরা বেসিক সেটআপ করলাম। পরের তিন মাসে ধাপে ধাপে উন্নত করুন:

### মাস ১ (সম্পন্ন): বেসিক সেটআপ

- সব সাইট, ডিভাইস, আইপি এন্ট্রি
- ক্যাবল কানেকশন ডকুমেন্ট
- বেসিক ইউজার সেটআপ

### মাস ২: অপটিমাইজেশন

- সব ডিভাইসে Interface Description আপডেট করুন
- Custom Fields সব ডিভাইসে পূরণ করুন
- Tags সিস্টেমেটিক্যালি ব্যবহার করুন
- Saved Filters তৈরি করুন কমন সার্চের জন্য
- Documentation আপডেট রাখার নিয়ম তৈরি করুন

### মাস ৩: অটোমেশন শুরু

- Python PyNautobot শিখুন
- সিম্পল রিপোর্ট স্ক্রিপ্ট লিখুন
- CSV এক্সপোর্ট অটোমেট করুন
- Backup স্ক্রিপ্ট ইমপ্রুভ করুন
- API ইউজ করে ডেটা কোয়েরি করুন

## টিম ট্রেনিং

সবাই Nautobot ভালোমতো ব্যবহার করতে পারে এটা নিশ্চিত করুন।

### সপ্তাহ ১: অরিয়েন্টেশন

- সবাইকে Nautobot দেখান
- কীভাবে লগইন করতে হয়
- কীভাবে সার্চ করতে হয়
- কীভাবে ডিভাইস খুঁজে পেতে হয়

### সপ্তাহ ২-৩: হ্যান্ডস-অন ট্রেনিং

- প্রতিটা টিম মেম্বারকে একটা করে সাইট দায়িত্ব দিন
- তাদের বলুন নতুন কোনো ডিভাইস এড করতে
- ভুল করলে কী হয় দেখান (তারপর আনডু করুন)
- কীভাবে রিপোর্ট এক্সপোর্ট করতে হয় শেখান

### সপ্তাহ ৪: সাপোর্ট এবং Q&A

- যেকোনো প্রশ্ন উত্তর দিন
- কমন সমস্যাগুলোর সমাধান দেখান
- Troubleshooting টিপস শেয়ার করুন

### কমন সমস্যা এবং সমাধান

### সমস্যা ১: "আমি CSV ইমপোর্ট করতে পারছি না"

**সমাধান:** 

- CSV ফাইল UTF-8 এনকোডিংয়ে সেভ করেছেন তো?
- সব রিকোয়ার্ড ফিল্ড আছে তো?
- Device Type, Location, Role - এসব Nautobot-এ আগে থেকে আছে তো?

### সমস্যা ২: "Interface এ IP অ্যাসাইন করতে পারছি না"

**সমাধান:**

- Interface টা Enabled আছে তো?
- IP টা সঠিক ফরম্যাটে দিয়েছেন তো? (যেমন: 10.10.1.1/24)
- Prefix আগে তৈরি করেছেন তো?

### সমস্যা ৩: "Cable Trace কাজ করছে না"

**সমাধান:**

- উভয় প্রান্তের ক্যাবল সঠিকভাবে টার্মিনেট করেছেন তো?
- Cable Status "Connected" আছে তো?

### ফাইনাল ফাইনাল চেকলিস্ট

তিন মাস পরে এই জিনিসগুলো হওয়া উচিত:

 **সব ডিভাইস ডকুমেন্টেড:** একটা ডিভাইসও বাদ নেই

 **সব কানেকশন ম্যাপড:** কোন তার কোথায় গেছে সব জানা

 **সব আইপি ট্র্যাক করা:** কোন আইপি কোথায় ইউজ হচ্ছে সব রেকর্ড

 **টিম ট্রেইনড:** সবাই Nautobot স্বাচ্ছন্দ্যে ব্যবহার করতে পারে

 **ডেইলি আপডেট:** নতুন ডিভাইস যোগ হলে সেদিনই Nautobot-এ এড হয়

 **রিপোর্টিং চালু:** মাসিক রিপোর্ট Nautobot থেকে জেনারেট হয়

এই সব হয়ে গেলে বুঝবেন SkyNet Bangladesh একটা প্রপার NSoT ইমপ্লিমেন্ট করে ফেলেছে। এখন তারা রেডি পরের লেভেলে যাওয়ার জন্য - ৫০ হাজার কাস্টমারে স্কেল করা।

পরের চ্যাপ্টারে আমরা দেখব মিডিয়াম আইএসপি (৫০ হাজার কাস্টমার) কীভাবে Nautobot ম্যানেজ করে, কী কী নতুন চ্যালেঞ্জ আসে, আর কীভাবে সেগুলো সামলানো যায়।