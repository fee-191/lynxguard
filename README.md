# LynxGuard

> **Đánh giá lỗ hổng và Giám sát bảo mật cho ứng dụng web**  
> Đồ án tốt nghiệp (IAP491) · Đại học FPT, Hoa Lạc · 09–12/2024  
> Nhóm 4 người · Được hội đồng vinh danh trên trang trường

📽️ **[Xem demo video](https://github.com/fee-191/lynxguard/releases/download/v1.0/lynxguard-demo.mp4)** · 📄 **[Slides (PDF)](https://github.com/fee-191/lynxguard/releases/download/v1.0/lynxguard-slides.pdf)**

---

## Đặt vấn đề

Các ứng dụng web của trường thiếu hệ thống giám sát bảo mật liên tục. Cách tiếp cận hiện tại dựa trên kiểm tra thủ công và các tool riêng lẻ — không có pipeline tập trung để phát hiện, tương quan và cảnh báo tấn công web theo thời gian thực. Các tấn công như SQLi, XSS, brute force và DoS có thể không bị phát hiện trong nhiều ngày.

LynxGuard giải quyết vấn đề này bằng cách tích hợp nhiều open-source security tools thành một SOC pipeline thống nhất: từ phát hiện ở tầng mạng (Snort), lọc ở tầng ứng dụng (ModSecurity), tập hợp và tương quan log tập trung (Wazuh SIEM), giám sát hạ tầng (Zabbix), đến cảnh báo real-time (Telegram bot).

---

## Kiến trúc

```
                        Internet Traffic
                              │
                              ▼
               ┌──────────────────────────┐
               │      pfSense Firewall     │
               │  NAT · Routing · Zones    │
               └──────────┬───────────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
     ┌─────────────┐ ┌──────────┐ ┌──────────────┐
     │    Snort    │ │  Apache  │ │    Zabbix    │
     │  IDS/IPS   │ │    +     │ │  Monitoring  │
     │  (Network) │ │ModSecurity│ │  (Infra)    │
     └──────┬──────┘ │  (WAF)  │ └──────┬───────┘
            │        └────┬────┘        │
            │  Logs/Events │             │
            └──────────────┼────────────┘
                           │  syslog / agents
                           ▼
              ┌────────────────────────┐
              │      Wazuh Manager     │
              │  Tập hợp log           │
              │  Correlation rules     │
              │  Threat detection      │
              │  Sinh cảnh báo         │
              └────────────┬───────────┘
                           │
              ┌────────────┼──────────────┐
              ▼            ▼              ▼
      ┌──────────────┐ ┌────────┐ ┌─────────────┐
      │Wazuh Dashboard│ │ Active │ │Telegram Bot │
      │(Kibana-based) │ │Response│ │  (Python)   │
      └──────────────┘ └────────┘ └─────────────┘
```

**Môi trường:** Ảo hóa hoàn toàn trên VMware Workstation. Tất cả components chạy trên các VM riêng biệt trên một host, với các virtual network segment mô phỏng môi trường enterprise thực tế.

| VM | OS | Vai trò |
|----|-----|---------|
| Gateway | pfSense 2.7 | Firewall, routing, NAT |
| Web Server | Ubuntu 22.04 | Apache + ModSecurity (WAF) |
| IDS | Ubuntu 22.04 | Snort 3 (network IDS) |
| SIEM | Ubuntu 22.04 | Wazuh Manager + Dashboard |
| Monitoring | Ubuntu 22.04 | Zabbix Server |
| Client | Windows 10 | Mô phỏng tấn công + testing |

---

## Công nghệ sử dụng

| Tầng | Tool | Phiên bản | Vai trò |
|------|------|-----------|---------|
| Firewall | pfSense | 2.7 | Kiểm soát perimeter, routing giữa các zone |
| IDS/IPS | Snort | 3.x | Phát hiện xâm nhập dựa trên signature |
| WAF | ModSecurity | 3.x | OWASP Core Rule Set, lọc tầng ứng dụng |
| SIEM | Wazuh | 4.x | Tập hợp log, correlation, phát hiện mối đe dọa |
| Monitoring | Zabbix | 6.x | Giám sát hạ tầng, cảnh báo theo ngưỡng |
| Vuln Assessment | Acunetix | — | Quét lỗ hổng web tự động |
| Pentesting | BurpSuite | Community | Kiểm thử thủ công |
| Alerting | Telegram Bot | Python | Cảnh báo real-time |

---

## Luồng dữ liệu

```
Phát hiện tấn công
    │
    ├─▶ Snort (tầng mạng)
    │       └─▶ Sinh alert → syslog → Wazuh agent
    │
    ├─▶ ModSecurity (tầng ứng dụng)
    │       └─▶ Ghi vào Apache error log → Wazuh agent
    │
    ├─▶ pfSense (firewall)
    │       └─▶ syslog chuyển tiếp về Wazuh Manager
    │
    └─▶ Zabbix (giám sát hạ tầng)
            └─▶ Trigger action → thông báo → Telegram

Tất cả nguồn ──▶ Wazuh Manager
                    ├─▶ Tương quan events từ nhiều nguồn
                    ├─▶ Áp dụng detection rules
                    ├─▶ Sinh unified alert
                    └─▶ Telegram Bot (events mức độ cao)
```

---

## Phạm vi phát hiện

| Loại tấn công | Phát hiện bởi | Phương pháp |
|--------------|---------------|-------------|
| SQL Injection | ModSecurity + Snort | OWASP CRS rules + Snort signatures |
| XSS | ModSecurity | OWASP CRS rules |
| Port Scanning | Snort | Threshold-based signature |
| Brute Force | Wazuh | Log correlation — tần suất failed auth |
| DoS / Lưu lượng cao | Zabbix + pfSense | Ngưỡng băng thông + packet rate |
| Directory Traversal | ModSecurity | OWASP CRS rules |
| Command Injection | ModSecurity | OWASP CRS rules |
| Protocol anomalies | Snort | Anomaly-based rules |

---

## Phương pháp kiểm thử

Kiểm thử được thực hiện qua 3 giai đoạn:

**Phase 1 — Quét lỗ hổng tự động**  
Acunetix quét môi trường web application để thiết lập baseline lỗ hổng trước khi triển khai monitoring stack.

**Phase 2 — Tấn công mô phỏng có kiểm soát**  
BurpSuite và kiểm thử thủ công được dùng để thực thi các tấn công (SQLi, XSS, brute force, directory traversal) để verify phát hiện và cảnh báo.

**Phase 3 — Phân tích log và tuning**  
Phân tích Wazuh alerts và Snort/ModSecurity logs từ Phase 2 để xác định false positive, tune rules và validate end-to-end từ tấn công đến Telegram notification.

---

## Đóng góp cá nhân

Làm việc trong nhóm 4 người, tôi phụ trách:

- **Dựng môi trường VMware:** Cài đặt và cấu hình tất cả VM — cài OS, gán network adapter, quản lý snapshot cho repeatable testing
- **Thiết kế mạng:** Thiết kế virtual network topology với các segment riêng biệt (WAN, LAN, DMZ zones) mô phỏng môi trường enterprise
- **Cấu hình pfSense:** Firewall rules, NAT policies, inter-zone routing, syslog forwarding về Wazuh
- **Telegram bot (Python):** Viết alerting bot nhận webhooks từ Zabbix và Wazuh, lọc theo severity và gửi notification có định dạng về Telegram group

```python
# Alert handler — lọc severity và định dạng message
def handle_alert(alert: dict) -> None:
    severity = alert.get("rule", {}).get("level", 0)
    if severity < ALERT_THRESHOLD:   # bỏ qua alert mức thấp
        return

    source = alert.get("agent", {}).get("name", "unknown")
    rule   = alert.get("rule", {}).get("description", "")
    time   = alert.get("timestamp", "")

    msg = (
        f"🚨 *Security Alert*\n"
        f"*Source:* `{source}`\n"
        f"*Rule:* {rule}\n"
        f"*Severity:* {severity}/15\n"
        f"*Time:* {time}"
    )
    bot.send_message(chat_id=CHAT_ID, text=msg, parse_mode="Markdown")
```

---

## Kết quả chính

- **Hiệu quả WAF:** ModSecurity với OWASP CRS block 100% automated SQLi/XSS từ Acunetix scanner
- **False positive của IDS:** Snort sinh nhiều false positive với HTTPS traffic thông thường — cần tuning rule đáng kể trước khi dùng thực tế
- **Giá trị của log correlation:** Wazuh tương quan thành công events từ ModSecurity, Snort và pfSense để phát hiện multi-stage attack; không có correlation, cùng một tấn công xuất hiện thành 3+ alert riêng biệt trên từng tool dashboard
- **End-to-end latency:** Thời gian trung bình từ lúc tấn công đến Telegram notification: ~8 giây

---

## Hạn chế

- Môi trường ảo hóa hoàn toàn — không capture được đặc trưng performance thực tế (hardware resource contention, physical network latency)
- Wazuh rule set dùng default rules + tuning tối thiểu; triển khai production cần customize đáng kể cho application stack cụ thể
- Phạm vi kiểm thử giới hạn ở simulated attacks trong môi trường kiểm soát — attacker behavior thực tế (evasion, slow-and-low techniques) chưa được test kỹ
- Snort false positive rate trong HTTPS-heavy environment cần tuning dài hạn, chưa được giải quyết trong phạm vi đồ án

---

## Tài liệu tham khảo

- [OWASP ModSecurity Core Rule Set](https://owasp.org/www-project-modsecurity-core-rule-set/)
- [Snort 3 Documentation](https://www.snort.org/documents)
- [Wazuh Documentation](https://documentation.wazuh.com/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
