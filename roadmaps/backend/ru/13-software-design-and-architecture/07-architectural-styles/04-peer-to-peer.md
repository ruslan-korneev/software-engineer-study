# Peer-to-Peer Architecture

## Что такое P2P архитектура?

Peer-to-Peer (P2P) архитектура — это распределённая архитектурная модель, в которой все участники сети (пиры) равноправны и могут выступать как в роли клиента, так и в роли сервера. В отличие от клиент-серверной модели, здесь нет центрального сервера — каждый узел может предоставлять и потреблять ресурсы.

> "P2P — это архитектура, в которой каждый узел является и клиентом, и сервером одновременно."

## Основные концепции

```
Клиент-сервер:                    P2P:

┌────────┐                        ┌────────┐ ◄───► ┌────────┐
│ Client │                        │  Peer  │       │  Peer  │
└───┬────┘                        └────┬───┘       └───┬────┘
    │                                  │               │
    ▼                                  │               │
┌────────┐                             ▼               ▼
│ Server │                        ┌────────┐ ◄───► ┌────────┐
└────────┘                        │  Peer  │       │  Peer  │
    ▲                             └────────┘       └────────┘
    │                                  ▲               ▲
┌───┴────┐                             │               │
│ Client │                             └───────┬───────┘
└────────┘                                     │
                                          ┌────┴───┐
                                          │  Peer  │
                                          └────────┘
```

## Типы P2P архитектуры

### Pure P2P (Чистая P2P)

В чистой P2P сети нет никакой центральной инфраструктуры. Все пиры равны и самоорганизуются.

```python
# Пример чистого P2P узла

import asyncio
import json
from dataclasses import dataclass
from typing import Dict, Set, List
import hashlib

@dataclass
class Peer:
    host: str
    port: int
    peer_id: str = ""

    def __post_init__(self):
        if not self.peer_id:
            self.peer_id = hashlib.sha256(
                f"{self.host}:{self.port}".encode()
            ).hexdigest()[:16]

    def __hash__(self):
        return hash(self.peer_id)

class PurePeerNode:
    """Узел чистой P2P сети."""

    def __init__(self, host: str, port: int):
        self.me = Peer(host, port)
        self.known_peers: Set[Peer] = set()
        self.shared_files: Dict[str, bytes] = {}
        self.server = None

    async def start(self):
        """Запуск P2P узла."""
        self.server = await asyncio.start_server(
            self._handle_connection,
            self.me.host,
            self.me.port
        )
        print(f"P2P Node started at {self.me.host}:{self.me.port}")

    async def _handle_connection(self, reader, writer):
        """Обработка входящего соединения."""
        data = await reader.read(4096)
        message = json.loads(data.decode())

        response = await self._process_message(message)

        writer.write(json.dumps(response).encode())
        await writer.drain()
        writer.close()

    async def _process_message(self, message: dict) -> dict:
        """Обработка сообщения от другого пира."""
        msg_type = message.get("type")

        if msg_type == "ping":
            return {"type": "pong", "peer_id": self.me.peer_id}

        elif msg_type == "get_peers":
            # Возвращаем известных пиров
            return {
                "type": "peers",
                "peers": [
                    {"host": p.host, "port": p.port, "peer_id": p.peer_id}
                    for p in self.known_peers
                ]
            }

        elif msg_type == "search_file":
            file_hash = message.get("file_hash")
            if file_hash in self.shared_files:
                return {
                    "type": "file_found",
                    "peer_id": self.me.peer_id,
                    "file_hash": file_hash
                }
            return {"type": "file_not_found"}

        elif msg_type == "get_file":
            file_hash = message.get("file_hash")
            if file_hash in self.shared_files:
                return {
                    "type": "file_data",
                    "data": self.shared_files[file_hash].hex()
                }
            return {"type": "error", "message": "File not found"}

        return {"type": "error", "message": "Unknown message type"}

    async def connect_to_peer(self, host: str, port: int):
        """Подключение к другому пиру."""
        try:
            reader, writer = await asyncio.open_connection(host, port)

            # Пинг для проверки соединения
            writer.write(json.dumps({"type": "ping"}).encode())
            await writer.drain()

            data = await reader.read(4096)
            response = json.loads(data.decode())

            if response.get("type") == "pong":
                peer = Peer(host, port, response.get("peer_id"))
                self.known_peers.add(peer)
                print(f"Connected to peer: {peer.peer_id}")

            writer.close()
            return True
        except Exception as e:
            print(f"Failed to connect to {host}:{port}: {e}")
            return False

    async def discover_peers(self):
        """Обнаружение новых пиров через известных."""
        for peer in list(self.known_peers):
            try:
                reader, writer = await asyncio.open_connection(
                    peer.host, peer.port
                )

                writer.write(json.dumps({"type": "get_peers"}).encode())
                await writer.drain()

                data = await reader.read(4096)
                response = json.loads(data.decode())

                for p in response.get("peers", []):
                    new_peer = Peer(p["host"], p["port"], p["peer_id"])
                    if new_peer.peer_id != self.me.peer_id:
                        self.known_peers.add(new_peer)

                writer.close()
            except Exception:
                # Пир недоступен - удаляем
                self.known_peers.discard(peer)

    def share_file(self, file_path: str):
        """Расшаривание файла в сети."""
        with open(file_path, 'rb') as f:
            content = f.read()
            file_hash = hashlib.sha256(content).hexdigest()
            self.shared_files[file_hash] = content
            return file_hash

    async def search_file(self, file_hash: str) -> List[Peer]:
        """Поиск файла в сети."""
        peers_with_file = []

        for peer in self.known_peers:
            try:
                reader, writer = await asyncio.open_connection(
                    peer.host, peer.port
                )

                writer.write(json.dumps({
                    "type": "search_file",
                    "file_hash": file_hash
                }).encode())
                await writer.drain()

                data = await reader.read(4096)
                response = json.loads(data.decode())

                if response.get("type") == "file_found":
                    peers_with_file.append(peer)

                writer.close()
            except Exception:
                pass

        return peers_with_file

    async def download_file(self, file_hash: str, peer: Peer) -> bytes:
        """Скачивание файла от пира."""
        reader, writer = await asyncio.open_connection(peer.host, peer.port)

        writer.write(json.dumps({
            "type": "get_file",
            "file_hash": file_hash
        }).encode())
        await writer.drain()

        data = await reader.read(1024 * 1024)  # 1MB max
        response = json.loads(data.decode())

        writer.close()

        if response.get("type") == "file_data":
            return bytes.fromhex(response["data"])

        raise Exception("Failed to download file")
```

