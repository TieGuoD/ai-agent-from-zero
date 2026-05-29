# 第 59 章：项目 3 -- 邮件 Agent

> **本章定位：** 邮件是职场人士每天面对最多的工具之一。一个高效的邮件 Agent 能够帮你阅读邮件、分类整理、起草回复、安排日程，把你从邮件的泥潭中解放出来。本章将构建一个完整的邮件处理 Agent，让你体验 Agent 在日常办公场景中的实用价值。

---

## 学习目标

完成本章学习后，你将能够：

1. **实现邮件协议的安全对接** -- 能够通过 IMAP/SMTP 协议安全地连接邮箱，读取和发送邮件，理解 OAuth2 认证流程
2. **构建智能邮件分类系统** -- 能够让 Agent 自动识别邮件的重要性、类型和所需操作，实现智能分拣
3. **实现邮件内容的深度理解** -- 能够让 Agent 理解邮件的语义、情绪和意图，提取关键信息和行动项
4. **构建智能回复生成功能** -- 能够让 Agent 根据邮件内容和上下文，生成得体、专业的回复草稿
5. **设计安全的邮件操作流程** -- 能够处理敏感信息、权限控制、操作审计等安全相关问题
6. **完成一个可日常使用的邮件助手** -- 包括自动分类、摘要生成、快速回复、日程提取等功能

## 核心问题

1. **邮件 Agent 和邮件客户端的规则过滤有什么区别？** 为什么简单的关键词过滤不够用？Agent 的"智能"体现在哪里？
2. **让 AI 读取和回复邮件，安全性如何保障？** 如何防止敏感信息泄露？如何确保 Agent 不会发送不当的回复？
3. **邮件的语境理解为什么重要？** 一封邮件的真实意图往往藏在字里行间，Agent 如何捕捉这些微妙之处？

---

## 59.1 项目背景与需求分析

### 59.1.1 邮件处理的痛点

根据 McKinsey 的研究报告，知识工作者平均每天花 28% 的工作时间处理邮件，其中大部分时间花在了不必要的操作上：阅读大量不相关的邮件、手动分类和归档、反复确认邮件内容、为简单的回复反复斟酌措辞。

一个典型的场景是这样的：你早上打开邮箱，里面有 50 封未读邮件。你需要逐封阅读，判断哪些重要、哪些可以忽略、哪些需要立即回复、哪些需要转发给同事。这个过程可能要花你一个小时，而且你很可能在阅读第 20 封邮件时就开始感到疲劳，错过一些重要的信息。

邮件 Agent 的价值在于，它能够在你打开邮箱之前，就帮你完成了大部分的分类和筛选工作。它能够告诉你："今天有 50 封新邮件，其中 3 封需要你立即处理，5 封需要你今天回复，12 封是通知类邮件可以稍后阅读，30 封是营销邮件可以直接归档。"

### 59.1.2 功能需求定义

基于以上分析，我们确定邮件 Agent 需要具备以下核心功能：

首先是**邮件读取和解析功能**。Agent 需要能够通过 IMAP 协议安全地连接用户的邮箱，读取邮件列表和邮件内容。这包括处理各种编码的邮件文本、解析 HTML 格式的邮件、提取附件信息等。

其次是**智能分类功能**。Agent 需要能够自动将邮件分为不同的类别：紧急、重要、普通、通知、营销等。这个分类不仅基于发件人和主题，还需要理解邮件的实际内容。

然后是**邮件摘要生成功能**。对于长邮件或大量邮件，Agent 需要能够生成简洁的摘要，让用户快速了解邮件的核心内容，而不需要逐字阅读。

最后是**智能回复生成功能**。Agent 需要能够根据邮件内容、发件人关系、历史邮件往来等因素，生成合适的回复草稿。用户只需要审核和微调就可以发送。

### 59.1.3 技术选型

邮件处理涉及网络协议和安全认证，技术选型需要格外谨慎。

在邮件协议层面，我们使用 Python 的 `imaplib` 和 `smtplib` 标准库。这两个库虽然 API 有些古老，但它们稳定可靠，支持所有主流的邮件协议。如果需要更现代的 API，可以使用 `imapclient` 库作为替代。

在安全认证层面，我们使用 OAuth2 认证。传统的用户名密码认证在 Gmail 和 Outlook 等主流邮箱中已经不再支持，OAuth2 成为了标准方案。

在内容解析层面，我们使用 Python 标准库中的 `email` 模块来解析邮件的 MIME 结构。

---

## 59.2 核心代码实现

### 59.2.1 邮件连接管理

