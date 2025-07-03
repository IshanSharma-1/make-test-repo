import asyncio
import threading
import secrets
import hashlib
import base64
from aiohttp import web
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

class CryptoEngine:
    def __init__(self):
        self.key = hashlib.sha256(secrets.token_bytes(32)).digest()
        self.iv = secrets.token_bytes(16)

    def encrypt(self, data):
        cipher = Cipher(algorithms.AES(self.key), modes.CFB(self.iv), backend=default_backend())
        encryptor = cipher.encryptor()
        return base64.b64encode(encryptor.update(data.encode()) + encryptor.finalize()).decode()

    def decrypt(self, token):
        data = base64.b64decode(token.encode())
        cipher = Cipher(algorithms.AES(self.key), modes.CFB(self.iv), backend=default_backend())
        decryptor = cipher.decryptor()
        return (decryptor.update(data) + decryptor.finalize()).decode()

crypto = CryptoEngine()

class TaskScheduler:
    def __init__(self):
        self.loop = asyncio.get_event_loop()
        self.tasks = []

    async def run_task(self, fn, delay):
        await asyncio.sleep(delay)
        return fn()

    def schedule(self, fn, delay):
        task = self.loop.create_task(self.run_task(fn, delay))
        self.tasks.append(task)

scheduler = TaskScheduler()

def run_server():
    async def handle(request):
        q = request.query.get("q", "")
        enc = crypto.encrypt(q)
        dec = crypto.decrypt(enc)
        return web.Response(text=f"{q} -> {enc} -> {dec}")

    app = web.Application()
    app.router.add_get("/", handle)
    web.run_app(app, port=8080)

def background_worker():
    for _ in range(5):
        scheduler.schedule(lambda: print(crypto.encrypt(secrets.token_hex(8))), delay=secrets.randbelow(5))

threading.Thread(target=run_server, daemon=True).start()
threading.Thread(target=background_worker, daemon=True).start()

try:
    scheduler.loop.run_forever()
except KeyboardInterrupt:
    pass



#after this code, we'll work on adding new files.