### Hybrid P2P (Гибридная P2P)

Гибридная P2P использует центральные серверы для координации, но передача данных происходит напрямую между пирами.

```python
# Гибридная P2P с центральным трекером

import asyncio
import json
from typing import Dict, List, Set
from dataclasses import dataclass, field
from datetime import datetime, timedelta

# === Центральный трекер ===

@dataclass
class PeerInfo:
    peer_id: str
    host: str
    port: int
    files: Set[str] = field(default_factory=set)
    last_seen: datetime = field(default_factory=datetime.now)

class Tracker:
    """Центральный сервер для координации пиров."""

    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.peers: Dict[str, PeerInfo] = {}
        self.file_index: Dict[str, Set[str]] = {}  # file_hash -> peer_ids

    async def start(self):
        server = await asyncio.start_server(
            self._handle_request,
            self.host,
            self.port
        )
        print(f"Tracker started at {self.host}:{self.port}")

        # Запускаем очистку неактивных пиров
        asyncio.create_task(self._cleanup_inactive_peers())

        async with server:
            await server.serve_forever()

    async def _handle_request(self, reader, writer):
        data = await reader.read(4096)
        request = json.loads(data.decode())

        response = self._process_request(request)

        writer.write(json.dumps(response).encode())
        await writer.drain()
        writer.close()

    def _process_request(self, request: dict) -> dict:
        action = request.get("action")

        if action == "register":
            # Регистрация нового пира
            peer_id = request["peer_id"]
            self.peers[peer_id] = PeerInfo(
                peer_id=peer_id,
                host=request["host"],
                port=request["port"]
            )
            return {"status": "ok", "message": "Registered"}

        elif action == "announce":
            # Объявление о наличии файла
            peer_id = request["peer_id"]
            file_hash = request["file_hash"]

            if peer_id in self.peers:
                self.peers[peer_id].files.add(file_hash)
                self.peers[peer_id].last_seen = datetime.now()

                if file_hash not in self.file_index:
                    self.file_index[file_hash] = set()
                self.file_index[file_hash].add(peer_id)

            return {"status": "ok"}

        elif action == "get_peers":
            # Получение списка пиров с файлом
            file_hash = request["file_hash"]
            peer_ids = self.file_index.get(file_hash, set())

            peers = [
                {
                    "peer_id": pid,
                    "host": self.peers[pid].host,
                    "port": self.peers[pid].port
                }
                for pid in peer_ids
                if pid in self.peers
            ]

            return {"status": "ok", "peers": peers}

        elif action == "heartbeat":
            # Обновление статуса пира
            peer_id = request["peer_id"]
            if peer_id in self.peers:
                self.peers[peer_id].last_seen = datetime.now()
            return {"status": "ok"}

        return {"status": "error", "message": "Unknown action"}

    async def _cleanup_inactive_peers(self):
        """Удаление неактивных пиров."""
        while True:
            await asyncio.sleep(60)
            now = datetime.now()
            inactive_threshold = timedelta(minutes=5)

            inactive_peers = [
                peer_id for peer_id, info in self.peers.items()
                if now - info.last_seen > inactive_threshold
            ]

            for peer_id in inactive_peers:
                # Удаляем из индекса файлов
                for file_hash in self.peers[peer_id].files:
                    if file_hash in self.file_index:
                        self.file_index[file_hash].discard(peer_id)

                del self.peers[peer_id]
                print(f"Removed inactive peer: {peer_id}")


# === P2P клиент ===

class HybridPeerClient:
    """Клиент гибридной P2P сети."""

    def __init__(self, peer_id: str, host: str, port: int, tracker_url: str):
        self.peer_id = peer_id
        self.host = host
        self.port = port
        self.tracker_host, self.tracker_port = tracker_url.split(":")
        self.tracker_port = int(self.tracker_port)
        self.shared_files: Dict[str, bytes] = {}

    async def register(self):
        """Регистрация на трекере."""
        response = await self._tracker_request({
            "action": "register",
            "peer_id": self.peer_id,
            "host": self.host,
            "port": self.port
        })
        return response.get("status") == "ok"

    async def announce_file(self, file_hash: str):
        """Объявление о наличии файла."""
        return await self._tracker_request({
            "action": "announce",
            "peer_id": self.peer_id,
            "file_hash": file_hash
        })

    async def find_peers(self, file_hash: str) -> List[dict]:
        """Поиск пиров с файлом через трекер."""
        response = await self._tracker_request({
            "action": "get_peers",
            "file_hash": file_hash
        })
        return response.get("peers", [])

    async def _tracker_request(self, data: dict) -> dict:
        """Отправка запроса трекеру."""
        reader, writer = await asyncio.open_connection(
            self.tracker_host,
            self.tracker_port
        )

        writer.write(json.dumps(data).encode())
        await writer.drain()

        response_data = await reader.read(4096)
        writer.close()

        return json.loads(response_data.decode())

    async def download_from_peer(
        self,
        peer: dict,
        file_hash: str
    ) -> bytes:
        """Прямое скачивание файла от пира."""
        reader, writer = await asyncio.open_connection(
            peer["host"],
            peer["port"]
        )

        writer.write(json.dumps({
            "action": "get_file",
            "file_hash": file_hash
        }).encode())
        await writer.drain()

        # Получаем данные напрямую от пира
        chunks = []
        while True:
            chunk = await reader.read(8192)
            if not chunk:
                break
            chunks.append(chunk)

        writer.close()
        return b''.join(chunks)

    async def start_heartbeat(self):
        """Периодические heartbeat сигналы трекеру."""
        while True:
            await asyncio.sleep(60)
            await self._tracker_request({
                "action": "heartbeat",
                "peer_id": self.peer_id
            })
```

