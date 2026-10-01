import asyncio
import json
import websockets

# Live rooms dictionary: { room_id: { "host": ws, "guest": ws, "current_q": 0 } }
ROOMS = {}

# Sample KBC Questions tailored for Class 7 (Science & Social Science)
CLASS_7_QUESTIONS = [
    {
        "question": "Which of the following is a parasite plant?",
        "options": ["Cuscuta (Amarbel)", "Rose", "Pitcher plant", "Algae"],
        "answer": "Cuscuta (Amarbel)"
    },
    {
        "question": "Who wrote the 'Akbarnama' during the Mughal Period?",
        "options": ["Akbar", "Birbal", "Abul Fazl", "Al-Biruni"],
        "answer": "Abul Fazl"
    },
    {
        "question": "What is the standard unit of speed?",
        "options": ["km/h", "m/s", "m/min", "km/s"],
        "answer": "m/s"
    }
]

async def handle_client(websocket):
    room_id = None
    role = None
    try:
        async for message in websocket:
            data = json.loads(message)
            action = data.get("action")

            if action == "create_room":
                room_id = data["room_id"]
                ROOMS[room_id] = {"host": websocket, "guest": None, "current_q": 0}
                role = "host"
                await websocket.send(json.dumps({"status": "room_created", "role": "host"}))
                print(f"Room {room_id} created by Host.")

            elif action == "join_room":
                room_id = data["room_id"]
                if room_id in ROOMS and ROOMS[room_id]["guest"] is None:
                    ROOMS[room_id]["guest"] = websocket
                    role = "guest"
                    await websocket.send(json.dumps({"status": "joined", "role": "guest"}))
                    # Notify Host that guest has connected
                    await ROOMS[room_id]["host"].send(json.dumps({"status": "guest_connected"}))
                    print(f"Guest joined Room {room_id}.")
                else:
                    await websocket.send(json.dumps({"status": "error", "message": "Room full or missing."}))

            elif action == "next_question" and role == "host":
                q_idx = ROOMS[room_id]["current_q"]
                if q_idx < len(CLASS_7_QUESTIONS):
                    q_data = CLASS_7_QUESTIONS[q_idx]
                    
                    # Host payload includes the correct answer
                    host_payload = {"status": "question", "data": q_data, "is_host": True}
                    # Guest payload excludes the correct answer
                    guest_payload = {
                        "status": "question", 
                        "data": {
                            "question": q_data["question"],
                            "options": q_data["options"]
                        }, 
                        "is_host": False
                    }
                    
                    await ROOMS[room_id]["host"].send(json.dumps(host_payload))
                    if ROOMS[room_id]["guest"]:
                        await ROOMS[room_id]["guest"].send(json.dumps(guest_payload))
                    ROOMS[room_id]["current_q"] += 1
                else:
                    end_msg = json.dumps({"status": "game_over"})
                    await ROOMS[room_id]["host"].send(end_msg)
                    if ROOMS[room_id]["guest"]:
                        await ROOMS[room_id]["guest"].send(end_msg)

    except websockets.exceptions.ConnectionClosed:
        print("A player disconnected.")
    finally:
        if room_id in ROOMS:
            if role == "host":
                del ROOMS[room_id]
            elif role == "guest" and room_id in ROOMS:
                ROOMS[room_id]["guest"] = None

async def main():
    async with websockets.serve(handle_client, "localhost", 8765):
        print("KBC Multiplayer Server started on ws = new WebSocket("wss://kbc-backend-username.snapdeploy.run");

        await asyncio.Future()  # Keep running

if __name__ == "__main__":
    asyncio.run(main())
