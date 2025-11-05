---
layout: post
title: "How I Finally Understood TCP and UDP"
date: 2025-11-06
tags: [networking, tcp, udp, kernel, learning, reflection]
category: "story"
summary: "A reflective journey into how data actually travels from an app through the kernel to the network — and what makes TCP and UDP fundamentally different."
---

### How I Finally Understood TCP and UDP

For the longest time, I couldn’t _imagine_ what TCP and UDP actually were.
I knew they had something to do with reliability and “rules of transmission,” but that knowledge felt abstract — like memorizing road signs without knowing what a road really looks like.

So I decided to start from the ground up.

---

#### The first big shift: apps don’t really “send” data

When I thought about two computers talking — say, through WhatsApp or Google Drive — I pictured the app doing everything.
The app “sending” and the app “receiving.”

Then I realized: the app doesn’t know how to push bytes through cables or Wi-Fi signals.
It just says, _“Hey system, send this data to that IP and port.”_

That’s when the lightbulb flicked on — it’s the **kernel**, not the app, that actually handles the delivery.
The app is just a customer; the kernel is the postal service inside my computer.

---

#### The kernel and its rulebooks

Once I pictured the kernel as that internal postal system, everything started to fall into place.
It has multiple _rulebooks_ for how to handle delivery — and TCP and UDP are simply two of them.

- **TCP** is the careful, reliable courier — tracking every package, confirming delivery, keeping them in order.
- **UDP** is the postcard approach — just drop it in the mailbox and move on.

Both are “rules” the kernel follows when the app opens a socket and says,

> “Use TCP (stream mode)” or “Use UDP (datagram mode).”

That small function call tells the OS which playbook to use for every packet.

```python
# Python examples
# TCP socket (stream-oriented)
import socket
tcp_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
tcp_sock.connect(("127.0.0.1", 80))

# UDP socket (datagram-oriented)
udp_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
udp_sock.sendto(b"Hello", ("127.0.0.1", 8080))
```

---

#### What’s really inside those packets

Seeing the headers made the picture vivid.
Every layer — Ethernet, IP, TCP/UDP — adds its own little envelope on top of my data, like nesting dolls.

A TCP packet felt like a registered parcel:
it had sequence numbers, acknowledgments, flags like SYN or FIN, and even a window size for how much data the receiver could handle.

UDP, on the other hand, was refreshingly minimal — just source, destination, and a quick checksum.
Eight bytes of “good luck” simplicity.

---

#### Why UDP still survives

I used to think UDP was outdated — why would anyone prefer something unreliable?
Then it clicked: in a world where the internet itself has become _so_ reliable, UDP’s speed becomes a feature, not a flaw.

Games, voice calls, live streams — they care about _freshness_ more than perfection.
If a single packet is late, it’s better to skip than to freeze the whole stream.
That’s when I realized reliability and speed aren’t opposites — they’re trade-offs.

---

#### Following a single “Hello”

Tracing a single “Hello” from my laptop to a server made it all real.

```python
# Simple Python TCP example

# Server
import socket
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(("0.0.0.0", 5000))
server.listen(1)
conn, addr = server.accept()
data = conn.recv(1024)
print("Received:", data.decode())
conn.sendall(b"Hi there!")
conn.close()

# Client
import socket
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(("<server-ip>", 5000))
client.sendall(b"Hello")     # internally calls send()
reply = client.recv(1024)
print("Server says:", reply.decode())
client.close()
```

- My app called `send()`.
- The kernel built a TCP header, wrapped it in IP, then Ethernet, and handed it to the network card.
- Routers stripped and rewrapped headers as it hopped across the internet.
- The receiver’s kernel unwrapped everything, verified sequence numbers, and finally passed “Hello” to the server’s app.

The layers stopped feeling like theory — they became a living assembly line.

---

#### When packets arrive out of order

At one point, I asked myself — _what if packets don’t arrive in sequence?_
Turns out, TCP quietly handles that behind the scenes.
The receiver buffers out-of-order pieces, reassembles them, and only gives the app a clean, ordered stream.

It’s like a librarian collecting scattered pages of a chapter before handing the full story to the reader.

