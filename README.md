# TGReplyBot-
#!/usr/bin/env python

import asyncio
import logging
import os
import json
import hashlib
from datetime import datetime, timedelta, timezone

import openai
from telethon.errors import FloodWaitError, AuthKeyDuplicatedError
from telethon.tl.types import User, MessageService
from opentele.td import TDesktop
from opentele.api import API, UseCurrentSession
from opentele.exception import TFileNotFound

# Load environment variables
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "api_key")
ASSISTANT_ID = os.getenv("ASSISTANT_ID", "asst_vjWizQjt06NVFYtHwS6OX3b1")
PROXIES = os.getenv("PROXIES", "ansible.9qw.ru:8126:admin:password")
PROXY_TYPE = os.getenv("PROXY_TYPE", "http")
CHECK_OLD_MESSAGES_LIMIT = int(os.getenv("CHECK_OLD_MESSAGES_LIMIT", 20))
MESSAGES_LIMIT = int(os.getenv("MESSAGES_LIMIT", 10))
MONITOR_INTERVAL = int(os.getenv("MONITOR_INTERVAL", 30))
DIALOGS_LIMIT = int(os.getenv("DIALOGS_LIMIT", 10))
DIALOGS_INTERVAL = int(os.getenv("DIALOGS_INTERVAL", 10))
CHATGPT_LIMIT = int(os.getenv("CHATGPT_LIMIT", 4))
CHATGPT_WAIT_LIMIT = int(os.getenv("CHATGPT_WAIT_LIMIT", 60))
SEND_DELAYED = int(os.getenv("SEND_DELAYED", 1))
DELAY_MINUTES = float(os.getenv("DELAY_MINUTES", 60))
FORWARD_ENABLED = int(os.getenv("FORWARD_ENABLED", 1))
DELAYED_MESSAGE = os.getenv("DELAYED_MESSAGE", "Приветствую, вы определились по заказу? Может доставку или самовывоз на сегодня?")

NON_TEXT_REPLY = "Добрый день, напишите пожалуйста текстом, где вы находитесь и какой товар вас интересует?"
GROUP_CHAT_ID = -1002510370326
FORWARD_WAIT_TIME = int(os.getenv("FORWARD_WAIT_TIME", 30))
openai.api_key = OPENAI_API_KEY

# Logging configuration
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger()

class dotdict(dict):
    __getattr__ = dict.__getitem__

class Proxy:
    def __init__(self, proxy_type):
        if proxy_type not in ("http", "socks5"):
            raise ValueError("Invalid proxy type")
        self.proxy_type = proxy_type

    def get_conn(self):
        try:
            addr, port, username, password = PROXIES.split(",")[0].strip().split(":")
            return dotdict({
                "proxy_type": self.proxy_type,
                "addr": addr,
                "port": int(port),
                "username": username,
                "password": password
            })
        except Exception as e:
            logger.error("Proxy parse error: %s", e)
            return None

class MyTelegramClient:
    def __init__(self, tdata_name, proxy_type=PROXY_TYPE):
        self.tdata_name = tdata_name
        self.proxy_type = proxy_type
        self.client = None
        self.me = None

    async def authorize(self):
        tdata_path = "tdatas/tdata/"
        if not os.path.exists(tdata_path):
            logger.error("tdata path not found: %s", tdata_path)
            return False
        try:
            tdesk = TDesktop(tdata_path)
            if not tdesk.accounts:
                logger.error("No accounts in tdata")
                return False
        except TFileNotFound as e:
            logger.error("TFileNotFound: %s", e)
            return False

        proxy_conn = Proxy(self.proxy_type).get_conn()
        if not proxy_conn:
            return False
        session_hash = hashlib.md5(json.dumps(proxy_conn, sort_keys=True).encode()).hexdigest()
        session_file = f"sessions/{self.tdata_name}_{session_hash}.session"

        try:
            self.client = await tdesk.ToTelethon(
                session_file, UseCurrentSession,
                api=API.TelegramIOS.Generate(),
                proxy=proxy_conn, connection_retries=0, retry_delay=1,
                auto_reconnect=True, request_retries=0
            )
            await self.client.connect()
            self.me = await self.client.get_me()
            return self
        except (AuthKeyDuplicatedError, FloodWaitError, ConnectionError) as e:
            logger.error("Authorization error: %s", e)
            return False

threads_cache = {}
thread_after = {}

async def chat_with_openai(dialog_id, prompt):
    try:
        if dialog_id not in threads_cache:
            threads_cache[dialog_id] = openai.beta.threads.create().id
        openai.beta.threads.messages.create(thread_id=threads_cache[dialog_id], role="user", content=prompt)
        run = openai.beta.threads.runs.create(thread_id=threads_cache[dialog_id], assistant_id=ASSISTANT_ID)

        while True:
            status = openai.beta.threads.runs.retrieve(thread_id=threads_cache[dialog_id], run_id=run.id).status
            if status in ["completed", "expired", "cancelled", "failed"]:
                break
            await asyncio.sleep(1)

        after = thread_after.get(threads_cache[dialog_id])
        msgs = openai.beta.threads.messages.list(thread_id=threads_cache[dialog_id], before=after)
        for msg in msgs.data:
            if msg.role == "assistant":
                thread_after[threads_cache[dialog_id]] = msg.id
                return msg.content[0].text.value
        return "Нет ответа от ассистента."
    except FloodWaitError as e:
        await asyncio.sleep(e.seconds)
        return "FloodWaitError"
    except Exception as e:
        logger.error("ChatGPT error: %s", e)
        return f"Ошибка: {e}"

async def reconnect_if_disconnected(client):
    if not client.client.is_connected():
        try:
            await client.client.connect()
        except Exception as e:
            logger.error("Reconnect failed: %s", e)
            await asyncio.sleep(10)

# Additional functions like process_dialogue and main() should follow in similar cleaned-up style
# To complete the refactoring, these functions would need to be added similarly structured and validated.

if __name__ == "__main__":
    asyncio.run(main())