```python
# mail_connector.py
import imaplib
import smtplib
import email
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.header import decode_header
from email.utils import parsedate_to_datetime
import os
import re
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional


@dataclass
class MailConfig:
    """邮件配置。"""
    imap_server: str
    imap_port: int = 993
    smtp_server: str = ""
    smtp_port: int = 587
    email_address: str = ""
    password: str = ""  # 或 OAuth2 access token
    use_ssl: bool = True

    @classmethod
    def from_env(cls, provider: str = "gmail") -> "MailConfig":
        """从环境变量加载配置。"""
        configs = {
            "gmail": {
                "imap_server": "imap.gmail.com",
                "smtp_server": "smtp.gmail.com",
                "smtp_port": 587,
            },
            "outlook": {
                "imap_server": "outlook.office365.com",
                "smtp_server": "smtp.office365.com",
                "smtp_port": 587,
            },
            "qq": {
                "imap_server": "imap.qq.com",
                "smtp_server": "smtp.qq.com",
                "smtp_port": 465,
            },
        }

        server_config = configs.get(provider, configs["gmail"])

        return cls(
            imap_server=server_config["imap_server"],
            imap_port=993,
            smtp_server=server_config["smtp_server"],
            smtp_port=server_config["smtp_port"],
            email_address=os.environ.get("EMAIL_ADDRESS", ""),
            password=os.environ.get("EMAIL_PASSWORD", ""),
            use_ssl=True,
        )


@dataclass
class EmailMessage:
    """解析后的邮件消息。"""
    uid: str
    subject: str
    sender: str
    recipients: list[str]
    date: Optional[datetime]
    body_text: str          # 纯文本内容
    body_html: str = ""     # HTML 内容
    attachments: list[dict] = field(default_factory=list)
    headers: dict = field(default_factory=dict)
    is_read: bool = False
    folder: str = "INBOX"

    def get_preview(self, max_length: int = 200) -> str:
        """获取邮件预览。"""
        preview = self.body_text[:max_length].replace("\n", " ").strip()
        if len(self.body_text) > max_length:
            preview += "..."
        return preview


class MailConnector:
    """邮件连接器。

    封装了 IMAP 和 SMTP 协议的操作细节，
    提供简洁的接口来读取和发送邮件。
    """

    def __init__(self, config: MailConfig):
        self.config = config
        self.imap: Optional[imaplib.IMAP4_SSL] = None

    def connect(self) -> bool:
        """连接到 IMAP 服务器。"""
        try:
            self.imap = imaplib.IMAP4_SSL(
                self.config.imap_server,
                self.config.imap_port
            )
            self.imap.login(self.config.email_address, self.config.password)
            print(f"[邮箱连接成功] {self.config.email_address}")
            return True
        except Exception as e:
            print(f"[邮箱连接失败] {e}")
            return False

    def disconnect(self):
        """断开连接。"""
        if self.imap:
            try:
                self.imap.logout()
            except Exception:
                pass
            self.imap = None

    def get_folders(self) -> list[str]:
        """获取所有邮箱文件夹。"""
        if not self.imap:
            return []
        status, folders = self.imap.list()
        if status == "OK":
            return [f.decode().split('" "')[-1].rstrip('"') for f in folders]
        return []

    def search_emails(self, folder: str = "INBOX", criteria: str = "ALL",
                      limit: int = 50) -> list[str]:
        """搜索邮件并返回 UID 列表。

        criteria 示例：
        - "ALL"：所有邮件
        - "UNSEEN"：未读邮件
        - "FROM user@example.com"：特定发件人
        - "SINCE 01-Jan-2025"：指定日期之后
        - "SUBJECT 重要"：主题包含关键词
        """
        if not self.imap:
            return []

        try:
            self.imap.select(folder, readonly=True)
            status, data = self.imap.search(None, criteria)
            if status != "OK":
                return []

            uids = data[0].split()
            # 只返回最新的 limit 封
            return [uid.decode() for uid in uids[-limit:]]
        except Exception as e:
            print(f"[搜索邮件失败] {e}")
            return []

    def fetch_email(self, uid: str, folder: str = "INBOX") -> Optional[EmailMessage]:
        """获取单封邮件的完整内容。"""
        if not self.imap:
            return None

        try:
            self.imap.select(folder, readonly=True)
            status, data = self.imap.fetch(uid.encode(), "(RFC822)")
            if status != "OK":
                return None

            raw_email = data[0][1]
            msg = email.message_from_bytes(raw_email)

            # 解析邮件头
            subject = self._decode_header(msg.get("Subject", ""))
            sender = msg.get("From", "")
            date_str = msg.get("Date", "")
            date = None
            if date_str:
                try:
                    date = parsedate_to_datetime(date_str)
                except Exception:
                    pass

            # 获取收件人列表
            to_field = msg.get("To", "")
            recipients = [addr.strip() for addr in to_field.split(",") if addr.strip()]

            # 解析邮件正文
            body_text, body_html = self._extract_body(msg)

            # 解析附件
            attachments = self._extract_attachments(msg)

            # 解析其他头部信息
            headers = {
                "Message-ID": msg.get("Message-ID", ""),
                "In-Reply-To": msg.get("In-Reply-To", ""),
                "References": msg.get("References", ""),
                "X-Priority": msg.get("X-Priority", "3"),
            }

            return EmailMessage(
                uid=uid,
                subject=subject,
                sender=sender,
                recipients=recipients,
                date=date,
                body_text=body_text,
                body_html=body_html,
                attachments=attachments,
                headers=headers,
                folder=folder,
            )

        except Exception as e:
            print(f"[获取邮件失败] UID={uid}, 错误: {e}")
            return None

    def fetch_multiple(self, uids: list[str], folder: str = "INBOX") -> list[EmailMessage]:
        """批量获取邮件。"""
        emails = []
        for uid in uids:
            msg = self.fetch_email(uid, folder)
            if msg:
                emails.append(msg)
        return emails

    def send_email(self, to: str, subject: str, body: str,
                   cc: str = "", bcc: str = "",
                   in_reply_to: str = "", references: str = "") -> bool:
        """发送邮件。

        Args:
            to: 收件人
            subject: 主题
            body: 正文
            cc: 抄送
            bcc: 密送
            in_reply_to: 回复的邮件 Message-ID
            references: 引用的邮件链
        """
        msg = MIMEMultipart()
        msg["From"] = self.config.email_address
        msg["To"] = to
        msg["Subject"] = subject
        if cc:
            msg["Cc"] = cc
        if in_reply_to:
            msg["In-Reply-To"] = in_reply_to
            msg["References"] = references

        msg.attach(MIMEText(body, "plain", "utf-8"))

        try:
            if self.config.smtp_port == 465:
                # SSL 连接
                server = smtplib.SMTP_SSL(
                    self.config.smtp_server,
                    self.config.smtp_port
                )
            else:
                # STARTTLS 连接
                server = smtplib.SMTP(
                    self.config.smtp_server,
                    self.config.smtp_port
                )
                server.starttls()

            server.login(self.config.email_address, self.config.password)

            recipients = [to]
            if cc:
                recipients.extend(cc.split(","))
            if bcc:
                recipients.extend(bcc.split(","))

            server.sendmail(self.config.email_address, recipients, msg.as_string())
            server.quit()

            print(f"[邮件已发送] 收件人: {to}, 主题: {subject}")
            return True

        except Exception as e:
            print(f"[邮件发送失败] {e}")
            return False

    def _decode_header(self, header: str) -> str:
        """解码邮件头（可能包含编码的中文等）。"""
        if not header:
            return ""
        decoded_parts = decode_header(header)
        result = []
        for part, charset in decoded_parts:
            if isinstance(part, bytes):
                result.append(part.decode(charset or "utf-8", errors="replace"))
            else:
                result.append(part)
        return " ".join(result)

    def _extract_body(self, msg) -> tuple[str, str]:
        """从邮件中提取纯文本和 HTML 正文。"""
        body_text = ""
        body_html = ""

        if msg.is_multipart():
            for part in msg.walk():
                content_type = part.get_content_type()
                content_disposition = str(part.get("Content-Disposition", ""))

                # 跳过附件
                if "attachment" in content_disposition:
                    continue

                if content_type == "text/plain":
                    payload = part.get_payload(decode=True)
                    if payload:
                        charset = part.get_content_charset() or "utf-8"
                        body_text = payload.decode(charset, errors="replace")
                elif content_type == "text/html":
                    payload = part.get_payload(decode=True)
                    if payload:
                        charset = part.get_content_charset() or "utf-8"
                        body_html = payload.decode(charset, errors="replace")
        else:
            payload = msg.get_payload(decode=True)
            if payload:
                charset = msg.get_content_charset() or "utf-8"
                text = payload.decode(charset, errors="replace")
                if msg.get_content_type() == "text/html":
                    body_html = text
                    body_text = self._html_to_text(text)
                else:
                    body_text = text

        # 如果没有纯文本版本，从 HTML 转换
        if not body_text and body_html:
            body_text = self._html_to_text(body_html)

        return body_text.strip(), body_html

    def _html_to_text(self, html: str) -> str:
        """将 HTML 转换为纯文本。"""
        # 移除 style 和 script 标签
        text = re.sub(r'<style[^>]*>.*?</style>', '', html, flags=re.DOTALL)
        text = re.sub(r'<script[^>]*>.*?</script>', '', text, flags=re.DOTALL)
        # 将 br 和 p 标签转换为换行
        text = re.sub(r'<br\s*/?>', '\n', text)
        text = re.sub(r'</p>', '\n', text)
        text = re.sub(r'</div>', '\n', text)
        # 移除所有 HTML 标签
        text = re.sub(r'<[^>]+>', '', text)
        # 清理多余空白
        text = re.sub(r'\n{3,}', '\n\n', text)
        text = re.sub(r' {2,}', ' ', text)
        return text.strip()

    def _extract_attachments(self, msg) -> list[dict]:
        """提取附件信息。"""
        attachments = []
        for part in msg.walk():
            content_disposition = str(part.get("Content-Disposition", ""))
            if "attachment" in content_disposition:
                filename = part.get_filename()
                if filename:
                    filename = self._decode_header(filename)
                attachments.append({
                    "filename": filename or "unknown",
                    "content_type": part.get_content_type(),
                    "size": len(part.get_payload(decode=True) or b""),
                })
        return attachments
```