### Structured P2P (DHT)

Distributed Hash Table (DHT) — структурированная P2P сеть, где данные распределяются по узлам согласно хеш-функции.

```python
# Упрощённая реализация Kademlia DHT

import hashlib
import heapq
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
import asyncio
import json

# Размер идентификатора в битах
ID_BITS = 160

@dataclass
class Node:
    node_id: bytes
    host: str
    port: int

    def __hash__(self):
        return hash(self.node_id)

    def distance(self, other_id: bytes) -> int:
        """XOR расстояние между узлами."""
        return int.from_bytes(self.node_id, 'big') ^ int.from_bytes(other_id, 'big')

class KBucket:
    """K-bucket для хранения контактов."""

    def __init__(self, k: int = 20):
        self.k = k
        self.nodes: List[Node] = []

    def add(self, node: Node):
        if node in self.nodes:
            self.nodes.remove(node)
            self.nodes.append(node)  # Перемещаем в конец (LRU)
        elif len(self.nodes) < self.k:
            self.nodes.append(node)

    def get_nodes(self) -> List[Node]:
        return list(self.nodes)

class RoutingTable:
    """Таблица маршрутизации Kademlia."""

    def __init__(self, local_id: bytes, k: int = 20):
        self.local_id = local_id
        self.k = k
        self.buckets: List[KBucket] = [KBucket(k) for _ in range(ID_BITS)]

    def get_bucket_index(self, node_id: bytes) -> int:
        """Определение индекса bucket по XOR расстоянию."""
        distance = int.from_bytes(self.local_id, 'big') ^ int.from_bytes(node_id, 'big')
        if distance == 0:
            return 0
        return distance.bit_length() - 1

    def add_node(self, node: Node):
        """Добавление узла в таблицу маршрутизации."""
        if node.node_id == self.local_id:
            return

        index = self.get_bucket_index(node.node_id)
        self.buckets[index].add(node)

    def find_closest(self, target_id: bytes, count: int = 20) -> List[Node]:
        """Поиск ближайших узлов к target_id."""
        all_nodes = []
        for bucket in self.buckets:
            all_nodes.extend(bucket.get_nodes())

        # Сортировка по XOR расстоянию
        target_int = int.from_bytes(target_id, 'big')

        def distance(node):
            return int.from_bytes(node.node_id, 'big') ^ target_int

        all_nodes.sort(key=distance)
        return all_nodes[:count]

class KademliaDHT:
    """Узел Kademlia DHT."""

    def __init__(self, host: str, port: int, bootstrap_nodes: List[Tuple[str, int]] = None):
        # Генерируем ID узла
        self.node_id = hashlib.sha1(f"{host}:{port}".encode()).digest()
        self.host = host
        self.port = port
        self.routing_table = RoutingTable(self.node_id)
        self.storage: Dict[bytes, bytes] = {}
        self.bootstrap_nodes = bootstrap_nodes or []

    async def start(self):
        """Запуск DHT узла."""
        server = await asyncio.start_server(
            self._handle_rpc,
            self.host,
            self.port
        )

        # Bootstrap: подключаемся к известным узлам
        for host, port in self.bootstrap_nodes:
            await self._bootstrap(host, port)

        print(f"DHT Node {self.node_id.hex()[:8]} started at {self.host}:{self.port}")

        async with server:
            await server.serve_forever()

    async def _bootstrap(self, host: str, port: int):
        """Подключение к bootstrap узлу."""
        try:
            # Запрашиваем ближайшие узлы к себе
            response = await self._send_rpc(host, port, {
                "method": "find_node",
                "target": self.node_id.hex()
            })

            if response:
                for node_data in response.get("nodes", []):
                    node = Node(
                        bytes.fromhex(node_data["node_id"]),
                        node_data["host"],
                        node_data["port"]
                    )
                    self.routing_table.add_node(node)

        except Exception as e:
            print(f"Bootstrap failed: {e}")

    async def _handle_rpc(self, reader, writer):
        """Обработка RPC запроса."""
        data = await reader.read(4096)
        request = json.loads(data.decode())

        # Добавляем отправителя в таблицу маршрутизации
        if "sender" in request:
            sender = Node(
                bytes.fromhex(request["sender"]["node_id"]),
                request["sender"]["host"],
                request["sender"]["port"]
            )
            self.routing_table.add_node(sender)

        response = self._process_rpc(request)

        writer.write(json.dumps(response).encode())
        await writer.drain()
        writer.close()

    def _process_rpc(self, request: dict) -> dict:
        method = request.get("method")

        if method == "ping":
            return {"result": "pong", "node_id": self.node_id.hex()}

        elif method == "find_node":
            target = bytes.fromhex(request["target"])
            closest = self.routing_table.find_closest(target)
            return {
                "result": "nodes",
                "nodes": [
                    {"node_id": n.node_id.hex(), "host": n.host, "port": n.port}
                    for n in closest
                ]
            }

        elif method == "find_value":
            key = bytes.fromhex(request["key"])
            if key in self.storage:
                return {"result": "value", "value": self.storage[key].hex()}
            else:
                closest = self.routing_table.find_closest(key)
                return {
                    "result": "nodes",
                    "nodes": [
                        {"node_id": n.node_id.hex(), "host": n.host, "port": n.port}
                        for n in closest
                    ]
                }

        elif method == "store":
            key = bytes.fromhex(request["key"])
            value = bytes.fromhex(request["value"])
            self.storage[key] = value
            return {"result": "ok"}

        return {"error": "Unknown method"}

    async def _send_rpc(self, host: str, port: int, request: dict) -> Optional[dict]:
        """Отправка RPC запроса."""
        try:
            reader, writer = await asyncio.open_connection(host, port)

            request["sender"] = {
                "node_id": self.node_id.hex(),
                "host": self.host,
                "port": self.port
            }

            writer.write(json.dumps(request).encode())
            await writer.drain()

            data = await reader.read(4096)
            writer.close()

            return json.loads(data.decode())
        except Exception:
            return None

    async def store(self, key: str, value: bytes):
        """Сохранение значения в DHT."""
        key_hash = hashlib.sha1(key.encode()).digest()

        # Находим ближайшие узлы к ключу
        closest = self.routing_table.find_closest(key_hash)

        # Сохраняем на k ближайших узлах
        for node in closest:
            await self._send_rpc(node.host, node.port, {
                "method": "store",
                "key": key_hash.hex(),
                "value": value.hex()
            })

        # Сохраняем и у себя, если мы близки
        self.storage[key_hash] = value

    async def get(self, key: str) -> Optional[bytes]:
        """Получение значения из DHT."""
        key_hash = hashlib.sha1(key.encode()).digest()

        # Сначала проверяем локальное хранилище
        if key_hash in self.storage:
            return self.storage[key_hash]

        # Итеративный поиск
        queried = set()
        to_query = self.routing_table.find_closest(key_hash)

        while to_query:
            node = to_query.pop(0)

            if node.node_id in queried:
                continue

            queried.add(node.node_id)

            response = await self._send_rpc(node.host, node.port, {
                "method": "find_value",
                "key": key_hash.hex()
            })

            if response:
                if response.get("result") == "value":
                    return bytes.fromhex(response["value"])

                # Добавляем новые узлы для запроса
                for node_data in response.get("nodes", []):
                    new_node = Node(
                        bytes.fromhex(node_data["node_id"]),
                        node_data["host"],
                        node_data["port"]
                    )
                    if new_node.node_id not in queried:
                        to_query.append(new_node)

                # Сортируем по расстоянию
                to_query.sort(key=lambda n: n.distance(key_hash))

        return None
```