---

#### Flow control: when the app is slow

Then I wondered — what if the app can’t keep up?
The kernel doesn’t let buffers overflow or crash the system.
Instead, it simply tells the sender, _“Hold on, I’m full,”_ by advertising a **zero window size.**
When space clears up, it says, _“Okay, send again.”_

It’s quiet backpressure — the network equivalent of breathing in rhythm.

---

#### The handshake that starts it all

The TCP handshake felt elegant once I saw it as a conversation of trust:

1. **SYN** — “Hey, I want to talk. My number starts at 1000.”
2. **SYN+ACK** — “Got it! My number starts at 5000.”
3. **ACK** — “Cool, we’re synced.”

That’s it — three simple messages that establish reliability, sequence, and mutual awareness.
And those random starting numbers? They protect against ghost packets from old connections.
(The TTL field makes sure those old ghosts eventually die, too.)

```text
TCP 3-Way Handshake

Client (initiates connection)                Server
─────────────────────────────────────────────────────────────
[SYN, Seq = x]  --------------------------->  (LISTEN → SYN-RECEIVED)
"I want to start, my initial number is x"

                 <---------------------------  [SYN, ACK, Seq = y, Ack = x+1]
(receives SYN+ACK) → (SYN-SENT → ESTABLISHED)
"Got it! My number is y, and I acknowledge yours."

[ACK, Seq = x+1, Ack = y+1] --------------->  (ESTABLISHED)
"All set — let's talk."
```

##### Quick notes

- Both sides exchange their **Initial Sequence Numbers (ISNs)** during this process.
- The **ACK in step 3** confirms both are alive and synchronized — completing the setup.
- After this, data transfer can begin in either direction (full duplex).

---

#### Closing the call — the four-step goodbye

When it’s time to end the connection, both sides politely finish their sentences:

1. One side says **FIN** — “I’m done talking.”
2. The other replies **ACK** — “Got it.”
3. Then it sends its own **FIN** — “I’m done too.”
4. Finally, a last **ACK** — “All clear.”

Once both FINs are exchanged and the final ACK is sent, the side that sent that last ACK enters **TIME-WAIT** — a short grace period to ensure no old or delayed packets resurface from the closed connection.
Only after this timer expires is the connection truly gone.
It’s a surprisingly graceful goodbye — no abrupt hang-ups, just a moment of patience before both walk away.

```text
TCP 4-Step Connection Teardown

Client (initiates close)                      Server
─────────────────────────────────────────────────────────────
[FIN, ACK]  ----------------------------->    (recv FIN)
(FIN-WAIT-1)                                 → (CLOSE-WAIT)

                 <-----------------------------  [ACK]
(receives ACK) → (FIN-WAIT-2)                 (still sending data if any)

                 <-----------------------------  [FIN, ACK]
(receives FIN) → (TIME-WAIT)                  → (LAST-ACK)

[ACK]  ------------------------------------->  (recv final ACK)
(wait ~2xMSL to clear old packets)            → (CLOSED)

After TIME-WAIT expires → (CLOSED)
```

##### Quick notes

- The side that **initiates** the close (sends the first FIN) ends up in **TIME-WAIT**.
- **2×MSL (Maximum Segment Lifetime)** is usually ~1–2 minutes — enough for delayed packets to expire.
- This waiting ensures clean separation before reusing the same port/IP combo.

---

#### The final realization

At the end of this journey, what once felt abstract — TCP, UDP, ports, headers — started to feel alive.
I stopped seeing them as cryptic protocols and started seeing them as **carefully layered agreements** between software, kernel, and hardware.

The app doesn’t send data.
It just makes a wish — and the kernel, the network stack, and the Internet’s machinery carry it through.

---

### 🧭 What I learned

- TCP and UDP aren’t just protocols — they’re _personalities_ in how data is treated.
- The kernel is the real engine of communication; the app only gives instructions.
- Reliability and speed are trade-offs, not opposites.
- Every packet is wrapped in context — headers are agreements between layers.
- And beneath it all, networking is just trust — built one SYN and ACK at a time.