### 59.2.2 智能分类器

```python
# classifier.py
import json
from openai import OpenAI
from dataclasses import dataclass


@dataclass
class EmailClassification:
    """邮件分类结果。"""
    importance: str       # "urgent", "important", "normal", "low"
    category: str         # "work", "notification", "marketing", "personal", "spam"
    sentiment: str        # "positive", "neutral", "negative"
    action_needed: bool   # 是否需要用户操作
    action_type: str      # "reply", "review", "forward", "archive", "none"
    summary: str          # 一句话摘要
    confidence: float     # 分类置信度


class EmailClassifier:
    """邮件智能分类器。

    使用 LLM 理解邮件的语义内容，进行多维度分类。
    比传统的规则过滤更准确，能处理复杂的语境。
    """

    CLASSIFICATION_PROMPT = """你是一个邮件分类专家。请对以下邮件进行分类分析。

## 邮件信息
发件人：{sender}
主题：{subject}
日期：{date}
内容预览：{body_preview}

## 分析维度

请从以下维度分析这封邮件：

1. **重要性**（importance）
   - urgent：需要在 1 小时内处理的紧急事项
   - important：今天内需要关注的重要事项
   - normal：普通的日常邮件
   - low：可以稍后处理或忽略的邮件

2. **类别**（category）
   - work：工作相关邮件
   - notification：系统通知、自动回复等
   - notification：营销推广邮件
   - personal：个人邮件
   - spam：垃圾邮件或钓鱼邮件

3. **情感**（sentiment）
   - positive：积极正面的
   - neutral：中性的
   - negative：消极负面的（投诉、催促等）

4. **是否需要操作**（action_needed）
   - 判断收件人是否需要采取行动

5. **操作类型**（action_type）
   - reply：需要回复
   - review：需要仔细阅读
   - forward：需要转发给他人
   - archive：归档即可
   - none：不需要任何操作

6. **一句话摘要**（summary）
   - 用不超过 20 个字概括邮件内容

请以 JSON 格式返回分析结果。
"""

    def __init__(self, api_key: str, model: str = "gpt-4o-mini",
                 base_url: str = None):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model

    def classify(self, sender: str, subject: str, date: str,
                 body_preview: str) -> EmailClassification:
        """对单封邮件进行分类。"""
        prompt = self.CLASSIFICATION_PROMPT.format(
            sender=sender,
            subject=subject,
            date=date,
            body_preview=body_preview[:500]
        )

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "你是一个邮件分类助手。请严格按照 JSON 格式返回结果。"},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.0,
                max_tokens=500,
                response_format={"type": "json_object"},
            )

            result = json.loads(response.choices[0].message.content)

            return EmailClassification(
                importance=result.get("importance", "normal"),
                category=result.get("category", "work"),
                sentiment=result.get("sentiment", "neutral"),
                action_needed=result.get("action_needed", False),
                action_type=result.get("action_type", "none"),
                summary=result.get("summary", ""),
                confidence=0.8,
            )

        except Exception as e:
            print(f"[分类失败] {e}")
            return EmailClassification(
                importance="normal",
                category="work",
                sentiment="neutral",
                action_needed=False,
                action_type="none",
                summary="分类失败",
                confidence=0.0,
            )

    def batch_classify(self, emails: list[dict]) -> list[EmailClassification]:
        """批量分类多封邮件。

        Args:
            emails: 邮件信息列表，每项包含 sender, subject, date, body_preview
        """
        results = []
        for mail in emails:
            classification = self.classify(
                sender=mail.get("sender", ""),
                subject=mail.get("subject", ""),
                date=mail.get("date", ""),
                body_preview=mail.get("body_preview", "")
            )
            results.append(classification)
        return results

    def classify_and_group(self, emails: list[dict]) -> dict:
        """分类并按重要性分组。"""
        classifications = self.batch_classify(emails)

        grouped = {
            "urgent": [],
            "important": [],
            "normal": [],
            "low": [],
        }

        for mail, cls in zip(emails, classifications):
            item = {**mail, "classification": cls}
            importance = cls.importance
            if importance in grouped:
                grouped[importance].append(item)
            else:
                grouped["normal"].append(item)

        # 统计
        total = len(classifications)
        need_action = sum(1 for c in classifications if c.action_needed)

        return {
            "total": total,
            "need_action": need_action,
            "grouped": grouped,
            "classifications": classifications,
        }
```