## BitTorrent-подобный протокол

```python
# Упрощённая реализация BitTorrent-подобного протокола

import hashlib
import asyncio
import json
from dataclasses import dataclass, field
from typing import Dict, List, Set, Optional
from enum import Enum
import os

PIECE_SIZE = 256 * 1024  # 256 KB

@dataclass
class TorrentMetadata:
    """Метаданные торрента."""
    name: str
    total_size: int
    piece_size: int
    pieces: List[str]  # Хеши кусков
    info_hash: str = ""

    def __post_init__(self):
        if not self.info_hash:
            content = f"{self.name}{self.total_size}{self.piece_size}{''.join(self.pieces)}"
            self.info_hash = hashlib.sha1(content.encode()).hexdigest()

class PeerState(Enum):
    CHOKED = "choked"
    INTERESTED = "interested"
    UNCHOKED = "unchoked"

@dataclass
class PeerConnection:
    host: str
    port: int
    peer_id: str
    bitfield: Set[int] = field(default_factory=set)
    state: PeerState = PeerState.CHOKED

class BitTorrentClient:
    """Клиент BitTorrent-подобного протокола."""

    def __init__(self, peer_id: str, host: str, port: int):
        self.peer_id = peer_id
        self.host = host
        self.port = port
        self.torrents: Dict[str, TorrentMetadata] = {}
        self.downloaded_pieces: Dict[str, Set[int]] = {}
        self.piece_data: Dict[str, Dict[int, bytes]] = {}
        self.connections: Dict[str, List[PeerConnection]] = {}

    def create_torrent(self, file_path: str) -> TorrentMetadata:
        """Создание торрент-файла."""
        with open(file_path, 'rb') as f:
            data = f.read()

        pieces = []
        piece_hashes = []

        for i in range(0, len(data), PIECE_SIZE):
            piece = data[i:i + PIECE_SIZE]
            piece_hash = hashlib.sha1(piece).hexdigest()
            pieces.append(piece)
            piece_hashes.append(piece_hash)

        metadata = TorrentMetadata(
            name=os.path.basename(file_path),
            total_size=len(data),
            piece_size=PIECE_SIZE,
            pieces=piece_hashes
        )

        self.torrents[metadata.info_hash] = metadata
        self.downloaded_pieces[metadata.info_hash] = set(range(len(pieces)))
        self.piece_data[metadata.info_hash] = {i: p for i, p in enumerate(pieces)}

        return metadata

    async def start_seeding(self, info_hash: str):
        """Начало раздачи торрента."""
        server = await asyncio.start_server(
            lambda r, w: self._handle_peer_connection(r, w, info_hash),
            self.host,
            self.port
        )

        print(f"Seeding {info_hash} at {self.host}:{self.port}")

        async with server:
            await server.serve_forever()

    async def _handle_peer_connection(self, reader, writer, info_hash: str):
        """Обработка подключения пира."""
        # Handshake
        handshake = await reader.read(68)

        # Отправляем свой handshake
        writer.write(self._create_handshake(info_hash))
        await writer.drain()

        # Отправляем bitfield
        bitfield = self._create_bitfield(info_hash)
        writer.write(json.dumps({
            "type": "bitfield",
            "pieces": list(self.downloaded_pieces.get(info_hash, set()))
        }).encode())
        await writer.drain()

        # Обработка запросов
        while True:
            try:
                data = await reader.read(4096)
                if not data:
                    break

                message = json.loads(data.decode())
                await self._handle_message(reader, writer, message, info_hash)
            except Exception:
                break

        writer.close()

    async def _handle_message(self, reader, writer, message: dict, info_hash: str):
        """Обработка сообщения от пира."""
        msg_type = message.get("type")

        if msg_type == "request":
            # Запрос куска
            piece_index = message["piece"]
            if piece_index in self.piece_data.get(info_hash, {}):
                piece = self.piece_data[info_hash][piece_index]
                writer.write(json.dumps({
                    "type": "piece",
                    "index": piece_index,
                    "data": piece.hex()
                }).encode())
                await writer.drain()

        elif msg_type == "interested":
            # Пир заинтересован
            writer.write(json.dumps({"type": "unchoke"}).encode())
            await writer.drain()

    def _create_handshake(self, info_hash: str) -> bytes:
        """Создание handshake сообщения."""
        return json.dumps({
            "type": "handshake",
            "info_hash": info_hash,
            "peer_id": self.peer_id
        }).encode()

    def _create_bitfield(self, info_hash: str) -> Set[int]:
        """Создание bitfield."""
        return self.downloaded_pieces.get(info_hash, set())

    async def download(self, metadata: TorrentMetadata, peers: List[dict]):
        """Скачивание торрента."""
        info_hash = metadata.info_hash
        self.torrents[info_hash] = metadata
        self.downloaded_pieces[info_hash] = set()
        self.piece_data[info_hash] = {}

        total_pieces = len(metadata.pieces)
        needed_pieces = set(range(total_pieces))

        # Подключаемся к пирам
        for peer in peers:
            asyncio.create_task(
                self._download_from_peer(peer, info_hash, needed_pieces)
            )

        # Ждём завершения скачивания
        while len(self.downloaded_pieces[info_hash]) < total_pieces:
            await asyncio.sleep(0.1)

        # Собираем файл
        return self._assemble_file(info_hash)

    async def _download_from_peer(
        self,
        peer: dict,
        info_hash: str,
        needed_pieces: Set[int]
    ):
        """Скачивание кусков от конкретного пира."""
        try:
            reader, writer = await asyncio.open_connection(
                peer["host"],
                peer["port"]
            )

            # Handshake
            writer.write(self._create_handshake(info_hash))
            await writer.drain()

            handshake = await reader.read(4096)

            # Получаем bitfield пира
            bitfield_data = await reader.read(4096)
            peer_bitfield = set(json.loads(bitfield_data.decode()).get("pieces", []))

            # Отправляем interested
            writer.write(json.dumps({"type": "interested"}).encode())
            await writer.drain()

            # Ждём unchoke
            unchoke = await reader.read(4096)

            # Запрашиваем куски
            available = peer_bitfield & needed_pieces

            for piece_index in available:
                if piece_index in self.downloaded_pieces[info_hash]:
                    continue

                # Запрашиваем кусок
                writer.write(json.dumps({
                    "type": "request",
                    "piece": piece_index
                }).encode())
                await writer.drain()

                # Получаем кусок
                response = await reader.read(PIECE_SIZE + 1024)
                piece_message = json.loads(response.decode())

                if piece_message.get("type") == "piece":
                    piece_data = bytes.fromhex(piece_message["data"])

                    # Проверяем хеш
                    piece_hash = hashlib.sha1(piece_data).hexdigest()
                    expected_hash = self.torrents[info_hash].pieces[piece_index]

                    if piece_hash == expected_hash:
                        self.piece_data[info_hash][piece_index] = piece_data
                        self.downloaded_pieces[info_hash].add(piece_index)
                        needed_pieces.discard(piece_index)
                        print(f"Downloaded piece {piece_index}")

            writer.close()

        except Exception as e:
            print(f"Error downloading from peer: {e}")

    def _assemble_file(self, info_hash: str) -> bytes:
        """Сборка файла из кусков."""
        pieces = self.piece_data[info_hash]
        data = b''.join(pieces[i] for i in sorted(pieces.keys()))
        return data


# Пример использования
async def main():
    # Создание торрента
    seeder = BitTorrentClient("seeder1", "localhost", 6881)
    metadata = seeder.create_torrent("example.txt")

    print(f"Created torrent: {metadata.info_hash}")

    # Запуск раздачи
    asyncio.create_task(seeder.start_seeding(metadata.info_hash))

    await asyncio.sleep(1)

    # Скачивание
    leecher = BitTorrentClient("leecher1", "localhost", 6882)
    data = await leecher.download(metadata, [{"host": "localhost", "port": 6881}])

    print(f"Downloaded {len(data)} bytes")

# asyncio.run(main())
```

