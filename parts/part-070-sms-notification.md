# Part 70: SMS Notification และ Multi-Channel Alerts

## สารบัญ
1. [SMS Gateway Integration](#sms-gateway)
2. [Trigger-based SMS](#triggers)
3. [Two-way SMS](#two-way)
4. [SMS Templates](#templates)
5. [Alert Escalation](#escalation)
6. [On-call Rotation](#on-call)
7. [SMS Logging](#logging)
8. [Cost Optimization](#cost)
9. [LINE Notify & Telegram](#line-telegram)
10. [Lab: Multi-channel Alert System](#lab)

---

## 1. SMS Gateway Integration {#sms-gateway}

### Thai SMS Services

| Provider | ราคา | Features | API |
|---------|------|---------|-----|
| Twilio | ~฿1.2/SMS | Global, reliable | REST API |
| ThaiSMS | ~฿0.30/SMS | ราคาถูก, ไทยเท่านั้น | REST API |
| DTAC/AIS API | ~฿0.25/SMS | Enterprise | REST API |
| SMSmanager.co.th | ~฿0.28/SMS | Thai focused | REST API |

```python
# app/services/notifications/sms_service.py
from abc import ABC, abstractmethod
from typing import Optional, List, Dict
import httpx
import logging
import json
from datetime import datetime
from sqlalchemy.orm import Session

logger = logging.getLogger(__name__)


class SMSProvider(ABC):
    """Base class สำหรับ SMS providers"""
    
    @abstractmethod
    async def send_sms(self, to: str, message: str) -> Dict:
        pass
    
    @abstractmethod
    def get_provider_name(self) -> str:
        pass


class TwilioSMSProvider(SMSProvider):
    """Twilio SMS Provider"""
    
    def __init__(self, account_sid: str, auth_token: str, from_number: str):
        self.account_sid = account_sid
        self.auth_token = auth_token
        self.from_number = from_number
        self.base_url = f"https://api.twilio.com/2010-04-01/Accounts/{account_sid}/Messages.json"
    
    def get_provider_name(self) -> str:
        return "twilio"
    
    async def send_sms(self, to: str, message: str) -> Dict:
        """ส่ง SMS ผ่าน Twilio"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.base_url,
                auth=(self.account_sid, self.auth_token),
                data={
                    "From": self.from_number,
                    "To": to,
                    "Body": message
                }
            )
            
            result = response.json()
            
            return {
                "success": result.get("status") not in ["failed", "undelivered"],
                "message_id": result.get("sid"),
                "status": result.get("status"),
                "provider": self.get_provider_name(),
                "raw_response": result
            }


class ThaiSMSProvider(SMSProvider):
    """Thai SMS Service Provider"""
    
    def __init__(self, api_key: str, sender: str):
        self.api_key = api_key
        self.sender = sender
        self.base_url = "https://api.thaismsservice.com/v2/send"
    
    def get_provider_name(self) -> str:
        return "thaisms"
    
    async def send_sms(self, to: str, message: str) -> Dict:
        """ส่ง SMS ผ่าน ThaiSMS"""
        # Normalize phone number to Thai format
        to = self._normalize_phone(to)
        
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.base_url,
                json={
                    "key": self.api_key,
                    "to": to,
                    "from": self.sender,
                    "message": message
                }
            )
            
            result = response.json()
            
            return {
                "success": result.get("code") == "0",
                "message_id": result.get("message_id"),
                "status": "sent" if result.get("code") == "0" else "failed",
                "provider": self.get_provider_name(),
                "error": result.get("message") if result.get("code") != "0" else None
            }
    
    def _normalize_phone(self, phone: str) -> str:
        """แปลงเบอร์ไทยเป็นรูปแบบมาตรฐาน (+66)"""
        phone = phone.replace("-", "").replace(" ", "")
        if phone.startswith("0"):
            phone = "+66" + phone[1:]
        elif not phone.startswith("+"):
            phone = "+66" + phone
        return phone


class SMSService:
    """Service จัดการการส่ง SMS"""
    
    def __init__(self, provider: SMSProvider, db: Session, redis_client=None):
        self.provider = provider
        self.db = db
        self.redis = redis_client
        self._rate_limit_window = 60  # 1 minute
        self._rate_limit_max = 10     # 10 SMS per minute per number
    
    async def send(
        self,
        to: str,
        message: str,
        template_name: str = None,
        metadata: Dict = None
    ) -> Dict:
        """ส่ง SMS พร้อม logging และ rate limiting"""
        
        # Rate limit check
        if not self._check_rate_limit(to):
            logger.warning(f"Rate limit exceeded for {to}")
            return {"success": False, "error": "rate_limit_exceeded"}
        
        # Validate message length (SMS = 160 chars, Thai = 70 chars)
        if len(message.encode('utf-8')) > 160:
            logger.warning(f"SMS message too long: {len(message)} chars")
        
        try:
            result = await self.provider.send_sms(to, message)
            
            # Log to database
            self._log_sms(
                to=to,
                message=message,
                result=result,
                template_name=template_name,
                metadata=metadata
            )
            
            return result
        except Exception as e:
            logger.error(f"SMS send failed: {e}")
            self._log_sms(
                to=to,
                message=message,
                result={"success": False, "error": str(e)},
                template_name=template_name
            )
            return {"success": False, "error": str(e)}
    
    def _check_rate_limit(self, to: str) -> bool:
        """ตรวจสอบ rate limit"""
        if not self.redis:
            return True
        
        key = f"sms_rate:{to}"
        count = self.redis.sync_client.incr(key)
        
        if count == 1:
            self.redis.sync_client.expire(key, self._rate_limit_window)
        
        return count <= self._rate_limit_max
    
    def _log_sms(self, to: str, message: str, result: Dict, 
                  template_name: str = None, metadata: Dict = None):
        """บันทึก SMS log"""
        from sqlalchemy import text
        
        try:
            self.db.execute(text("""
                INSERT INTO sms_log 
                    (recipient, message, template_name, provider, status, 
                     message_id, metadata, sent_at)
                VALUES 
                    (:to, :message, :template, :provider, :status,
                     :msg_id, :metadata::JSONB, NOW())
            """), {
                "to": to,
                "message": message,
                "template": template_name,
                "provider": result.get("provider"),
                "status": "sent" if result.get("success") else "failed",
                "msg_id": result.get("message_id"),
                "metadata": json.dumps(metadata or {})
            })
            self.db.commit()
        except Exception as e:
            logger.error(f"Failed to log SMS: {e}")
```

---

## 2. SMS Templates {#templates}

```python
# app/services/notifications/sms_templates.py
from typing import Dict, Optional
from jinja2 import Template
import logging

logger = logging.getLogger(__name__)


class SMSTemplateManager:
    """จัดการ SMS templates"""
    
    TEMPLATES = {
        # Billing templates
        "invoice_due": {
            "th": "ใบแจ้งหนี้ {{ invoice_no }} จำนวน ฿{{ amount }} ครบกำหนด {{ due_date }} กรุณาชำระด่วน",
            "en": "Invoice {{ invoice_no }} of ฿{{ amount }} due on {{ due_date }}. Please pay promptly."
        },
        "payment_received": {
            "th": "รับชำระเงินแล้ว ฿{{ amount }} สำหรับใบแจ้งหนี้ {{ invoice_no }} ขอบคุณ",
            "en": "Payment received: ฿{{ amount }} for invoice {{ invoice_no }}. Thank you."
        },
        "service_suspended": {
            "th": "บริการอินเทอร์เน็ตของท่านถูกระงับชั่วคราว เนื่องจากค้างชำระ กรุณาติดต่อ {{ contact }}",
            "en": "Your internet service has been suspended due to non-payment. Contact {{ contact }}"
        },
        "service_activated": {
            "th": "บริการอินเทอร์เน็ตของท่านเปิดใช้งานแล้ว ยินดีต้อนรับกลับมา",
            "en": "Your internet service has been activated. Welcome back!"
        },
        
        # Network alert templates
        "router_down": {
            "th": "แจ้งเตือน: Router {{ router_name }} ({{ ip }}) ไม่ตอบสนอง เวลา {{ time }}",
            "en": "Alert: Router {{ router_name }} ({{ ip }}) is not responding at {{ time }}"
        },
        "high_cpu": {
            "th": "แจ้งเตือน: CPU ของ {{ router_name }} สูงถึง {{ cpu_percent }}% เวลา {{ time }}",
            "en": "Alert: {{ router_name }} CPU at {{ cpu_percent }}% at {{ time }}"
        },
        "bandwidth_exceeded": {
            "th": "แจ้งเตือน: Bandwidth ของ {{ interface }} เกิน {{ threshold }}% เวลา {{ time }}",
            "en": "Alert: {{ interface }} bandwidth exceeded {{ threshold }}% at {{ time }}"
        },
        
        # Customer service templates
        "otp": {
            "th": "รหัส OTP ของท่านคือ {{ otp }} มีอายุ {{ expire_minutes }} นาที",
            "en": "Your OTP is {{ otp }}. Valid for {{ expire_minutes }} minutes."
        },
        "password_reset": {
            "th": "รหัสรีเซ็ตรหัสผ่าน: {{ code }} หมดอายุใน {{ expire_minutes }} นาที",
            "en": "Password reset code: {{ code }}. Expires in {{ expire_minutes }} minutes."
        }
    }
    
    def render(
        self,
        template_name: str,
        variables: Dict,
        language: str = "th"
    ) -> str:
        """Render template ด้วย variables"""
        if template_name not in self.TEMPLATES:
            raise ValueError(f"Template '{template_name}' not found")
        
        templates = self.TEMPLATES[template_name]
        template_str = templates.get(language) or templates.get("en", "")
        
        if not template_str:
            raise ValueError(f"No template for language '{language}'")
        
        template = Template(template_str)
        rendered = template.render(**variables)
        
        # ตรวจสอบความยาว
        if len(rendered) > 160:
            logger.warning(f"SMS template '{template_name}' too long: {len(rendered)} chars")
        
        return rendered
    
    def add_custom_template(
        self,
        name: str,
        th_template: str,
        en_template: str = None
    ):
        """เพิ่ม custom template"""
        self.TEMPLATES[name] = {
            "th": th_template,
            "en": en_template or th_template
        }
    
    def list_templates(self) -> List[str]:
        return list(self.TEMPLATES.keys())


# Singleton
template_manager = SMSTemplateManager()


# Usage example
async def send_invoice_reminder(
    sms_service: SMSService,
    phone: str,
    invoice_no: str,
    amount: float,
    due_date: str
):
    message = template_manager.render(
        "invoice_due",
        {
            "invoice_no": invoice_no,
            "amount": f"{amount:,.2f}",
            "due_date": due_date
        },
        language="th"
    )
    
    return await sms_service.send(
        to=phone,
        message=message,
        template_name="invoice_due"
    )
```

---

## 3. Alert Escalation {#escalation}

```python
# app/services/notifications/escalation.py
from datetime import datetime, timedelta
from typing import List, Dict, Optional
import asyncio
import logging

logger = logging.getLogger(__name__)


class EscalationPolicy:
    """กำหนด escalation policy"""
    
    def __init__(self, levels: List[Dict]):
        """
        levels: [
            {
                "delay_minutes": 0,
                "contacts": [{"name": "NOC", "phone": "0801234567"}],
                "channels": ["sms", "line"]
            },
            {
                "delay_minutes": 15,
                "contacts": [{"name": "Mgr", "phone": "0812345678"}],
                "channels": ["sms", "call"]
            }
        ]
        """
        self.levels = levels


class AlertEscalationService:
    """จัดการ alert escalation"""
    
    def __init__(self, sms_service, line_service, redis_client):
        self.sms = sms_service
        self.line = line_service
        self.redis = redis_client
    
    async def escalate_alert(
        self,
        alert_id: str,
        alert_data: Dict,
        policy: EscalationPolicy
    ):
        """Escalate alert ตาม policy"""
        
        escalation_key = f"escalation:{alert_id}"
        
        # ตรวจสอบว่า alert ถูก acknowledged แล้วหรือยัง
        if self.redis.exists(f"ack:{alert_id}"):
            logger.info(f"Alert {alert_id} already acknowledged, skipping escalation")
            return
        
        for level_idx, level in enumerate(policy.levels):
            delay_minutes = level.get("delay_minutes", 0)
            
            # รอตาม delay
            if delay_minutes > 0:
                await asyncio.sleep(delay_minutes * 60)
            
            # ตรวจสอบอีกครั้งว่ายัง active อยู่
            if self.redis.exists(f"ack:{alert_id}"):
                logger.info(f"Alert {alert_id} acknowledged during escalation level {level_idx}")
                return
            
            contacts = level.get("contacts", [])
            channels = level.get("channels", ["sms"])
            
            for contact in contacts:
                for channel in channels:
                    try:
                        await self._send_escalation(
                            contact=contact,
                            channel=channel,
                            alert_data=alert_data,
                            level=level_idx + 1
                        )
                    except Exception as e:
                        logger.error(f"Escalation send failed: {e}")
            
            # บันทึก escalation level
            self.redis.set(
                escalation_key,
                {"level": level_idx + 1, "time": datetime.utcnow().isoformat()},
                ttl=86400
            )
        
        logger.warning(f"Alert {alert_id} escalated through all levels without acknowledgment")
    
    async def _send_escalation(
        self,
        contact: Dict,
        channel: str,
        alert_data: Dict,
        level: int
    ):
        """ส่ง escalation notification"""
        severity = alert_data.get("severity", "unknown").upper()
        router = alert_data.get("router_name", "unknown")
        message_text = alert_data.get("message", "")
        
        message = f"[L{level}] {severity}: {router} - {message_text}"
        
        if channel == "sms":
            await self.sms.send(
                to=contact["phone"],
                message=message,
                template_name="escalation_alert"
            )
        elif channel == "line":
            await self.line.send_message(
                to=contact.get("line_id"),
                message=message
            )
    
    def acknowledge_alert(self, alert_id: str, acknowledged_by: str):
        """Mark alert as acknowledged - หยุด escalation"""
        self.redis.set(
            f"ack:{alert_id}",
            {"by": acknowledged_by, "at": datetime.utcnow().isoformat()},
            ttl=86400
        )
        logger.info(f"Alert {alert_id} acknowledged by {acknowledged_by}")
```

---

## 4. LINE Notify & Telegram {#line-telegram}

```python
# app/services/notifications/line_service.py
import httpx
from typing import Optional, Dict, List
import logging

logger = logging.getLogger(__name__)


class LINENotifyService:
    """LINE Notify Integration"""
    
    NOTIFY_URL = "https://notify-api.line.me/api/notify"
    
    def __init__(self, token: str):
        self.token = token
    
    async def send_message(self, message: str, image_url: str = None) -> Dict:
        """ส่ง LINE Notify message"""
        async with httpx.AsyncClient() as client:
            data = {"message": message}
            if image_url:
                data["imageThumbnail"] = image_url
                data["imageFullsize"] = image_url
            
            response = await client.post(
                self.NOTIFY_URL,
                headers={"Authorization": f"Bearer {self.token}"},
                data=data
            )
            
            result = response.json()
            return {
                "success": result.get("status") == 200,
                "message": result.get("message")
            }
    
    async def send_sticker(self, message: str, package_id: str, sticker_id: str) -> Dict:
        """ส่ง LINE Notify พร้อม sticker"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.NOTIFY_URL,
                headers={"Authorization": f"Bearer {self.token}"},
                data={
                    "message": message,
                    "stickerPackageId": package_id,
                    "stickerId": sticker_id
                }
            )
            return response.json()


class TelegramBotService:
    """Telegram Bot Integration"""
    
    def __init__(self, bot_token: str):
        self.bot_token = bot_token
        self.base_url = f"https://api.telegram.org/bot{bot_token}"
    
    async def send_message(
        self,
        chat_id: str,
        message: str,
        parse_mode: str = "Markdown"
    ) -> Dict:
        """ส่ง Telegram message"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/sendMessage",
                json={
                    "chat_id": chat_id,
                    "text": message,
                    "parse_mode": parse_mode
                }
            )
            
            result = response.json()
            return {
                "success": result.get("ok"),
                "message_id": result.get("result", {}).get("message_id")
            }
    
    async def send_document(
        self,
        chat_id: str,
        file_bytes: bytes,
        filename: str,
        caption: str = ""
    ) -> Dict:
        """ส่งไฟล์ผ่าน Telegram"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/sendDocument",
                data={"chat_id": chat_id, "caption": caption},
                files={"document": (filename, file_bytes)}
            )
            return response.json()


class MultiChannelNotificationService:
    """ส่ง notifications ผ่านหลาย channels"""
    
    def __init__(
        self,
        sms_service=None,
        line_service: LINENotifyService = None,
        telegram_service: TelegramBotService = None
    ):
        self.sms = sms_service
        self.line = line_service
        self.telegram = telegram_service
    
    async def send_alert(
        self,
        alert: Dict,
        channels: List[str] = None,
        recipients: Dict = None
    ):
        """ส่ง alert ผ่านหลาย channels"""
        if channels is None:
            channels = ["sms", "line", "telegram"]
        
        message = self._format_alert_message(alert)
        results = {}
        
        if "sms" in channels and self.sms and recipients:
            for phone in recipients.get("phones", []):
                result = await self.sms.send(to=phone, message=message)
                results[f"sms:{phone}"] = result
        
        if "line" in channels and self.line:
            line_message = f"🚨 *{alert.get('severity', '').upper()}*\n\n{message}"
            result = await self.line.send_message(line_message)
            results["line"] = result
        
        if "telegram" in channels and self.telegram and recipients:
            for chat_id in recipients.get("telegram_chats", []):
                telegram_msg = (
                    f"🚨 *{alert.get('severity', '').upper()} Alert*\n"
                    f"Router: `{alert.get('router_name', '')}`\n"
                    f"Message: {alert.get('message', '')}\n"
                    f"Time: {alert.get('timestamp', '')}"
                )
                result = await self.telegram.send_message(chat_id, telegram_msg)
                results[f"telegram:{chat_id}"] = result
        
        return results
    
    def _format_alert_message(self, alert: Dict) -> str:
        severity_emoji = {
            "critical": "🔴",
            "warning": "🟡",
            "info": "🔵",
            "resolved": "🟢"
        }
        emoji = severity_emoji.get(alert.get("severity", "info"), "⚪")
        
        return (
            f"{emoji} [{alert.get('severity', 'UNKNOWN').upper()}] "
            f"{alert.get('router_name', 'Unknown Router')}: "
            f"{alert.get('message', '')} "
            f"({alert.get('timestamp', '')})"
        )
```

---

## 5. Lab: Multi-channel Alert System {#lab}

### Complete Implementation

```python
# Example: Setup multi-channel alerts
import asyncio
from app.services.notifications.sms_service import SMSService, TwilioSMSProvider
from app.services.notifications.line_service import (
    LINENotifyService, TelegramBotService, MultiChannelNotificationService
)
from app.services.notifications.escalation import (
    AlertEscalationService, EscalationPolicy
)


async def setup_notification_system():
    # SMS via Twilio
    twilio = TwilioSMSProvider(
        account_sid="ACxxxxxxxx",
        auth_token="xxxxxxxx",
        from_number="+15551234567"
    )
    
    # LINE Notify
    line = LINENotifyService(token="your-line-notify-token")
    
    # Telegram
    telegram = TelegramBotService(bot_token="your-bot-token")
    
    # Multi-channel service
    notifier = MultiChannelNotificationService(
        line_service=line,
        telegram_service=telegram
    )
    
    # Escalation policy
    policy = EscalationPolicy(levels=[
        {
            "delay_minutes": 0,
            "contacts": [
                {"name": "NOC Team", "phone": "0801234567"}
            ],
            "channels": ["line", "telegram"]
        },
        {
            "delay_minutes": 15,
            "contacts": [
                {"name": "NOC Supervisor", "phone": "0812345678"}
            ],
            "channels": ["sms", "line"]
        },
        {
            "delay_minutes": 30,
            "contacts": [
                {"name": "IT Manager", "phone": "0823456789"}
            ],
            "channels": ["sms"]
        }
    ])
    
    # Test alert
    alert = {
        "id": "alert-001",
        "severity": "critical",
        "router_name": "Core Router 1",
        "message": "Router is not responding",
        "timestamp": "2024-01-01 10:00:00"
    }
    
    # ส่งผ่านทุก channels
    results = await notifier.send_alert(
        alert=alert,
        channels=["line", "telegram"],
        recipients={
            "telegram_chats": ["-100123456789"]  # Group chat ID
        }
    )
    
    print("Notification results:", results)
    return results


# Run
if __name__ == "__main__":
    asyncio.run(setup_notification_system())
```

### Database Schema สำหรับ SMS Logging

```sql
CREATE TABLE sms_log (
    id BIGSERIAL PRIMARY KEY,
    recipient VARCHAR(20) NOT NULL,
    message TEXT NOT NULL,
    template_name VARCHAR(100),
    provider VARCHAR(50),
    status VARCHAR(20),
    message_id VARCHAR(100),
    metadata JSONB DEFAULT '{}',
    cost DECIMAL(8, 4),
    sent_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_sms_log_recipient ON sms_log(recipient, sent_at DESC);
CREATE INDEX idx_sms_log_status ON sms_log(status);
```

### Verification Checklist

- [ ] SMS sends successfully to Thai numbers
- [ ] LINE Notify delivers messages
- [ ] Telegram bot sends to group/channel
- [ ] Templates render correctly in Thai
- [ ] Rate limiting prevents spam
- [ ] Escalation fires at correct intervals
- [ ] Acknowledgment stops escalation
- [ ] SMS logs stored in database
- [ ] Multi-channel failover works

> **Tip:** ทดสอบด้วย Twilio sandbox ก่อน เพื่อไม่เสียค่าใช้จ่าย

> **Warning:** LINE Notify มี limit 1000 messages/hour ต้องระวังถ้า alert เยอะ

---

## Summary

Part นี้ครอบคลุม:
- **SMS integration** ทั้ง Twilio และ Thai providers
- **Template system** สำหรับ SMS messages
- **Escalation policies** อัตโนมัติ
- **LINE Notify** และ **Telegram** bot
- **Multi-channel** notification orchestration
- **Cost optimization** ด้วย rate limiting

---

[← Part 69: Report Generation](part-069-report-generation.md) | [Part 71: Enterprise Design →](part-071-enterprise-design.md)