### 59.2.3 回复生成器

```python
# reply_generator.py
from openai import OpenAI
from typing import Optional


class ReplyGenerator:
    """邮件回复生成器。

    根据邮件内容和上下文，生成得体、专业的回复草稿。
    支持不同风格的回复：正式、友好、简洁。
    """

    REPLY_PROMPT = """你是一个专业的邮件回复助手。请根据以下信息生成一封邮件回复。

## 原始邮件
发件人：{sender}
主题：{subject}
内容：
{body}

## 回复要求
- 风格：{style}
- 要点：{key_points}
- 语言：{language}

## 回复原则

1. **称呼得体**：根据发件人的身份和关系选择合适的称呼
2. **内容完整**：回复应该涵盖原始邮件提到的所有要点
3. **语气恰当**：根据邮件的正式程度调整语气
4. **简洁明了**：除非需要详细解释，否则尽量简洁
5. **行动明确**：如果需要采取后续行动，明确说明

请直接输出邮件正文，不需要包含邮件头信息。
"""

    SUMMARY_PROMPT = """请为以下邮件生成一句话摘要（不超过 30 个字）：

邮件主题：{subject}
邮件内容：{body}
"""

    ACTION_ITEMS_PROMPT = """请从以下邮件中提取所有需要采取的行动项：

邮件主题：{subject}
邮件内容：{body}

请以 JSON 数组格式返回行动项列表，每个行动项包含：
- action: 行动描述
- deadline: 截止时间（如果邮件中提到的话，否则为 null）
- priority: 优先级（high/medium/low）
"""

    def __init__(self, api_key: str, model: str = "gpt-4o",
                 base_url: str = None):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model

    def generate_reply(self, sender: str, subject: str, body: str,
                       style: str = "professional",
                       key_points: str = "全面回复邮件提到的所有要点",
                       language: str = "中文") -> str:
        """生成邮件回复。

        Args:
            sender: 原始邮件发件人
            subject: 原始邮件主题
            body: 原始邮件正文
            style: 回复风格（professional/formal/friendly/concise）
            key_points: 需要特别强调的要点
            language: 回复语言

        Returns:
            生成的回复正文
        """
        prompt = self.REPLY_PROMPT.format(
            sender=sender,
            subject=subject,
            body=body,
            style=style,
            key_points=key_points,
            language=language,
        )

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "你是一个专业的邮件助手。"},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.3,
                max_tokens=1500,
            )
            return response.choices[0].message.content
        except Exception as e:
            return f"[回复生成失败: {e}]"

    def generate_summary(self, subject: str, body: str) -> str:
        """生成邮件摘要。"""
        prompt = self.SUMMARY_PROMPT.format(subject=subject, body=body[:2000])

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "你是一个简洁的摘要助手。"},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.1,
                max_tokens=100,
            )
            return response.choices[0].message.content.strip()
        except Exception as e:
            return "摘要生成失败"

    def extract_action_items(self, subject: str, body: str) -> list[dict]:
        """从邮件中提取行动项。"""
        import json
        prompt = self.ACTION_ITEMS_PROMPT.format(subject=subject, body=body[:2000])

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "你是一个行动项提取助手。请返回 JSON 数组。"},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.0,
                max_tokens=500,
                response_format={"type": "json_object"},
            )
            result = json.loads(response.choices[0].message.content)
            if isinstance(result, dict) and "action_items" in result:
                return result["action_items"]
            elif isinstance(result, list):
                return result
            return []
        except Exception as e:
            print(f"[行动项提取失败] {e}")
            return []
```