## Преимущества P2P архитектуры

### 1. Масштабируемость

```python
# Чем больше пиров, тем больше пропускная способность сети

class ScalablePeerNetwork:
    def calculate_network_bandwidth(self, peer_count: int, avg_bandwidth: float):
        """
        В P2P сети общая пропускная способность растёт линейно
        с количеством участников.
        """
        return peer_count * avg_bandwidth

    def compare_architectures(self, peer_count: int):
        avg_peer_bandwidth = 10  # Mbps

        # Клиент-сервер: ограничено пропускной способностью сервера
        server_bandwidth = 1000  # Mbps
        client_server_capacity = min(server_bandwidth, peer_count * avg_peer_bandwidth)

        # P2P: растёт с количеством пиров
        p2p_capacity = self.calculate_network_bandwidth(peer_count, avg_peer_bandwidth)

        return {
            "client_server": client_server_capacity,
            "p2p": p2p_capacity
        }
```

### 2. Отказоустойчивость

```python
# Нет единой точки отказа

class FaultTolerantP2P:
    def __init__(self, replication_factor: int = 3):
        self.replication_factor = replication_factor

    async def store_with_replication(self, key: str, value: bytes, peers: List[Node]):
        """Хранение с репликацией на несколько узлов."""
        stored_count = 0

        for peer in peers[:self.replication_factor]:
            try:
                await self._store_on_peer(peer, key, value)
                stored_count += 1
            except Exception:
                continue

        if stored_count == 0:
            raise Exception("Failed to store data")

        return stored_count

    async def get_with_fallback(self, key: str, peers: List[Node]) -> Optional[bytes]:
        """Получение с fallback на другие узлы."""
        for peer in peers:
            try:
                value = await self._get_from_peer(peer, key)
                if value:
                    return value
            except Exception:
                continue

        return None
```

