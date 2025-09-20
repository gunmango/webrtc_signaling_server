import asyncio 
import websockets
import json
import os
from collections import defaultdict

rooms = defaultdict(lambda: {
    "websocket": set(),
    "uuid": set()
})

async def send_others(room_id, websocket, jsonText):
    for client in rooms[room_id]["websocket"]:
        if client != websocket:
            await client.send(jsonText)

def left_room(room_id, websocket, uuid):
    if websocket in rooms[room_id]["websocket"]:
        rooms[room_id]["websocket"].remove(websocket)
        rooms[room_id]["uuid"].remove(uuid)
        print(f"Client left room: {room_id} ({uuid})")
        if not rooms[room_id]:
            del rooms[room_id]

async def signaling_server(websocket):
    room_id = None
    uuid = None
    try:
        async for message in websocket:
            data = json.loads(message)

            if data["type"] == "join":
                room_id = data["room"]
                uuid = data["uuid"]
                rooms[room_id]["websocket"].add(websocket)
                rooms[room_id]["uuid"].add(uuid)
                print(f"Client joined room: {room_id} ({uuid})")

                await send_others(room_id, websocket, json.dumps({
                    "type": "new_client", "room": room_id, "uuid": uuid
                }))

            elif data["type"] in {"offer", "answer", "candidate"} and room_id:
                await send_others(room_id, websocket, message)

            elif data["type"] == "leave":
                left_room(data["room"], websocket, data["uuid"])

    except websockets.exceptions.ConnectionClosed:
        print("Client disconnected.")
    finally:
        if room_id and uuid:
            left_room(room_id, websocket, uuid)

async def main():
    port = int(os.environ.get("PORT", 8765))  # Render가 PORT를 환경변수로 지정함
    async with websockets.serve(signaling_server, "0.0.0.0", port):
        print(f"Signaling server started on ws://0.0.0.0:{port}")
        await asyncio.Future()  # run forever

if __name__ == "__main__":
    asyncio.run(main())