### 59.2.4 邮件 Agent 主体

```python
# agent.py
import json
import os
from datetime import datetime
from mail_connector import MailConnector, MailConfig, EmailMessage
from classifier import EmailClassifier, EmailClassification
from reply_generator import ReplyGenerator


class EmailAgent:
    """邮件 Agent 的主体类。

    协调所有组件，提供完整的邮件处理工作流。
    """

    def __init__(self, provider: str = "gmail"):
        # 初始化配置
        self.mail_config = MailConfig.from_env(provider)
        api_key = os.environ.get("OPENAI_API_KEY", "")

        # 初始化组件
        self.connector = MailConnector(self.mail_config)
        self.classifier = EmailClassifier(api_key=api_key)
        self.reply_generator = ReplyGenerator(api_key=api_key)

        self.classified_emails: dict[str, EmailMessage] = {}
        self.classifications: dict[str, EmailClassification] = {}

    def connect(self) -> bool:
        """连接到邮箱。"""
        return self.connector.connect()

    def disconnect(self):
        """断开连接。"""
        self.connector.disconnect()

    def daily_briefing(self, folder: str = "INBOX",
                       limit: int = 30) -> str:
        """生成每日邮件简报。

        这是邮件 Agent 最核心的功能：
        读取最近的邮件，进行分类，生成一份结构化的简报。
        """
        print(f"\n[生成每日简报] 正在读取最近 {limit} 封邮件...")

        # 搜索最近的邮件
        uids = self.connector.search_emails(folder=folder, limit=limit)
        if not uids:
            return "没有找到新邮件。"

        # 获取邮件内容
        emails = self.connector.fetch_multiple(uids, folder)
        print(f"[已读取] {len(emails)} 封邮件")

        # 分类
        mail_dicts = []
        for em in emails:
            mail_dicts.append({
                "uid": em.uid,
                "sender": em.sender,
                "subject": em.subject,
                "date": str(em.date) if em.date else "",
                "body_preview": em.get_preview(500),
            })

        result = self.classifier.classify_and_group(mail_dicts)

        # 保存分类结果
        for em, cls in zip(emails, result["classifications"]):
            self.classified_emails[em.uid] = em
            self.classifications[em.uid] = cls

        # 生成简报
        briefing = self._format_briefing(result)
        return briefing

    def _format_briefing(self, classify_result: dict) -> str:
        """格式化每日简报。"""
        lines = []
        lines.append("=" * 50)
        lines.append(f"  每日邮件简报 - {datetime.now().strftime('%Y-%m-%d')}")
        lines.append("=" * 50)

        total = classify_result["total"]
        need_action = classify_result["need_action"]
        grouped = classify_result["grouped"]

        lines.append(f"\n共 {total} 封邮件，{need_action} 封需要你操作\n")

        # 紧急邮件
        if grouped["urgent"]:
            lines.append("[!] 紧急邮件：")
            for item in grouped["urgent"]:
                cls = item["classification"]
                lines.append(f"  - [{cls.summary}] 来自: {item['sender']}")
                lines.append(f"    主题: {item['subject']}")
            lines.append("")

        # 重要邮件
        if grouped["important"]:
            lines.append("[*] 重要邮件：")
            for item in grouped["important"]:
                cls = item["classification"]
                lines.append(f"  - [{cls.summary}] 来自: {item['sender']}")
                lines.append(f"    主题: {item['subject']}")
                lines.append(f"    操作: {cls.action_type}")
            lines.append("")

        # 普通邮件
        if grouped["normal"]:
            lines.append(f"[=] 普通邮件（{len(grouped['normal'])} 封）：")
            for item in grouped["normal"][:5]:
                cls = item["classification"]
                lines.append(f"  - [{cls.summary}] {item['subject']}")
            if len(grouped["normal"]) > 5:
                lines.append(f"  ... 还有 {len(grouped['normal']) - 5} 封")
            lines.append("")

        # 低优先级
        if grouped["low"]:
            lines.append(f"[ ] 低优先级（{len(grouped['low'])} 封，可忽略）")

        return "\n".join(lines)

    def reply_to_email(self, uid: str, style: str = "professional",
                       custom_points: str = "") -> str:
        """为指定邮件生成回复。"""
        email_msg = self.classified_emails.get(uid)
        if not email_msg:
            # 尝试重新获取
            email_msg = self.connector.fetch_email(uid)
            if not email_msg:
                return "未找到指定邮件。"

        reply = self.reply_generator.generate_reply(
            sender=email_msg.sender,
            subject=email_msg.subject,
            body=email_msg.body_text,
            style=style,
            key_points=custom_points or "全面回复邮件提到的所有要点",
        )

        return f"To: {email_msg.sender}\nSubject: Re: {email_msg.subject}\n\n{reply}"

    def send_reply(self, uid: str, reply_body: str,
                   cc: str = "") -> bool:
        """发送回复。"""
        email_msg = self.classified_emails.get(uid)
        if not email_msg:
            return False

        return self.connector.send_email(
            to=email_msg.sender,
            subject=f"Re: {email_msg.subject}",
            body=reply_body,
            cc=cc,
            in_reply_to=email_msg.headers.get("Message-ID", ""),
            references=email_msg.headers.get("References", "") or email_msg.headers.get("Message-ID", ""),
        )

    def search_and_summarize(self, query: str, folder: str = "INBOX",
                             limit: int = 10) -> str:
        """搜索邮件并生成摘要。"""
        uids = self.connector.search_emails(folder=folder, criteria=query, limit=limit)
        if not uids:
            return f"未找到匹配 '{query}' 的邮件。"

        emails = self.connector.fetch_multiple(uids, folder)

        summaries = []
        for em in emails:
            summary = self.reply_generator.generate_summary(
                subject=em.subject,
                body=em.body_text
            )
            summaries.append(f"- {em.subject}（{em.sender}）：{summary}")

        header = f"搜索 '{query}' 找到 {len(emails)} 封邮件：\n\n"
        return header + "\n".join(summaries)

    def get_action_items(self, uid: str) -> list[dict]:
        """提取邮件中的行动项。"""
        email_msg = self.classified_emails.get(uid)
        if not email_msg:
            email_msg = self.connector.fetch_email(uid)
        if not email_msg:
            return []

        return self.reply_generator.extract_action_items(
            subject=email_msg.subject,
            body=email_msg.body_text
        )
```