### 3. Децентрализация

```python
# Нет центральной точки контроля

class DecentralizedNetwork:
    def __init__(self):
        self.governance_votes: Dict[str, Dict[str, bool]] = {}

    async def propose_change(self, proposal_id: str, proposal: dict):
        """Предложение изменения в сети."""
        self.governance_votes[proposal_id] = {}
        # Рассылка предложения всем пирам
        await self.broadcast({
            "type": "proposal",
            "id": proposal_id,
            "content": proposal
        })

    def vote(self, proposal_id: str, peer_id: str, approve: bool):
        """Голосование за предложение."""
        if proposal_id not in self.governance_votes:
            return

        self.governance_votes[proposal_id][peer_id] = approve

    def check_consensus(self, proposal_id: str, threshold: float = 0.67) -> bool:
        """Проверка достижения консенсуса."""
        votes = self.governance_votes.get(proposal_id, {})
        if not votes:
            return False

        approvals = sum(1 for v in votes.values() if v)
        return approvals / len(votes) >= threshold
```

## Недостатки P2P архитектуры

### 1. Сложность поиска

```python
# Поиск в P2P сети может быть медленным

class SearchComplexity:
    def flooding_search(self, ttl: int, avg_connections: int):
        """
        Flooding: O(n) сообщений
        Очень неэффективно для больших сетей.
        """
        messages = 0
        for hop in range(ttl):
            messages += avg_connections ** hop
        return messages

    def dht_search(self, network_size: int):
        """
        DHT (Kademlia): O(log n) хопов
        Гораздо эффективнее для больших сетей.
        """
        import math
        return math.log2(network_size)
```

