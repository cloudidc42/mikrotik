# Part 66: Billing System สำหรับ ISP

## สารบัญ
1. [Billing System Architecture](#architecture)
2. [Service Plans](#service-plans)
3. [Usage Tracking](#usage-tracking)
4. [Invoice Generation](#invoices)
5. [Payment Processing](#payment)
6. [Automatic Suspension/Activation](#suspension)
7. [Notifications](#notifications)
8. [Reporting](#reporting)
9. [Thai Payment Gateways](#thai-payment)
10. [Lab: Complete Billing System](#lab)

---

## 1. Billing System Architecture {#architecture}

### Component Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Billing System                             │
│                                                               │
│  ┌──────────────┐    ┌─────────────┐    ┌────────────────┐  │
│  │   Customer   │    │   Invoice   │    │    Payment     │  │
│  │  Management  │───>│  Generator  │───>│   Processor    │  │
│  └──────────────┘    └─────────────┘    └────────────────┘  │
│         │                                        │            │
│         ▼                                        ▼            │
│  ┌──────────────┐    ┌─────────────┐    ┌────────────────┐  │
│  │   Usage      │    │   Account   │    │   MikroTik     │  │
│  │  Tracking    │    │   Balance   │    │  Integration   │  │
│  └──────────────┘    └─────────────┘    └────────────────┘  │
│                                                               │
│  ┌──────────────┐    ┌─────────────┐    ┌────────────────┐  │
│  │ Notification │    │  Reporting  │    │  Auto Actions  │  │
│  │   Service    │    │   Engine    │    │(Suspend/Active)│  │
│  └──────────────┘    └─────────────┘    └────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Database Schema สำหรับ Billing

```sql
-- Billing-specific tables
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_number VARCHAR(20) UNIQUE NOT NULL,
    customer_id UUID NOT NULL REFERENCES customers(id),
    service_id UUID REFERENCES customer_services(id),
    status VARCHAR(20) DEFAULT 'pending'
        CHECK (status IN ('draft', 'pending', 'paid', 'overdue', 'cancelled', 'refunded')),
    issue_date DATE NOT NULL,
    due_date DATE NOT NULL,
    paid_date TIMESTAMP WITH TIME ZONE,
    
    -- Amounts
    subtotal DECIMAL(12, 2) NOT NULL DEFAULT 0,
    discount_amount DECIMAL(12, 2) DEFAULT 0,
    tax_amount DECIMAL(12, 2) DEFAULT 0,
    total_amount DECIMAL(12, 2) NOT NULL,
    
    -- Thai VAT
    vat_rate DECIMAL(5, 2) DEFAULT 7.00,
    vat_amount DECIMAL(12, 2) DEFAULT 0,
    
    notes TEXT,
    metadata JSONB DEFAULT '{}',
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE invoice_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description VARCHAR(255) NOT NULL,
    item_type VARCHAR(50) DEFAULT 'service',
    quantity DECIMAL(10, 2) DEFAULT 1,
    unit_price DECIMAL(12, 2) NOT NULL,
    total_price DECIMAL(12, 2) NOT NULL,
    period_start DATE,
    period_end DATE
);

CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    payment_method VARCHAR(50) NOT NULL,
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'THB',
    status VARCHAR(20) DEFAULT 'pending'
        CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded')),
    transaction_id VARCHAR(255),
    gateway_response JSONB,
    paid_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE usage_records (
    id BIGSERIAL PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES customers(id),
    service_id UUID REFERENCES customer_services(id),
    period_start TIMESTAMP WITH TIME ZONE NOT NULL,
    period_end TIMESTAMP WITH TIME ZONE NOT NULL,
    
    -- Traffic usage
    bytes_downloaded BIGINT DEFAULT 0,
    bytes_uploaded BIGINT DEFAULT 0,
    total_bytes BIGINT GENERATED ALWAYS AS (bytes_downloaded + bytes_uploaded) STORED,
    
    -- Session info
    session_count INT DEFAULT 0,
    peak_speed_rx_bps BIGINT DEFAULT 0,
    peak_speed_tx_bps BIGINT DEFAULT 0,
    
    is_billed BOOLEAN DEFAULT FALSE,
    billed_at TIMESTAMP WITH TIME ZONE
);

-- Auto-generate invoice number
CREATE SEQUENCE invoice_seq START 100001;

CREATE OR REPLACE FUNCTION generate_invoice_number()
RETURNS TRIGGER AS $$
BEGIN
    NEW.invoice_number = 'INV-' || TO_CHAR(NOW(), 'YYYYMM') || '-' || 
                          LPAD(nextval('invoice_seq')::TEXT, 6, '0');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_invoice_number
    BEFORE INSERT ON invoices
    FOR EACH ROW
    WHEN (NEW.invoice_number IS NULL)
    EXECUTE FUNCTION generate_invoice_number();
```

---

## 2. Service Plans {#service-plans}

```python
# app/services/billing/plan_service.py
from decimal import Decimal
from typing import List, Optional, Dict
from pydantic import BaseModel
from sqlalchemy.orm import Session


class ServicePlanModel(BaseModel):
    name: str
    code: str
    download_speed_mbps: int
    upload_speed_mbps: int
    price: Decimal
    billing_cycle: str = 'monthly'
    data_limit_gb: Optional[int] = None
    features: Dict = {}
    
    class Config:
        schema_extra = {
            "examples": [
                {
                    "name": "Home 30/10",
                    "code": "HOME-30-10",
                    "download_speed_mbps": 30,
                    "upload_speed_mbps": 10,
                    "price": 590.00,
                    "billing_cycle": "monthly",
                    "data_limit_gb": None
                },
                {
                    "name": "Business 100/50",
                    "code": "BIZ-100-50",
                    "download_speed_mbps": 100,
                    "upload_speed_mbps": 50,
                    "price": 2500.00,
                    "billing_cycle": "monthly",
                    "data_limit_gb": None,
                    "features": {
                        "static_ip": True,
                        "sla": "99.5%",
                        "support": "24/7"
                    }
                }
            ]
        }


# ISP Plans ในไทยที่พบบ่อย
DEFAULT_PLANS = [
    ServicePlanModel(
        name="Home 10/3",
        code="HOME-010-003",
        download_speed_mbps=10,
        upload_speed_mbps=3,
        price=Decimal("390.00"),
        billing_cycle="monthly"
    ),
    ServicePlanModel(
        name="Home 30/10",
        code="HOME-030-010",
        download_speed_mbps=30,
        upload_speed_mbps=10,
        price=Decimal("590.00"),
        billing_cycle="monthly"
    ),
    ServicePlanModel(
        name="Home 100/30",
        code="HOME-100-030",
        download_speed_mbps=100,
        upload_speed_mbps=30,
        price=Decimal("890.00"),
        billing_cycle="monthly"
    ),
    ServicePlanModel(
        name="Business 50/30",
        code="BIZ-050-030",
        download_speed_mbps=50,
        upload_speed_mbps=30,
        price=Decimal("1500.00"),
        billing_cycle="monthly",
        features={"static_ip": True, "sla": "99%"}
    ),
    ServicePlanModel(
        name="Business 200/100",
        code="BIZ-200-100",
        download_speed_mbps=200,
        upload_speed_mbps=100,
        price=Decimal("3500.00"),
        billing_cycle="monthly",
        features={"static_ip": 2, "sla": "99.5%", "support": "24/7"}
    ),
]
```

---

## 3. Usage Tracking {#usage-tracking}

```python
# app/services/billing/usage_tracker.py
from datetime import datetime, timedelta
from typing import Dict, Optional
import logging
from sqlalchemy.orm import Session
from sqlalchemy import text

from app.services.mikrotik import RouterManager

logger = logging.getLogger(__name__)


class UsageTracker:
    """ติดตาม usage ของ customers"""
    
    def __init__(self, db: Session, router_manager: RouterManager):
        self.db = db
        self.router_manager = router_manager
    
    def collect_pppoe_usage(self, router_id: str) -> List[Dict]:
        """เก็บ usage จาก PPPoE active connections"""
        try:
            with self.router_manager.get_connection(router_id) as conn:
                secrets = conn.run_command('/ppp/active/print', **{
                    '=.proplist': 'name,address,bytes-in,bytes-out,uptime,caller-id'
                })
                
                usage_data = []
                for secret in secrets:
                    usage_data.append({
                        'username': secret.get('name'),
                        'ip_address': secret.get('address'),
                        'bytes_downloaded': int(secret.get('bytes-in', 0)),
                        'bytes_uploaded': int(secret.get('bytes-out', 0)),
                        'uptime': secret.get('uptime'),
                        'caller_id': secret.get('caller-id'),
                        'router_id': router_id,
                        'collected_at': datetime.utcnow()
                    })
                
                return usage_data
        except Exception as e:
            logger.error(f"Failed to collect PPPoE usage from {router_id}: {e}")
            return []
    
    def collect_hotspot_usage(self, router_id: str) -> List[Dict]:
        """เก็บ usage จาก Hotspot active users"""
        try:
            with self.router_manager.get_connection(router_id) as conn:
                active = conn.run_command('/ip/hotspot/active/print', **{
                    '=.proplist': 'user,address,bytes-in,bytes-out,uptime,mac-address'
                })
                
                return [{
                    'username': a.get('user'),
                    'ip_address': a.get('address'),
                    'bytes_downloaded': int(a.get('bytes-in', 0)),
                    'bytes_uploaded': int(a.get('bytes-out', 0)),
                    'mac_address': a.get('mac-address'),
                    'router_id': router_id,
                    'collected_at': datetime.utcnow()
                } for a in active]
        except Exception as e:
            logger.error(f"Failed to collect Hotspot usage from {router_id}: {e}")
            return []
    
    def store_usage_snapshot(self, router_id: str):
        """เก็บ usage snapshot ลง database"""
        pppoe_data = self.collect_pppoe_usage(router_id)
        hotspot_data = self.collect_hotspot_usage(router_id)
        
        all_usage = pppoe_data + hotspot_data
        
        for usage in all_usage:
            # ค้นหา customer service
            service = self.db.execute(
                text("""
                    SELECT cs.id, cs.customer_id
                    FROM customer_services cs
                    WHERE cs.username = :username AND cs.status = 'active'
                    LIMIT 1
                """),
                {"username": usage['username']}
            ).fetchone()
            
            if service:
                self.db.execute(
                    text("""
                        INSERT INTO usage_snapshots 
                            (service_id, customer_id, bytes_downloaded, bytes_uploaded,
                             ip_address, snapshot_time, router_id)
                        VALUES 
                            (:service_id, :customer_id, :bytes_dl, :bytes_ul,
                             :ip, :time, :router_id)
                    """),
                    {
                        "service_id": str(service.id),
                        "customer_id": str(service.customer_id),
                        "bytes_dl": usage['bytes_downloaded'],
                        "bytes_ul": usage['bytes_uploaded'],
                        "ip": usage['ip_address'],
                        "time": usage['collected_at'],
                        "router_id": router_id
                    }
                )
        
        self.db.commit()
        return len(all_usage)
```

---

## 4. Invoice Generation {#invoices}

```python
# app/services/billing/invoice_service.py
from decimal import Decimal
from datetime import date, timedelta
from typing import List, Optional
import uuid
import logging
from sqlalchemy.orm import Session

logger = logging.getLogger(__name__)


class InvoiceService:
    """Service สำหรับสร้างและจัดการ invoices"""
    
    VAT_RATE = Decimal('0.07')  # 7% VAT
    
    def __init__(self, db: Session):
        self.db = db
    
    def generate_monthly_invoices(self, billing_month: date = None) -> int:
        """สร้าง invoices รายเดือนสำหรับทุก active services"""
        if billing_month is None:
            billing_month = date.today().replace(day=1)
        
        # หา services ที่ถึงเวลาออก invoice
        from sqlalchemy import text
        services = self.db.execute(text("""
            SELECT 
                cs.id as service_id,
                cs.customer_id,
                cs.plan_id,
                sp.price,
                sp.name as plan_name,
                cs.start_date,
                c.first_name,
                c.last_name,
                c.email
            FROM customer_services cs
            JOIN service_plans sp ON sp.id = cs.plan_id
            JOIN customers c ON c.id = cs.customer_id
            WHERE cs.status = 'active'
            AND NOT EXISTS (
                SELECT 1 FROM invoices i
                WHERE i.service_id = cs.id
                AND DATE_TRUNC('month', i.issue_date) = :billing_month
                AND i.status != 'cancelled'
            )
        """), {"billing_month": billing_month}).fetchall()
        
        created_count = 0
        for service in services:
            try:
                self.create_monthly_invoice(
                    customer_id=str(service.customer_id),
                    service_id=str(service.service_id),
                    plan_name=service.plan_name,
                    price=service.price,
                    billing_month=billing_month
                )
                created_count += 1
            except Exception as e:
                logger.error(f"Failed to create invoice for service {service.service_id}: {e}")
        
        self.db.commit()
        logger.info(f"Created {created_count} invoices for {billing_month}")
        return created_count
    
    def create_monthly_invoice(
        self,
        customer_id: str,
        service_id: str,
        plan_name: str,
        price: Decimal,
        billing_month: date
    ) -> Dict:
        """สร้าง invoice รายเดือน"""
        issue_date = billing_month
        due_date = billing_month + timedelta(days=15)  # Due ใน 15 วัน
        
        period_start = billing_month
        period_end = (billing_month.replace(day=1) + timedelta(days=32)).replace(day=1) - timedelta(days=1)
        
        subtotal = price
        vat_amount = subtotal * self.VAT_RATE
        total = subtotal + vat_amount
        
        invoice_data = {
            "customer_id": customer_id,
            "service_id": service_id,
            "status": "pending",
            "issue_date": issue_date,
            "due_date": due_date,
            "subtotal": float(subtotal),
            "vat_rate": float(self.VAT_RATE * 100),
            "vat_amount": float(vat_amount),
            "total_amount": float(total),
            "items": [
                {
                    "description": f"Internet Service - {plan_name}",
                    "item_type": "service",
                    "quantity": 1,
                    "unit_price": float(price),
                    "total_price": float(price),
                    "period_start": period_start.isoformat(),
                    "period_end": period_end.isoformat()
                }
            ]
        }
        
        from sqlalchemy import text
        result = self.db.execute(text("""
            INSERT INTO invoices (customer_id, service_id, status, issue_date, due_date,
                                  subtotal, vat_rate, vat_amount, total_amount)
            VALUES (:customer_id::UUID, :service_id::UUID, :status, :issue_date, :due_date,
                    :subtotal, :vat_rate, :vat_amount, :total_amount)
            RETURNING id, invoice_number
        """), invoice_data)
        
        invoice = result.fetchone()
        
        # เพิ่ม invoice items
        for item in invoice_data['items']:
            self.db.execute(text("""
                INSERT INTO invoice_items (invoice_id, description, item_type, 
                                          quantity, unit_price, total_price,
                                          period_start, period_end)
                VALUES (:invoice_id::UUID, :description, :item_type,
                        :quantity, :unit_price, :total_price,
                        :period_start, :period_end)
            """), {"invoice_id": str(invoice.id), **item})
        
        return {
            "invoice_id": str(invoice.id),
            "invoice_number": invoice.invoice_number,
            "total": float(total)
        }
    
    def generate_pdf_invoice(self, invoice_id: str) -> bytes:
        """สร้าง PDF invoice"""
        try:
            from reportlab.lib.pagesizes import A4
            from reportlab.lib import colors
            from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
            from reportlab.lib.styles import getSampleStyleSheet
            from io import BytesIO
            
            buffer = BytesIO()
            doc = SimpleDocTemplate(buffer, pagesize=A4)
            
            # ดึงข้อมูล invoice
            invoice = self._get_invoice_details(invoice_id)
            
            elements = []
            styles = getSampleStyleSheet()
            
            # Header
            elements.append(Paragraph("INVOICE", styles['Title']))
            elements.append(Spacer(1, 12))
            
            # Company info
            elements.append(Paragraph("Your ISP Company Ltd.", styles['Normal']))
            elements.append(Paragraph("123 Tech Street, Bangkok 10110", styles['Normal']))
            elements.append(Paragraph(f"Invoice #: {invoice['invoice_number']}", styles['Normal']))
            elements.append(Spacer(1, 12))
            
            # Items table
            data = [['Description', 'Qty', 'Unit Price', 'Total']]
            for item in invoice['items']:
                data.append([
                    item['description'],
                    str(item['quantity']),
                    f"฿{item['unit_price']:,.2f}",
                    f"฿{item['total_price']:,.2f}"
                ])
            
            # Subtotal, VAT, Total rows
            data.append(['', '', 'Subtotal:', f"฿{invoice['subtotal']:,.2f}"])
            data.append(['', '', f"VAT {invoice['vat_rate']}%:", f"฿{invoice['vat_amount']:,.2f}"])
            data.append(['', '', 'TOTAL:', f"฿{invoice['total_amount']:,.2f}"])
            
            table = Table(data, colWidths=[250, 50, 100, 100])
            table.setStyle(TableStyle([
                ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
                ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
                ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
                ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
                ('FONTSIZE', (0, 0), (-1, 0), 14),
                ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
                ('BACKGROUND', (0, 1), (-1, -1), colors.beige),
                ('GRID', (0, 0), (-1, -1), 1, colors.black),
            ]))
            
            elements.append(table)
            doc.build(elements)
            
            pdf_data = buffer.getvalue()
            buffer.close()
            
            return pdf_data
        except ImportError:
            logger.error("reportlab not installed: pip install reportlab")
            raise


    def _get_invoice_details(self, invoice_id: str) -> Dict:
        """Helper: ดึงข้อมูล invoice ครบ"""
        from sqlalchemy import text
        
        invoice = self.db.execute(text("""
            SELECT i.*, 
                   c.first_name || ' ' || c.last_name as customer_name,
                   c.email, c.address
            FROM invoices i
            JOIN customers c ON c.id = i.customer_id
            WHERE i.id = :id::UUID
        """), {"id": invoice_id}).fetchone()
        
        items = self.db.execute(text("""
            SELECT * FROM invoice_items WHERE invoice_id = :id::UUID
        """), {"id": invoice_id}).fetchall()
        
        return {
            **dict(invoice._mapping),
            "items": [dict(i._mapping) for i in items]
        }
```

---

## 5. Payment Processing {#payment}

```python
# app/services/billing/payment_service.py
from decimal import Decimal
from typing import Dict, Optional
import logging
import httpx

logger = logging.getLogger(__name__)


class PaymentProcessor:
    """Base class สำหรับ payment processors"""
    
    async def process_payment(self, amount: Decimal, currency: str, 
                               payment_data: Dict) -> Dict:
        raise NotImplementedError


class OmisePaymentGateway(PaymentProcessor):
    """Omise payment gateway (ไทย)"""
    
    def __init__(self, public_key: str, secret_key: str):
        self.public_key = public_key
        self.secret_key = secret_key
        self.base_url = "https://api.omise.co"
    
    async def create_charge(
        self,
        amount_satang: int,  # จำนวนเงิน * 100 (สตางค์)
        currency: str,
        source_token: str,
        description: str,
        metadata: Dict = None
    ) -> Dict:
        """สร้าง charge ผ่าน Omise"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/charges",
                auth=(self.secret_key, ""),
                json={
                    "amount": amount_satang,
                    "currency": currency,
                    "source": source_token,
                    "description": description,
                    "metadata": metadata or {}
                }
            )
            
            response.raise_for_status()
            return response.json()
    
    async def process_payment(self, amount: Decimal, currency: str,
                               payment_data: Dict) -> Dict:
        amount_satang = int(amount * 100)
        
        try:
            charge = await self.create_charge(
                amount_satang=amount_satang,
                currency=currency,
                source_token=payment_data['token'],
                description=payment_data.get('description', 'ISP Payment'),
                metadata={"invoice_id": payment_data.get('invoice_id')}
            )
            
            return {
                "success": charge['status'] == 'successful',
                "transaction_id": charge['id'],
                "status": charge['status'],
                "gateway_response": charge
            }
        except Exception as e:
            logger.error(f"Omise payment failed: {e}")
            return {"success": False, "error": str(e)}


class PaymentService:
    """Service จัดการ payments"""
    
    def __init__(self, db, gateway: PaymentProcessor):
        self.db = db
        self.gateway = gateway
    
    async def process_invoice_payment(
        self,
        invoice_id: str,
        payment_method: str,
        payment_data: Dict
    ) -> Dict:
        """ประมวลผลการชำระเงิน"""
        from sqlalchemy import text
        
        # ดึงข้อมูล invoice
        invoice = self.db.execute(text("""
            SELECT id, total_amount, status FROM invoices
            WHERE id = :id::UUID AND status IN ('pending', 'overdue')
        """), {"id": invoice_id}).fetchone()
        
        if not invoice:
            return {"success": False, "error": "Invoice not found or already paid"}
        
        amount = Decimal(str(invoice.total_amount))
        
        # ประมวลผลผ่าน gateway
        result = await self.gateway.process_payment(
            amount=amount,
            currency="THB",
            payment_data={
                **payment_data,
                "invoice_id": invoice_id
            }
        )
        
        if result['success']:
            # อัพเดต invoice status
            self.db.execute(text("""
                UPDATE invoices 
                SET status = 'paid', paid_date = NOW()
                WHERE id = :id::UUID
            """), {"id": invoice_id})
            
            # บันทึก payment record
            self.db.execute(text("""
                INSERT INTO payments (invoice_id, payment_method, amount, 
                                     status, transaction_id, gateway_response)
                VALUES (:invoice_id::UUID, :method, :amount,
                        'completed', :tx_id, :response::JSONB)
            """), {
                "invoice_id": invoice_id,
                "method": payment_method,
                "amount": float(amount),
                "tx_id": result.get('transaction_id'),
                "response": str(result.get('gateway_response', {}))
            })
            
            self.db.commit()
            
            # Activate/reactivate service
            await self._activate_service_after_payment(invoice_id)
        
        return result
    
    async def _activate_service_after_payment(self, invoice_id: str):
        """เปิดใช้งาน service หลังชำระเงิน"""
        from sqlalchemy import text
        
        service = self.db.execute(text("""
            SELECT cs.id, cs.username, cs.router_id
            FROM customer_services cs
            JOIN invoices i ON i.service_id = cs.id
            WHERE i.id = :invoice_id::UUID
        """), {"invoice_id": invoice_id}).fetchone()
        
        if service and service.status == 'suspended':
            # Enable PPPoE user ใน MikroTik
            # ...
            pass
```

---

## 6. Automatic Suspension/Activation {#suspension}

```python
# app/services/billing/suspension_service.py
from datetime import date, datetime, timedelta
from typing import List
import logging
from sqlalchemy.orm import Session
from sqlalchemy import text

from app.services.mikrotik import RouterManager

logger = logging.getLogger(__name__)


class SuspensionService:
    """จัดการ automatic suspension และ activation"""
    
    GRACE_PERIOD_DAYS = 7  # ผ่อนผัน 7 วันหลังครบกำหนด
    
    def __init__(self, db: Session, router_manager: RouterManager):
        self.db = db
        self.router_manager = router_manager
    
    def check_and_suspend_overdue(self):
        """ตรวจสอบและ suspend accounts ที่ค้างชำระ"""
        grace_date = date.today() - timedelta(days=self.GRACE_PERIOD_DAYS)
        
        overdue_services = self.db.execute(text("""
            SELECT DISTINCT cs.id, cs.customer_id, cs.username, 
                           cs.router_id, cs.plan_id,
                           i.invoice_number, i.due_date, i.total_amount
            FROM customer_services cs
            JOIN invoices i ON i.service_id = cs.id
            WHERE cs.status = 'active'
            AND i.status = 'pending'
            AND i.due_date < :grace_date
        """), {"grace_date": grace_date}).fetchall()
        
        suspended_count = 0
        for service in overdue_services:
            try:
                self._suspend_service(
                    service_id=str(service.id),
                    username=service.username,
                    router_id=str(service.router_id),
                    reason=f"Non-payment: Invoice {service.invoice_number}"
                )
                suspended_count += 1
                
                # อัพเดต invoice status เป็น overdue
                self.db.execute(text("""
                    UPDATE invoices SET status = 'overdue'
                    WHERE service_id = :service_id::UUID AND status = 'pending'
                    AND due_date < :today
                """), {"service_id": str(service.id), "today": date.today()})
                
            except Exception as e:
                logger.error(f"Failed to suspend service {service.id}: {e}")
        
        self.db.commit()
        logger.info(f"Suspended {suspended_count} services for non-payment")
        return suspended_count
    
    def _suspend_service(self, service_id: str, username: str, 
                          router_id: str, reason: str):
        """Suspend service ใน database และ MikroTik"""
        # อัพเดต database
        self.db.execute(text("""
            UPDATE customer_services
            SET status = 'suspended', suspend_reason = :reason,
                updated_at = NOW()
            WHERE id = :id::UUID
        """), {"id": service_id, "reason": reason})
        
        # Disable PPPoE user ใน MikroTik
        if router_id:
            try:
                with self.router_manager.get_connection(router_id) as conn:
                    # ค้นหา PPPoE secret
                    secrets = conn.run_command('/ppp/secret/print', 
                                              **{'?name': username})
                    if secrets:
                        secret_id = secrets[0].get('.id')
                        conn.run_command('/ppp/secret/set',
                                        **{'=.id': secret_id, '=disabled': 'yes'})
                    
                    # Kill active sessions
                    active = conn.run_command('/ppp/active/print',
                                            **{'?name': username})
                    for session in active:
                        conn.run_command('/ppp/active/remove',
                                        **{'=.id': session.get('.id')})
                    
                    logger.info(f"Suspended PPPoE user: {username}")
            except Exception as e:
                logger.error(f"Failed to disable PPPoE user {username}: {e}")
    
    def activate_service(self, service_id: str):
        """Activate service หลังชำระเงิน"""
        service = self.db.execute(text("""
            SELECT id, username, router_id
            FROM customer_services
            WHERE id = :id::UUID
        """), {"id": service_id}).fetchone()
        
        if not service:
            raise ValueError(f"Service {service_id} not found")
        
        # Enable ใน MikroTik
        if service.router_id and service.username:
            try:
                with self.router_manager.get_connection(str(service.router_id)) as conn:
                    secrets = conn.run_command('/ppp/secret/print',
                                              **{'?name': service.username})
                    if secrets:
                        conn.run_command('/ppp/secret/set', **{
                            '=.id': secrets[0].get('.id'),
                            '=disabled': 'no'
                        })
            except Exception as e:
                logger.error(f"Failed to enable PPPoE user: {e}")
        
        # อัพเดต database
        self.db.execute(text("""
            UPDATE customer_services
            SET status = 'active', suspend_reason = NULL, updated_at = NOW()
            WHERE id = :id::UUID
        """), {"id": service_id})
        
        self.db.commit()
```

---

## 7. Notifications {#notifications}

```python
# app/services/billing/notification_service.py
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.application import MIMEApplication
from typing import Optional, List
import logging
from jinja2 import Template

logger = logging.getLogger(__name__)


class BillingNotificationService:
    """ส่ง notifications สำหรับ billing events"""
    
    def __init__(self, smtp_config: Dict, sms_service=None):
        self.smtp_config = smtp_config
        self.sms_service = sms_service
    
    def send_invoice_email(
        self,
        customer_email: str,
        customer_name: str,
        invoice_number: str,
        amount: float,
        due_date: str,
        pdf_attachment: bytes = None
    ):
        """ส่ง invoice ทาง email"""
        template = Template("""
        <html>
        <body>
            <h2>ใบแจ้งหนี้ {{ invoice_number }}</h2>
            <p>เรียน คุณ{{ customer_name }},</p>
            <p>กรุณาชำระค่าบริการอินเทอร์เน็ต:</p>
            <table>
                <tr><td>เลขที่ใบแจ้งหนี้:</td><td>{{ invoice_number }}</td></tr>
                <tr><td>จำนวนเงิน:</td><td>฿{{ amount|float|round(2) }}</td></tr>
                <tr><td>กำหนดชำระ:</td><td>{{ due_date }}</td></tr>
            </table>
            <p>ช่องทางชำระเงิน:</p>
            <ul>
                <li>โอนเงินผ่านธนาคาร</li>
                <li>ชำระผ่าน QR Code</li>
                <li>ชำระที่สำนักงาน</li>
            </ul>
        </body>
        </html>
        """)
        
        html_content = template.render(
            invoice_number=invoice_number,
            customer_name=customer_name,
            amount=amount,
            due_date=due_date
        )
        
        msg = MIMEMultipart('alternative')
        msg['Subject'] = f"ใบแจ้งหนี้ {invoice_number} - กรุณาชำระภายใน {due_date}"
        msg['From'] = self.smtp_config['sender']
        msg['To'] = customer_email
        
        msg.attach(MIMEText(html_content, 'html', 'utf-8'))
        
        if pdf_attachment:
            attachment = MIMEApplication(pdf_attachment, _subtype="pdf")
            attachment.add_header('Content-Disposition', 'attachment',
                                  filename=f"{invoice_number}.pdf")
            msg.attach(attachment)
        
        self._send_email(customer_email, msg)
    
    def send_suspension_warning(self, customer_email: str, customer_name: str,
                                  days_until_suspension: int):
        """แจ้งเตือนก่อน suspend"""
        subject = f"แจ้งเตือน: บริการของท่านจะถูกระงับใน {days_until_suspension} วัน"
        body = f"""
        เรียน คุณ{customer_name},
        
        บัญชีของท่านมีค่าบริการค้างชำระ
        หากไม่ชำระภายใน {days_until_suspension} วัน บริการอินเทอร์เน็ตจะถูกระงับ
        
        กรุณาติดต่อเราเพื่อชำระเงินโดยด่วน
        
        ขอบคุณ
        ทีมงาน ISP
        """
        
        self._send_simple_email(customer_email, subject, body)
    
    def _send_email(self, to_email: str, msg):
        """ส่ง email"""
        try:
            with smtplib.SMTP(self.smtp_config['host'], self.smtp_config['port']) as server:
                server.starttls()
                server.login(self.smtp_config['username'], self.smtp_config['password'])
                server.send_message(msg)
            logger.info(f"Email sent to {to_email}")
        except Exception as e:
            logger.error(f"Failed to send email to {to_email}: {e}")
    
    def _send_simple_email(self, to_email: str, subject: str, body: str):
        """ส่ง email แบบ simple text"""
        msg = MIMEText(body, 'plain', 'utf-8')
        msg['Subject'] = subject
        msg['From'] = self.smtp_config['sender']
        msg['To'] = to_email
        self._send_email(to_email, msg)
```

---

## 8. Thai Payment Gateways {#thai-payment}

```python
# app/services/payment/thai_gateways.py
"""Thai Payment Gateway Integrations"""
import httpx
import hashlib
import hmac
import json
from decimal import Decimal


class OmiseGateway:
    """Omise - รองรับ PromptPay, Credit Card, Internet Banking"""
    API_URL = "https://api.omise.co"
    
    def __init__(self, secret_key: str, public_key: str):
        self.secret_key = secret_key
        self.public_key = public_key
    
    async def create_promptpay_charge(self, amount_thb: Decimal) -> Dict:
        """สร้าง PromptPay QR Code"""
        async with httpx.AsyncClient() as client:
            # สร้าง source ก่อน
            source_resp = await client.post(
                f"{self.API_URL}/sources",
                auth=(self.public_key, ""),
                json={
                    "amount": int(amount_thb * 100),
                    "currency": "THB",
                    "type": "promptpay"
                }
            )
            source = source_resp.json()
            
            # สร้าง charge
            charge_resp = await client.post(
                f"{self.API_URL}/charges",
                auth=(self.secret_key, ""),
                json={
                    "amount": int(amount_thb * 100),
                    "currency": "THB",
                    "source": source['id']
                }
            )
            charge = charge_resp.json()
            
            return {
                "charge_id": charge['id'],
                "qr_code_url": charge.get('source', {}).get('scannable_code', {}).get('image', {}).get('download_uri'),
                "expires_at": charge.get('expires_at'),
                "status": charge['status']
            }


class GBPrimePay:
    """GB Prime Pay - Thai payment gateway"""
    
    def __init__(self, public_key: str, secret_key: str, is_sandbox: bool = True):
        self.public_key = public_key
        self.secret_key = secret_key
        self.base_url = "https://api.gbprimepay.com" if not is_sandbox else "https://api.gbprimepay.com"
    
    def create_qr30(self, amount: Decimal, reference: str) -> Dict:
        """สร้าง Thai QR Code (QR30)"""
        payload = {
            "token": self.public_key,
            "amount": f"{amount:.2f}",
            "referenceNo": reference,
            "backgroundUrl": "https://your-site.com/payment/callback",
            "detail": "ISP Payment",
            "customerName": "Customer",
            "customerEmail": "customer@example.com",
            "merchantDefined1": reference
        }
        
        import requests
        response = requests.post(f"{self.base_url}/v1/qrcode/create", json=payload)
        return response.json()


class KBankQRCode:
    """KBank Payment Gateway (PromptPay)"""
    
    def __init__(self, api_key: str, merchant_id: str):
        self.api_key = api_key
        self.merchant_id = merchant_id
        self.base_url = "https://openapi.kasikornbank.com"
    
    async def generate_qr(self, amount: Decimal, ref1: str, ref2: str) -> Dict:
        """สร้าง KBank PromptPay QR"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/v1/qrpayment/request",
                headers={
                    "apikey": self.api_key,
                    "Content-Type": "application/json"
                },
                json={
                    "partnerTxnUid": ref1,
                    "partnerId": self.merchant_id,
                    "partnerSecret": self.api_key,
                    "requestDt": "2024-01-01T00:00:00+07:00",
                    "metadata": {
                        "detail": "ISP Payment",
                        "billPaymentRef1": ref1,
                        "billPaymentRef2": ref2,
                        "billPaymentRef3": "ISP",
                        "amount": str(amount),
                        "currencyCode": "THB"
                    }
                }
            )
            return response.json()
```

---

## 9. Lab: Complete Billing System {#lab}

### Lab Setup

```bash
# 1. Setup database
psql $DATABASE_URL -f database/billing_schema.sql

# 2. Insert sample data
psql $DATABASE_URL <<EOF
-- Service plans
INSERT INTO service_plans (name, code, download_speed_mbps, upload_speed_mbps, price)
VALUES 
    ('Home 30/10', 'HOME-30-10', 30, 10, 590.00),
    ('Business 100/50', 'BIZ-100-50', 100, 50, 2500.00);

-- Sample customer
INSERT INTO customers (customer_code, first_name, last_name, email, phone)
VALUES ('CUST-001', 'สมชาย', 'ใจดี', 'somchai@email.com', '0812345678');
EOF

# 3. Generate invoices
python -m scripts.generate_monthly_invoices

# 4. Check invoices
psql $DATABASE_URL -c "SELECT invoice_number, total_amount, status FROM invoices LIMIT 10;"
```

### Billing Automation Script

```python
# scripts/run_billing.py
"""Script สำหรับรัน billing processes"""
import schedule
import time
import logging
from datetime import date

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def daily_billing_jobs():
    """งาน billing ประจำวัน"""
    from app.database import SessionLocal
    from app.services.billing.suspension_service import SuspensionService
    from app.services.billing.notification_service import BillingNotificationService
    from app.services.mikrotik import router_manager
    
    db = SessionLocal()
    try:
        suspension_svc = SuspensionService(db, router_manager)
        
        # 1. Suspend overdue accounts
        suspended = suspension_svc.check_and_suspend_overdue()
        logger.info(f"Suspended {suspended} accounts")
        
        # 2. Send warning notifications (7 days before due)
        # ...
        
    finally:
        db.close()


def monthly_billing_jobs():
    """งาน billing ประจำเดือน"""
    from app.database import SessionLocal
    from app.services.billing.invoice_service import InvoiceService
    
    db = SessionLocal()
    try:
        invoice_svc = InvoiceService(db)
        today = date.today()
        
        if today.day == 1:  # วันแรกของเดือน
            created = invoice_svc.generate_monthly_invoices()
            logger.info(f"Generated {created} invoices")
    finally:
        db.close()


# Schedule jobs
schedule.every().day.at("09:00").do(daily_billing_jobs)
schedule.every().day.at("00:01").do(monthly_billing_jobs)

if __name__ == "__main__":
    logger.info("Billing scheduler started")
    while True:
        schedule.run_pending()
        time.sleep(60)
```

### Verification Checklist

- [ ] Invoices generate automatically monthly
- [ ] PDF invoices created correctly
- [ ] Email notifications sent
- [ ] Overdue accounts suspended in MikroTik
- [ ] Payment re-activates service
- [ ] Thai payment gateway integration works
- [ ] VAT calculated correctly (7%)
- [ ] Invoice numbers auto-generated

> **Note:** ต้องขอ license จาก PromptPay (NITMX) สำหรับใช้งานจริง

> **Warning:** ทดสอบ gateway ใน sandbox mode ก่อนเสมอ

---

## Summary

Part นี้ครอบคลุม:
- **Billing architecture** ครบวงจร
- **Invoice generation** รายเดือนอัตโนมัติ
- **Payment processing** ด้วย Omise, GBPrimePay, KBank
- **Auto suspension** สำหรับ non-payment
- **Email notifications** พร้อม PDF attachment
- **Thai VAT** (7%) calculation

---

[← Part 65: Docker Deployment](part-065-docker-deployment.md) | [Part 67: ISP Platform →](part-067-isp-platform.md)