### 59.2.5 入口文件

```python
# main.py
import os
from agent import EmailAgent


def print_menu():
    print("\n" + "=" * 40)
    print("  邮件 Agent - 主菜单")
    print("=" * 40)
    print("  1. 生成每日简报")
    print("  2. 回复邮件")
    print("  3. 搜索邮件")
    print("  4. 查看行动项")
    print("  5. 退出")
    print("-" * 40)


def main():
    print("邮件 Agent v1.0")
    print("让 AI 帮你管理邮件\n")

    # 检查环境变量
    if not os.environ.get("EMAIL_ADDRESS"):
        print("[错误] 请设置 EMAIL_ADDRESS 和 EMAIL_PASSWORD 环境变量。")
        print("  对于 Gmail，需要使用应用专用密码。")
        return

    if not os.environ.get("OPENAI_API_KEY"):
        print("[错误] 请设置 OPENAI_API_KEY 环境变量。")
        return

    # 选择邮箱提供商
    provider = input("选择邮箱提供商 (gmail/outlook/qq) [默认 gmail]: ").strip() or "gmail"

    # 创建 Agent
    agent = EmailAgent(provider=provider)

    # 连接
    if not agent.connect():
        print("无法连接到邮箱，请检查配置。")
        return

    try:
        while True:
            print_menu()
            choice = input("请选择操作: ").strip()

            if choice == "1":
                briefing = agent.daily_briefing()
                print(briefing)

            elif choice == "2":
                uid = input("输入邮件 UID: ").strip()
                style = input("回复风格 (professional/friendly/concise) [默认 professional]: ").strip() or "professional"
                custom = input("额外要点（可选）: ").strip()

                reply = agent.reply_to_email(uid, style=style, custom_points=custom)
                print(f"\n生成的回复：\n{reply}")

                confirm = input("\n是否发送？(y/n): ").strip().lower()
                if confirm == "y":
                    # 提取回复正文（去掉 header）
                    body = reply.split("\n\n", 1)[1] if "\n\n" in reply else reply
                    agent.send_reply(uid, body)
                    print("邮件已发送！")

            elif choice == "3":
                query = input("搜索关键词: ").strip()
                result = agent.search_and_summarize(query)
                print(f"\n{result}")

            elif choice == "4":
                uid = input("输入邮件 UID: ").strip()
                items = agent.get_action_items(uid)
                if items:
                    print(f"\n行动项：")
                    for item in items:
                        print(f"  - [{item.get('priority', 'medium')}] {item.get('action', '')}")
                        if item.get('deadline'):
                            print(f"    截止: {item['deadline']}")
                else:
                    print("未找到行动项。")

            elif choice == "5":
                break

            else:
                print("无效选择，请重试。")

    finally:
        agent.disconnect()
        print("\n已断开连接。再见！")


if __name__ == "__main__":
    main()
```