### 2. Безопасность

```python
# P2P сети уязвимы к различным атакам

class SecurityConcerns:
    """Типичные угрозы безопасности в P2P."""

    def sybil_attack(self):
        """
        Sybil атака: злоумышленник создаёт множество
        поддельных узлов для контроля части сети.
        """
        pass

    def eclipse_attack(self):
        """
        Eclipse атака: изоляция узла путём контроля
        всех его соединений.
        """
        pass

    def content_poisoning(self):
        """
        Отравление контента: распространение
        вредоносных данных с правильными метаданными.
        """
        pass


# Защита от Sybil атаки
class SybilResistance:
    def __init__(self):
        self.reputation_scores: Dict[str, float] = {}

    def calculate_reputation(self, peer_id: str, history: List[dict]) -> float:
        """Расчёт репутации на основе истории."""
        if not history:
            return 0.0

        successful = sum(1 for h in history if h["success"])
        return successful / len(history)

    def should_trust_peer(self, peer_id: str, min_reputation: float = 0.5) -> bool:
        """Проверка, стоит ли доверять пиру."""
        return self.reputation_scores.get(peer_id, 0.0) >= min_reputation
```

## Best Practices

### 1. Используйте DHT для эффективного поиска

```python
# DHT обеспечивает O(log n) поиск

class EfficientLookup:
    async def iterative_find(self, target_id: bytes, alpha: int = 3):
        """
        Итеративный поиск с параллельными запросами.
        alpha - количество параллельных запросов.
        """
        closest = self.routing_table.find_closest(target_id, alpha)
        queried = set()

        while True:
            # Параллельные запросы к alpha узлам
            tasks = []
            for node in closest[:alpha]:
                if node.node_id not in queried:
                    tasks.append(self._query_node(node, target_id))
                    queried.add(node.node_id)

            if not tasks:
                break

            results = await asyncio.gather(*tasks, return_exceptions=True)

            # Обновляем список ближайших
            for result in results:
                if isinstance(result, list):
                    for node in result:
                        if node not in closest:
                            closest.append(node)

            closest.sort(key=lambda n: n.distance(target_id))
            closest = closest[:20]  # Оставляем k ближайших

        return closest
```

