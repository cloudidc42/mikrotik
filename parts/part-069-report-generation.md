# Part 69: Report Generation

## สารบัญ
1. [Report Types](#report-types)
2. [Data Aggregation](#aggregation)
3. [PDF Generation](#pdf)
4. [Excel Reports](#excel)
5. [Email Scheduling](#email-schedule)
6. [Dashboard Reports](#dashboard)
7. [Custom Report Builder](#builder)
8. [Data Visualization](#visualization)
9. [Export Formats](#export)
10. [Lab: Automated Reporting System](#lab)

---

## 1. Report Types {#report-types}

### รายการ Report ที่จำเป็นสำหรับ ISP

| Report | ความถี่ | ผู้รับ | รูปแบบ |
|--------|---------|--------|--------|
| Traffic Summary | Daily/Weekly/Monthly | NOC, Management | PDF, Excel |
| Customer Usage | Monthly | Finance, Customer | PDF |
| Billing Summary | Monthly | Finance | Excel |
| Security Events | Daily/Weekly | Security Team | PDF |
| Network Performance | Weekly | NOC, Engineering | PDF |
| SLA Report | Monthly | Management, Customer | PDF |
| Top Talkers | Daily | NOC | Dashboard |
| Bandwidth Utilization | Weekly | Engineering | Excel |

---

## 2. Data Aggregation {#aggregation}

```python
# app/services/reporting/aggregator.py
from datetime import datetime, timedelta, date
from typing import Dict, List, Optional
from sqlalchemy.orm import Session
from sqlalchemy import text
import logging

logger = logging.getLogger(__name__)


class ReportDataAggregator:
    """Aggregate data สำหรับ reports"""
    
    def __init__(self, db: Session):
        self.db = db
    
    def get_traffic_summary(
        self,
        start_date: datetime,
        end_date: datetime,
        router_ids: List[str] = None
    ) -> Dict:
        """รวม traffic data สำหรับ report"""
        
        router_filter = ""
        params = {
            "start_date": start_date,
            "end_date": end_date
        }
        
        if router_ids:
            router_filter = "AND ts.router_id = ANY(:router_ids::UUID[])"
            params["router_ids"] = router_ids
        
        # Daily traffic summary
        daily_data = self.db.execute(text(f"""
            SELECT 
                DATE(ts.timestamp) as date,
                r.name as router_name,
                ts.interface_name,
                SUM(ts.rx_bytes) as total_rx_bytes,
                SUM(ts.tx_bytes) as total_tx_bytes,
                MAX(ts.rx_bytes) as peak_rx_bytes,
                MAX(ts.tx_bytes) as peak_tx_bytes,
                AVG(ts.rx_bytes) as avg_rx_bytes,
                AVG(ts.tx_bytes) as avg_tx_bytes
            FROM traffic_stats ts
            JOIN routers r ON r.id = ts.router_id
            WHERE ts.timestamp BETWEEN :start_date AND :end_date
            {router_filter}
            GROUP BY DATE(ts.timestamp), r.name, ts.interface_name
            ORDER BY DATE(ts.timestamp), r.name
        """), params).fetchall()
        
        # Overall summary
        summary = self.db.execute(text(f"""
            SELECT 
                SUM(ts.rx_bytes + ts.tx_bytes) as total_bytes,
                SUM(ts.rx_bytes) as total_rx,
                SUM(ts.tx_bytes) as total_tx,
                MAX(ts.rx_bytes + ts.tx_bytes) as peak_bytes,
                AVG(ts.rx_bytes + ts.tx_bytes) as avg_bytes
            FROM traffic_stats ts
            WHERE ts.timestamp BETWEEN :start_date AND :end_date
            {router_filter}
        """), params).fetchone()
        
        return {
            "period": {
                "start": start_date.isoformat(),
                "end": end_date.isoformat()
            },
            "summary": dict(summary._mapping) if summary else {},
            "daily_data": [dict(d._mapping) for d in daily_data]
        }
    
    def get_customer_usage_report(
        self,
        billing_month: date,
        customer_id: str = None
    ) -> List[Dict]:
        """Usage report สำหรับ customers"""
        
        params = {
            "month_start": billing_month.replace(day=1),
            "month_end": (billing_month.replace(day=1) + timedelta(days=32)).replace(day=1)
        }
        
        customer_filter = ""
        if customer_id:
            customer_filter = "AND c.id = :customer_id::UUID"
            params["customer_id"] = customer_id
        
        usage_data = self.db.execute(text(f"""
            SELECT 
                c.customer_code,
                c.first_name || ' ' || c.last_name as customer_name,
                sp.name as plan_name,
                sp.download_speed_mbps,
                sp.upload_speed_mbps,
                SUM(ur.bytes_downloaded) as total_download,
                SUM(ur.bytes_uploaded) as total_upload,
                SUM(ur.bytes_downloaded + ur.bytes_uploaded) as total_bytes,
                sp.data_limit_gb,
                CASE 
                    WHEN sp.data_limit_gb IS NOT NULL 
                    THEN ROUND(
                        SUM(ur.bytes_downloaded + ur.bytes_uploaded) / 
                        (sp.data_limit_gb::FLOAT * 1073741824) * 100, 
                        2
                    )
                    ELSE NULL 
                END as usage_percent
            FROM customers c
            JOIN customer_services cs ON cs.customer_id = c.id
            JOIN service_plans sp ON sp.id = cs.plan_id
            LEFT JOIN usage_records ur ON ur.service_id = cs.id
                AND ur.period_start >= :month_start
                AND ur.period_start < :month_end
            WHERE cs.status IN ('active', 'suspended')
            {customer_filter}
            GROUP BY c.customer_code, c.first_name, c.last_name, 
                     sp.name, sp.download_speed_mbps, sp.upload_speed_mbps,
                     sp.data_limit_gb
            ORDER BY total_bytes DESC
        """), params).fetchall()
        
        return [dict(d._mapping) for d in usage_data]
    
    def get_billing_summary(self, billing_month: date) -> Dict:
        """Billing summary รายเดือน"""
        month_start = billing_month.replace(day=1)
        month_end = (month_start + timedelta(days=32)).replace(day=1)
        
        result = self.db.execute(text("""
            SELECT 
                COUNT(*) as total_invoices,
                COUNT(*) FILTER (WHERE status = 'paid') as paid_invoices,
                COUNT(*) FILTER (WHERE status = 'pending') as pending_invoices,
                COUNT(*) FILTER (WHERE status = 'overdue') as overdue_invoices,
                SUM(total_amount) as total_billed,
                SUM(total_amount) FILTER (WHERE status = 'paid') as total_collected,
                SUM(total_amount) FILTER (WHERE status IN ('pending', 'overdue')) as total_outstanding,
                COUNT(DISTINCT customer_id) as unique_customers
            FROM invoices
            WHERE issue_date >= :month_start AND issue_date < :month_end
        """), {"month_start": month_start, "month_end": month_end}).fetchone()
        
        # Revenue by plan
        by_plan = self.db.execute(text("""
            SELECT 
                sp.name as plan_name,
                COUNT(i.id) as invoice_count,
                SUM(i.total_amount) as revenue
            FROM invoices i
            JOIN customer_services cs ON cs.id = i.service_id
            JOIN service_plans sp ON sp.id = cs.plan_id
            WHERE i.issue_date >= :month_start AND i.issue_date < :month_end
            GROUP BY sp.name
            ORDER BY revenue DESC
        """), {"month_start": month_start, "month_end": month_end}).fetchall()
        
        return {
            "month": billing_month.strftime("%Y-%m"),
            "summary": dict(result._mapping) if result else {},
            "revenue_by_plan": [dict(p._mapping) for p in by_plan]
        }
```

---

## 3. PDF Generation {#pdf}

```python
# app/services/reporting/pdf_generator.py
from io import BytesIO
from typing import Dict, List
from datetime import datetime
import logging

logger = logging.getLogger(__name__)


class PDFReportGenerator:
    """สร้าง PDF reports"""
    
    def __init__(self):
        try:
            from reportlab.lib.pagesizes import A4, landscape
            from reportlab.lib import colors
            from reportlab.platypus import (
                SimpleDocTemplate, Table, TableStyle, 
                Paragraph, Spacer, PageBreak, Image
            )
            from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
            from reportlab.lib.units import cm
            from reportlab.graphics.shapes import Drawing
            from reportlab.graphics.charts.lineplots import LinePlot
            self._rl_available = True
        except ImportError:
            logger.warning("reportlab not installed: pip install reportlab")
            self._rl_available = False
    
    def generate_traffic_report(
        self,
        report_data: Dict,
        company_name: str = "ISP Company"
    ) -> bytes:
        """สร้าง traffic report PDF"""
        if not self._rl_available:
            raise ImportError("reportlab is required: pip install reportlab")
        
        from reportlab.lib.pagesizes import A4, landscape
        from reportlab.lib import colors
        from reportlab.platypus import (
            SimpleDocTemplate, Table, TableStyle,
            Paragraph, Spacer, PageBreak
        )
        from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
        from reportlab.lib.units import cm
        
        buffer = BytesIO()
        doc = SimpleDocTemplate(
            buffer,
            pagesize=landscape(A4),
            leftMargin=1.5*cm,
            rightMargin=1.5*cm,
            topMargin=2*cm,
            bottomMargin=2*cm
        )
        
        styles = getSampleStyleSheet()
        
        # Custom styles
        title_style = ParagraphStyle(
            'CustomTitle',
            parent=styles['Title'],
            fontSize=18,
            spaceAfter=20
        )
        
        header_style = ParagraphStyle(
            'CustomHeader',
            parent=styles['Heading2'],
            fontSize=12,
            spaceAfter=10
        )
        
        elements = []
        
        # Title page
        elements.append(Paragraph(f"{company_name}", title_style))
        elements.append(Paragraph("Traffic Analysis Report", title_style))
        
        period = report_data.get("period", {})
        elements.append(Paragraph(
            f"Period: {period.get('start', '')} to {period.get('end', '')}",
            styles['Normal']
        ))
        elements.append(Paragraph(
            f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M')}",
            styles['Normal']
        ))
        elements.append(Spacer(1, 20))
        
        # Summary section
        elements.append(Paragraph("Executive Summary", header_style))
        
        summary = report_data.get("summary", {})
        
        def bytes_to_human(b):
            if not b:
                return "0 B"
            b = float(b)
            for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
                if abs(b) < 1024.0:
                    return f"{b:.2f} {unit}"
                b /= 1024.0
            return f"{b:.2f} PB"
        
        summary_data = [
            ['Metric', 'Value'],
            ['Total Traffic', bytes_to_human(summary.get('total_bytes'))],
            ['Total Download', bytes_to_human(summary.get('total_rx'))],
            ['Total Upload', bytes_to_human(summary.get('total_tx'))],
            ['Peak Traffic', bytes_to_human(summary.get('peak_bytes'))],
            ['Average Traffic', bytes_to_human(summary.get('avg_bytes'))],
        ]
        
        summary_table = Table(summary_data, colWidths=[200, 200])
        summary_table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.HexColor('#2196F3')),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.white),
            ('ALIGN', (0, 0), (-1, -1), 'LEFT'),
            ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
            ('FONTSIZE', (0, 0), (-1, 0), 11),
            ('BACKGROUND', (0, 1), (-1, -1), colors.HexColor('#F5F5F5')),
            ('ROWBACKGROUNDS', (0, 1), (-1, -1), 
             [colors.white, colors.HexColor('#F5F5F5')]),
            ('GRID', (0, 0), (-1, -1), 0.5, colors.HexColor('#CCCCCC')),
            ('TOPPADDING', (0, 0), (-1, -1), 6),
            ('BOTTOMPADDING', (0, 0), (-1, -1), 6),
        ]))
        
        elements.append(summary_table)
        elements.append(Spacer(1, 20))
        
        # Daily breakdown
        elements.append(Paragraph("Daily Traffic Breakdown", header_style))
        
        daily_data = report_data.get("daily_data", [])
        if daily_data:
            table_data = [['Date', 'Router', 'Interface', 'Download', 'Upload', 'Total', 'Peak']]
            
            for row in daily_data[:50]:  # Limit to 50 rows
                table_data.append([
                    str(row.get('date', '')),
                    str(row.get('router_name', '')),
                    str(row.get('interface_name', '')),
                    bytes_to_human(row.get('total_rx_bytes')),
                    bytes_to_human(row.get('total_tx_bytes')),
                    bytes_to_human(
                        (row.get('total_rx_bytes') or 0) + 
                        (row.get('total_tx_bytes') or 0)
                    ),
                    bytes_to_human(row.get('peak_rx_bytes')),
                ])
            
            col_widths = [80, 120, 100, 90, 90, 90, 90]
            data_table = Table(table_data, colWidths=col_widths)
            data_table.setStyle(TableStyle([
                ('BACKGROUND', (0, 0), (-1, 0), colors.HexColor('#1976D2')),
                ('TEXTCOLOR', (0, 0), (-1, 0), colors.white),
                ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
                ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
                ('FONTSIZE', (0, 0), (-1, -1), 8),
                ('ROWBACKGROUNDS', (0, 1), (-1, -1), 
                 [colors.white, colors.HexColor('#E3F2FD')]),
                ('GRID', (0, 0), (-1, -1), 0.5, colors.grey),
                ('TOPPADDING', (0, 0), (-1, -1), 4),
                ('BOTTOMPADDING', (0, 0), (-1, -1), 4),
            ]))
            
            elements.append(data_table)
        
        doc.build(elements)
        
        pdf_bytes = buffer.getvalue()
        buffer.close()
        
        return pdf_bytes
```

---

## 4. Excel Reports {#excel}

```python
# app/services/reporting/excel_generator.py
from io import BytesIO
from typing import Dict, List
from datetime import datetime
import logging

logger = logging.getLogger(__name__)


class ExcelReportGenerator:
    """สร้าง Excel reports"""
    
    def generate_billing_report(self, billing_data: Dict) -> bytes:
        """สร้าง billing Excel report"""
        try:
            import openpyxl
            from openpyxl.styles import (
                PatternFill, Font, Alignment, Border, Side
            )
            from openpyxl.utils import get_column_letter
            from openpyxl.chart import BarChart, Reference
        except ImportError:
            raise ImportError("openpyxl required: pip install openpyxl")
        
        wb = openpyxl.Workbook()
        
        # Sheet 1: Summary
        ws_summary = wb.active
        ws_summary.title = "Billing Summary"
        
        # Header styles
        header_fill = PatternFill(start_color="1976D2", end_color="1976D2", fill_type="solid")
        header_font = Font(color="FFFFFF", bold=True, size=11)
        
        # Title
        ws_summary['A1'] = "Billing Summary Report"
        ws_summary['A1'].font = Font(bold=True, size=16)
        ws_summary['A2'] = f"Month: {billing_data.get('month', '')}"
        ws_summary['A3'] = f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M')}"
        
        # Summary table
        summary = billing_data.get('summary', {})
        summary_headers = ['Metric', 'Value']
        summary_rows = [
            ['Total Invoices', summary.get('total_invoices', 0)],
            ['Paid Invoices', summary.get('paid_invoices', 0)],
            ['Pending Invoices', summary.get('pending_invoices', 0)],
            ['Overdue Invoices', summary.get('overdue_invoices', 0)],
            ['Total Billed (THB)', f"฿{summary.get('total_billed', 0):,.2f}"],
            ['Total Collected (THB)', f"฿{summary.get('total_collected', 0):,.2f}"],
            ['Outstanding (THB)', f"฿{summary.get('total_outstanding', 0):,.2f}"],
        ]
        
        # Write headers
        row_start = 5
        for col, header in enumerate(summary_headers, 1):
            cell = ws_summary.cell(row=row_start, column=col, value=header)
            cell.fill = header_fill
            cell.font = header_font
            cell.alignment = Alignment(horizontal='center')
        
        # Write data
        for row_idx, row_data in enumerate(summary_rows, row_start + 1):
            for col_idx, value in enumerate(row_data, 1):
                cell = ws_summary.cell(row=row_idx, column=col_idx, value=value)
                if row_idx % 2 == 0:
                    cell.fill = PatternFill(
                        start_color="E3F2FD",
                        end_color="E3F2FD",
                        fill_type="solid"
                    )
        
        # Auto-fit columns
        ws_summary.column_dimensions['A'].width = 30
        ws_summary.column_dimensions['B'].width = 20
        
        # Sheet 2: Revenue by Plan
        ws_plans = wb.create_sheet("Revenue by Plan")
        
        plan_headers = ['Plan', 'Invoices', 'Revenue (THB)']
        for col, header in enumerate(plan_headers, 1):
            cell = ws_plans.cell(row=1, column=col, value=header)
            cell.fill = header_fill
            cell.font = header_font
        
        plans_data = billing_data.get('revenue_by_plan', [])
        for row_idx, plan in enumerate(plans_data, 2):
            ws_plans.cell(row=row_idx, column=1, value=plan.get('plan_name', ''))
            ws_plans.cell(row=row_idx, column=2, value=plan.get('invoice_count', 0))
            ws_plans.cell(row=row_idx, column=3, value=float(plan.get('revenue', 0)))
        
        # สร้าง chart
        if len(plans_data) > 0:
            chart = BarChart()
            chart.title = "Revenue by Plan"
            chart.y_axis.title = "Revenue (THB)"
            chart.x_axis.title = "Plan"
            
            data_ref = Reference(
                ws_plans,
                min_col=3,
                min_row=1,
                max_row=len(plans_data) + 1
            )
            cats_ref = Reference(
                ws_plans,
                min_col=1,
                min_row=2,
                max_row=len(plans_data) + 1
            )
            
            chart.add_data(data_ref, titles_from_data=True)
            chart.set_categories(cats_ref)
            
            ws_plans.add_chart(chart, "E2")
        
        # แต่ละ column width
        for col in ['A', 'B', 'C']:
            ws_plans.column_dimensions[col].width = 20
        
        buffer = BytesIO()
        wb.save(buffer)
        
        excel_bytes = buffer.getvalue()
        buffer.close()
        
        return excel_bytes
    
    def generate_customer_usage_excel(self, usage_data: List[Dict]) -> bytes:
        """สร้าง customer usage Excel"""
        try:
            import openpyxl
            from openpyxl.styles import PatternFill, Font
        except ImportError:
            raise ImportError("openpyxl required")
        
        wb = openpyxl.Workbook()
        ws = wb.active
        ws.title = "Customer Usage"
        
        headers = [
            'Customer Code', 'Customer Name', 'Plan', 
            'Download Speed', 'Upload Speed',
            'Total Download', 'Total Upload', 'Total Usage',
            'Data Limit', 'Usage %'
        ]
        
        header_fill = PatternFill(start_color="1565C0", end_color="1565C0", fill_type="solid")
        header_font = Font(color="FFFFFF", bold=True)
        
        for col, header in enumerate(headers, 1):
            cell = ws.cell(row=1, column=col, value=header)
            cell.fill = header_fill
            cell.font = header_font
        
        def bytes_to_gb(b):
            if not b:
                return 0
            return round(float(b) / (1024**3), 2)
        
        for row_idx, usage in enumerate(usage_data, 2):
            ws.cell(row=row_idx, column=1, value=usage.get('customer_code'))
            ws.cell(row=row_idx, column=2, value=usage.get('customer_name'))
            ws.cell(row=row_idx, column=3, value=usage.get('plan_name'))
            ws.cell(row=row_idx, column=4, value=f"{usage.get('download_speed_mbps')} Mbps")
            ws.cell(row=row_idx, column=5, value=f"{usage.get('upload_speed_mbps')} Mbps")
            ws.cell(row=row_idx, column=6, value=bytes_to_gb(usage.get('total_download')))
            ws.cell(row=row_idx, column=7, value=bytes_to_gb(usage.get('total_upload')))
            ws.cell(row=row_idx, column=8, value=bytes_to_gb(usage.get('total_bytes')))
            ws.cell(row=row_idx, column=9, value=usage.get('data_limit_gb') or 'Unlimited')
            ws.cell(row=row_idx, column=10, value=usage.get('usage_percent') or 'N/A')
        
        # Auto-fit
        for col_num in range(1, len(headers) + 1):
            ws.column_dimensions[chr(64 + col_num)].width = 18
        
        buffer = BytesIO()
        wb.save(buffer)
        return buffer.getvalue()
```

---

## 5. Email Scheduling {#email-schedule}

```python
# app/services/reporting/scheduler.py
import schedule
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.application import MIMEApplication
from email.mime.text import MIMEText
from datetime import datetime, date, timedelta
from typing import List, Dict
import logging

logger = logging.getLogger(__name__)


class ReportScheduler:
    """Scheduler สำหรับส่ง reports อัตโนมัติ"""
    
    def __init__(self, smtp_config: Dict, db_session_factory, aggregator_class):
        self.smtp_config = smtp_config
        self.db_factory = db_session_factory
        self.aggregator_class = aggregator_class
    
    def send_daily_traffic_report(self, recipients: List[str]):
        """ส่ง daily traffic report"""
        yesterday = datetime.now().replace(hour=0, minute=0, second=0) - timedelta(days=1)
        today = datetime.now().replace(hour=0, minute=0, second=0)
        
        db = self.db_factory()
        try:
            aggregator = self.aggregator_class(db)
            data = aggregator.get_traffic_summary(yesterday, today)
            
            from app.services.reporting.pdf_generator import PDFReportGenerator
            pdf_gen = PDFReportGenerator()
            pdf_bytes = pdf_gen.generate_traffic_report(data)
            
            filename = f"traffic_report_{yesterday.strftime('%Y%m%d')}.pdf"
            subject = f"Daily Traffic Report - {yesterday.strftime('%Y-%m-%d')}"
            
            self._send_report_email(recipients, subject, pdf_bytes, filename)
            logger.info(f"Daily traffic report sent to {len(recipients)} recipients")
        finally:
            db.close()
    
    def send_monthly_billing_report(self, recipients: List[str]):
        """ส่ง monthly billing report"""
        last_month = (datetime.now().replace(day=1) - timedelta(days=1)).date()
        
        db = self.db_factory()
        try:
            aggregator = self.aggregator_class(db)
            billing_data = aggregator.get_billing_summary(last_month)
            
            from app.services.reporting.excel_generator import ExcelReportGenerator
            excel_gen = ExcelReportGenerator()
            excel_bytes = excel_gen.generate_billing_report(billing_data)
            
            filename = f"billing_report_{last_month.strftime('%Y%m')}.xlsx"
            subject = f"Monthly Billing Report - {last_month.strftime('%B %Y')}"
            
            self._send_report_email(recipients, subject, excel_bytes, filename, 
                                   content_type='application/vnd.ms-excel')
        finally:
            db.close()
    
    def _send_report_email(
        self,
        recipients: List[str],
        subject: str,
        attachment: bytes,
        filename: str,
        content_type: str = 'application/pdf'
    ):
        """ส่ง email พร้อม attachment"""
        msg = MIMEMultipart()
        msg['Subject'] = subject
        msg['From'] = self.smtp_config['sender']
        msg['To'] = ', '.join(recipients)
        
        body = MIMEText(f"Please find the report attached: {filename}", 'plain', 'utf-8')
        msg.attach(body)
        
        attachment_part = MIMEApplication(attachment, _subtype=content_type.split('/')[-1])
        attachment_part.add_header('Content-Disposition', 'attachment', filename=filename)
        msg.attach(attachment_part)
        
        with smtplib.SMTP(self.smtp_config['host'], self.smtp_config['port']) as server:
            server.starttls()
            server.login(self.smtp_config['username'], self.smtp_config['password'])
            server.sendmail(self.smtp_config['sender'], recipients, msg.as_string())
    
    def setup_schedules(
        self,
        daily_recipients: List[str],
        monthly_recipients: List[str]
    ):
        """ตั้ง schedule สำหรับ reports"""
        # Daily traffic report เวลา 07:00
        schedule.every().day.at("07:00").do(
            self.send_daily_traffic_report,
            recipients=daily_recipients
        )
        
        # Monthly billing report วันที่ 2 ของเดือน เวลา 08:00
        schedule.every().day.at("08:00").do(
            lambda: self.send_monthly_billing_report(monthly_recipients)
            if datetime.now().day == 2
            else None
        )
        
        logger.info("Report schedules configured")
```

---

## 6. Lab: Automated Reporting System {#lab}

### Setup

```bash
# Install dependencies
pip install reportlab openpyxl matplotlib pandas schedule

# Test PDF generation
python -c "
from app.database import SessionLocal
from app.services.reporting.aggregator import ReportDataAggregator
from app.services.reporting.pdf_generator import PDFReportGenerator
from datetime import datetime, timedelta

db = SessionLocal()
agg = ReportDataAggregator(db)
end = datetime.now()
start = end - timedelta(days=7)
data = agg.get_traffic_summary(start, end)

pdf_gen = PDFReportGenerator()
pdf = pdf_gen.generate_traffic_report(data)

with open('/tmp/traffic_report.pdf', 'wb') as f:
    f.write(pdf)
print('PDF generated: /tmp/traffic_report.pdf')
"

# Test Excel generation
python -c "
from app.database import SessionLocal
from app.services.reporting.aggregator import ReportDataAggregator
from app.services.reporting.excel_generator import ExcelReportGenerator
from datetime import date

db = SessionLocal()
agg = ReportDataAggregator(db)
data = agg.get_billing_summary(date.today().replace(day=1))

excel_gen = ExcelReportGenerator()
excel = excel_gen.generate_billing_report(data)

with open('/tmp/billing_report.xlsx', 'wb') as f:
    f.write(excel)
print('Excel generated: /tmp/billing_report.xlsx')
"
```

### Verification Checklist

- [ ] PDF reports generate with proper formatting
- [ ] Excel reports include charts
- [ ] Data aggregation returns correct numbers
- [ ] Email delivery with attachments works
- [ ] Scheduled reports fire at correct times
- [ ] Large datasets handled efficiently (>10,000 rows)
- [ ] Thai characters display correctly in PDF/Excel

> **Tip:** ใช้ `pandas` สำหรับ data manipulation ก่อน generate report

> **Note:** สำหรับ Thai font ใน PDF ต้องเพิ่ม font file และ register กับ reportlab

---

## Summary

Part นี้ครอบคลุม:
- **Traffic reports** PDF แบบ professional
- **Billing reports** Excel พร้อม charts
- **Email scheduling** อัตโนมัติ
- **Data aggregation** สำหรับ complex reports
- **Multiple export formats** (PDF, Excel)

---

[← Part 68: Inventory System](part-068-inventory-system.md) | [Part 70: SMS Notification →](part-070-sms-notification.md)