---

## 59.3 运行指南

```bash
# 安装依赖
pip install openai

# 设置环境变量
# Gmail 用户需要生成"应用专用密码"：
# Google 账号 -> 安全 -> 两步验证 -> 应用专用密码

export EMAIL_ADDRESS="your-email@gmail.com"
export EMAIL_PASSWORD="your-app-password"
export OPENAI_API_KEY="your-api-key"

# 运行
python main.py
```

---

## 59.4 案例分析

### 59.4.1 案例一：处理工作日邮件

早上 9 点启动邮件 Agent，它连接到 Gmail 邮箱，读取了昨天下午到今天早上的 35 封邮件。经过分类，Agent 报告：

```
每日邮件简报 - 2025-06-15

共 35 封邮件，5 封需要你操作

[!] 紧急邮件：
  - [项目截止日期变更通知] 来自: boss@company.com
    主题: 关于 Q2 项目截止日期的调整
    操作: reply

[*] 重要邮件：
  - [代码审查请求] 来自: colleague@company.com
    主题: 请审查 PR #234
    操作: review
  - [会议安排确认] 来自: hr@company.com
    主题: 明天下午 2 点的面试安排
    操作: reply
```

### 59.4.2 案例二：批量处理客户邮件

使用搜索和摘要功能，Agent 能够快速了解与某个客户相关的所有邮件往来：