### 2. Реализуйте репликацию данных

```python
# Репликация для надёжности

class DataReplication:
    def __init__(self, replication_factor: int = 3):
        self.k = replication_factor

    async def replicate(self, key: bytes, value: bytes, dht: KademliaDHT):
        """Репликация данных на k ближайших узлов."""
        closest = dht.routing_table.find_closest(key, self.k)

        tasks = [
            dht._send_rpc(node.host, node.port, {
                "method": "store",
                "key": key.hex(),
                "value": value.hex()
            })
            for node in closest
        ]

        results = await asyncio.gather(*tasks, return_exceptions=True)
        successful = sum(1 for r in results if r and r.get("result") == "ok")

        return successful >= self.k // 2 + 1  # Большинство
```

### 3. Проверяйте целостность данных

```python
# Merkle Tree для проверки целостности

class MerkleTree:
    def __init__(self, data_blocks: List[bytes]):
        self.leaves = [hashlib.sha256(block).digest() for block in data_blocks]
        self.tree = self._build_tree()

    def _build_tree(self) -> List[List[bytes]]:
        tree = [self.leaves]

        while len(tree[-1]) > 1:
            level = tree[-1]
            new_level = []

            for i in range(0, len(level), 2):
                if i + 1 < len(level):
                    combined = level[i] + level[i + 1]
                else:
                    combined = level[i] + level[i]
                new_level.append(hashlib.sha256(combined).digest())

            tree.append(new_level)

        return tree

    @property
    def root(self) -> bytes:
        return self.tree[-1][0]

    def get_proof(self, index: int) -> List[Tuple[bytes, bool]]:
        """Получение Merkle proof для листа."""
        proof = []
        for level in self.tree[:-1]:
            sibling_index = index ^ 1
            if sibling_index < len(level):
                is_right = index % 2 == 0
                proof.append((level[sibling_index], is_right))
            index //= 2
        return proof

    @staticmethod
    def verify_proof(leaf: bytes, proof: List[Tuple[bytes, bool]], root: bytes) -> bool:
        """Проверка Merkle proof."""
        current = hashlib.sha256(leaf).digest()

        for sibling, is_right in proof:
            if is_right:
                combined = current + sibling
            else:
                combined = sibling + current
            current = hashlib.sha256(combined).digest()

        return current == root
```

## Когда использовать P2P?

### Подходит для:

1. **Распределённого хранения файлов** (BitTorrent, IPFS)
2. **Криптовалют и блокчейна** (Bitcoin, Ethereum)
3. **Децентрализованных приложений** (dApps)
4. **Real-time коммуникаций** (WebRTC)
5. **CDN и доставки контента**

### Не подходит для:

1. **Систем с требованиями к централизованному контролю**
2. **Приложений с высокими требованиями к консистентности**
3. **Систем с жёсткими требованиями к безопасности**
4. **Простых клиент-серверных приложений**

## Заключение

P2P архитектура — мощный инструмент для создания масштабируемых и отказоустойчивых распределённых систем. Она особенно эффективна для приложений, где децентрализация является ключевым требованием. Однако P2P системы сложнее в разработке и требуют тщательного проектирования для обеспечения безопасности и эффективности поиска.