```
搜索 'from:client@example.com' 找到 8 封邮件：

- Re: 合同条款讨论：客户对付款条件有异议，希望延长至 60 天
- 项目进度报告：客户确认了本月的里程碑
- 技术问题反馈：客户报告了两个 bug
...
```

---

## 59.5 常见坑与解决方案

### 坑一：Gmail 不允许简单密码登录

**问题描述：** Gmail 从 2022 年起不再支持"允许不安全应用"的登录方式。

**解决方案：** 使用"应用专用密码"。在 Google 账号安全设置中开启两步验证后，可以生成一个专门用于第三方应用的密码。

### 坑二：邮件编码不统一

**问题描述：** 不同邮件客户端生成的邮件编码不同，解码后可能出现乱码。

**解决方案：** 在解码邮件头和正文时，始终尝试多种编码（utf-8, gbk, gb2312, latin-1），并使用 `errors="replace"` 避免解码异常。

### 坑三：HTML 邮件内容难以提取

**问题描述：** 很多营销邮件和通知邮件是纯 HTML 格式，直接提取文本效果很差。

**解决方案：** 使用 `beautifulsoup4` 库来解析 HTML 邮件，它能更好地处理复杂的 HTML 结构。对于特别复杂的邮件，可以先用浏览器渲染 HTML，然后再提取文本。

### 坑四：分类结果不稳定

**问题描述：** 同一封邮件，两次分类的结果可能不同。

**解决方案：** 使用 `temperature=0.0` 来提高输出的确定性。对于关键的分类决策，可以多次运行取多数结果（投票法）。

---

## 59.6 练习题

### 练习一：添加日历提取功能（难度：初级）

请为邮件 Agent 添加自动提取日程信息的功能。当邮件中提到会议或约会时间时，自动识别并生成日历事件。

### 练习二：实现邮件模板（难度：中级）

请实现邮件模板系统。用户可以预定义常用的回复模板（如"会议确认"、"项目延期通知"等），Agent 根据邮件内容自动选择合适的模板并填充具体内容。

### 练习三：构建邮件统计面板（难度：中级）

请实现邮件数据统计功能，包括：每日邮件数量趋势、发件人分布、邮件处理时间分析等。使用 matplotlib 生成可视化图表。

### 练习四：实现邮件自动转发规则（难度：高级）

请实现类似邮件客户端的自动转发规则。例如："所有来自 boss 的邮件，自动转发给 team-lead"，"主题包含'紧急'的邮件，发送短信提醒"。

### 练习五：添加多邮箱管理（难度：高级）

请支持同时管理多个邮箱账户，统一查看和处理所有邮箱的邮件。

---

## 59.7 实战任务

### 任务一：配置并测试邮件 Agent

在你自己的邮箱上配置并测试邮件 Agent。从简单的每日简报开始，逐步测试回复生成和行动项提取功能。记录 Agent 的准确率和你认为需要改进的地方。

### 任务二：优化分类效果

使用你自己过去一周的邮件作为测试集，评估分类器的准确率。找出分类错误的案例，分析错误原因，然后通过优化提示词来提高准确率。

### 任务三：构建自动化工作流

设计一个完整的邮件自动化工作流。例如：收到特定发件人的邮件时自动回复"已收到"，将营销邮件自动归档，将紧急邮件的关键信息推送到 Slack 或微信。

---

## 59.8 本章小结

邮件 Agent 是 Agent 在办公自动化领域最典型的应用之一。通过本章的项目实践，我们掌握了几个重要的技术要点。

第一，**邮件协议的安全对接是基础**。IMAP/SMTP 是成熟的协议，但 OAuth2 认证和应用专用密码等安全机制需要正确配置。

第二，**智能分类的核心是语义理解**。传统的规则过滤只能处理简单的场景，而基于 LLM 的分类器能够理解邮件的真实意图和情感，做出更准确的判断。

第三，**回复生成需要平衡质量和安全**。Agent 可以生成高质量的回复草稿，但最终的发送决策必须由人类做出。这种"人在回路"的设计是邮件 Agent 的安全保障。

下一章，我们将构建一个代码 Agent，将 Agent 的能力延伸到软件开发领域。
